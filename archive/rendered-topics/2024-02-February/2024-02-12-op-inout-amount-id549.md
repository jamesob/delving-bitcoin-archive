# Op_inout_amount

Chris_Stewart_5 | 2024-02-12 15:05:39 UTC | #1

Hi all,

I've attempted to implement `OP_INOUT_AMOUNT`. The purpose of `OP_INOUT_AMOUNT` is to push the input value and a set of output values onto the Script interpreter stack.

Here is my draft BIP:

https://github.com/Christewart/bips/blob/92c108136a0400b3a2fd66ea6c291ec317ee4a01/bip-op-inout-amount.mediawiki

Here is the implementation. 

https://github.com/Christewart/bitcoin/tree/op-inout-amount

One purpose of implementing this is to see how this works in conjunction with my [64bit op code PR](https://github.com/bitcoin/bitcoin/pull/29221). For more information on this PR see the [delving bitcoin discussion here](https://delvingbitcoin.org/t/64-bit-arithmetic-soft-fork/397/25).

### Design questions

One limitation of the current implementation is that it only can push the input and output amounts at the _current_ index being verified inside of the Script interpreter. While easiest to implement, I wonder if this should be extended to verifying more than just the input and output at the current input index being verified. It seems like something similar to how `SIGHASH` flags work would be interesting. Interested in hearing others thoughts.

### Implementation questions

Currently I extend `BaseTransactionSignatureChecker` to have 2 new methods

- `GetNIn()` - the input index we are currently verifying
- `GetTransactionData()` - gives us access to `PreComputedTransactionData` so we have access to the output that is funding us, and the set of outputs we are spending to.

I don't think this is necessarily the best place to put these methods, but it seemed like the most convenient place for putting these to hack something together. Would be interested in hearing others thoughts of how to structure the implementation.

My end goal is to use all this stuff in conjunction for [OP_TLUV](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2021-September/019419.html).

-------------------------

halseth | 2024-02-27 14:02:08 UTC | #2

Thanks for moving this along! 

I think it would be a missed opportunity to have this be available only for the current input/output pair. 

Having it be possible to specify indexes would be very helpful in creating more sophisticated merges/splits of UTXOS. And I think the implementation complexity would not drastically increase?

-------------------------

Chris_Stewart_5 | 2025-03-13 21:08:20 UTC | #3

Ok Johan, i finally took you up on this. Here is my attempt at implementing what I think you and [AJ Towns](https://gnusha.org/pi/bitcoindev/20210909065330.GB22496@erisian.com.au/) were hinting at.

>Having it be possible to specify indexes would be very helpful in creating more sophisticated merges/splits of UTXOS. And I think the implementation complexity would not drastically increase?

>... or it could even accept a parameter letting you specify which input ...

## Background

Elements has 2 opcodes called [OP_INSPECTINPUTVALUE](https://github.com/ElementsProject/elements/blob/df4e99dca3b9f4a49592b98fed3af8aeed6035ae/src/script/interpreter.cpp#L1875) and [OP_INSPECTOUTPUTVALUE](https://github.com/ElementsProject/elements/blob/df4e99dca3b9f4a49592b98fed3af8aeed6035ae/src/script/interpreter.cpp#L1958). These opcodes are limited to pushing the value of the current input index and its corresponding output index onto the stack. I believe this means that there must be a one-to-one size relationship for inputs and outputs in element's transactions if either `OP_INSPECTINPUTVALUE` or `OP_INSPECTOUTPUTVALUE` is used. Also - IIUC - this creates a simple heuristic to correlate inputs and outputs in a transaction. If this is incorrect, please let me know. 

## Design

A design goal of mine for this iteration of `OP_INOUT_AMOUNT` is to allow for variable size input and output sets in a bitcoin transaction.

I reworked `OP_INOUT_AMOUNT` to take 2 stack parameters. They are each interpreted as bitmaps. We sum the outputs and inputs at the set indices and push the result onto the stack.

Lets assume our transaction has the format

>CTx(input_amounts=[1BTC,2BTC],output_amounts=[0.25BTC,0.5BTC,1BTC])

and the current stack state before evaluating `OP_INOUT_AMOUNT` is

```
10100000 # output indices to push onto the stack
11000000 # input indices to push onto the stack
```

This would mean the values at output indexes 0 and 2 (`0.25BTC` and `1BTC`) should be added together and pushed onto the stack.

This also mean the values at input indices 0 and 1 (`1BTC` and `2BTC`) should be added together and pushed onto the stack.

The resulting stack would be

```
1.25BTC # output value pushed onto the stack
3BTC # input value pushed onto the stack
```

Here is the to see the [implementation](https://github.com/Christewart/bitcoin/blob/df5ef7baf2493c59062991d872425cfdf39d181f/src/script/interpreter.cpp#L1283) and [test cases](https://github.com/Christewart/bitcoin/blob/df5ef7baf2493c59062991d872425cfdf39d181f/test/functional/feature_inout_amount.py#L7).

## Coinjoin use case

Lets assume we want to specify that the first 5 outputs have uniform amounts of 1BTC in our transaction.

The funding output with the `OP_INOUT_AMOUNT` Script contains `2.1BTC`. This means the counterparty must bring `3BTC` or more to the transaction to satisfy the witness script. This implies the counterparty can claim a `0.1BTC` fee for their services.  Here is what the Script looks like

>OP_0 OP_1 OP_INOUT_AMOUNT ONE_BTC OP_EQUALVERIFY OP_DROP 
>OP_0 OP_2 OP_INOUT_AMOUNT ONE_BTC OP_EQUALVERIFY OP_DROP 
>OP_0 OP_4 OP_INOUT_AMOUNT ONE_BTC OP_EQUALVERIFY OP_DROP 
>OP_0 OP_8 OP_INOUT_AMOUNT ONE_BTC OP_EQUALVERIFY OP_DROP 
>OP_0 OP_16 OP_INOUT_AMOUNT ONE_BTC OP_EQUALVERIFY OP_DROP 
>OP_1

You could put these scripts in a tapscript tree to allow for more fees for larger output sets - you could imagine having an alternative tapleaf that requires `0.5BTC` outputs and paying a larger fee ( `0.6BTC`in this toy example) compensate for the larger network fee for more outputs and a larger witness script.

## Design questions / problems that I see

These are design issues I've seen, I'm posting them so I've got them written down and out of my head. There are probably more issues that I haven't understood or realized yet, so please comment below if you see them. Perhaps someone cleverer than me can figure out a way to fix them, explain why they aren't a problem, or suggest why other blockchains such as Elements chose different designs to avoid these problems?

### Bitmaps are too flexible?

Lets assume my use case is to make an `OP_INOUT_AMOUNT` Script that guarantees a max fee in a transaction. This means my Script needs to know about _all_ input and output values in the transaction. If my Script is unaware of all values, it cannot accurately calculate the absolute fee. This is a similar to the attack that possible before [BIP341](https://github.com/bitcoin/bips/blob/050d422b2ac24d8221edab0ff0053e0f585409f7/bip-0341.mediawiki#cite_note-18) adjusted its sighash algorithm to commit to all input values.

One way to solve this is to require that the bitmap specifies sets _all_ transaction inputs and outputs.

However I believe this would exclude other use cases where you would only like to check a subset of inputs/outputs, like vaults or coinjoins.

In a coinjoin transaction, you may want to exclude 1 output from being required to have a uniform value to send "change" too. This is convenient as its unlikely that  `(sum(input_amounts) - network_fees) % COINJOIN_AMOUNT` is going to be a round number.

To put it simply, is there a way we have both of these features depending on the context

1. A guarantee we know about all inputs/outputs
2. The flexibility to specify subsets of inputs and outputs

I guess this can be solved with another opcode like `OP_PUSHINPUTSET_BITMAP` / `OP_PUSHOUTPUTSET_BITMAP`, but that introduces more complexity...

### Ordering of stack inputs

`OP_INOUT_AMOUNT` as I've designed it consumes 2 inputs. The stack top is a bitmap of output indices, the second item on the stack is a bitmap of input indices.

If for some reason you would like to express a input value pattern, but allow the caller of the Script to express an output value pattern, this would not be possible. I can't think of a use case for this, but it does seem like a design issue. This probably means this opcode should be broken down into 2 opcodes, as done in elements?

### Malleability

Lets assume my witness script looks like this

>OP_1 OP_INOUT_AMOUNT ONE_BTC OP_DUP OP_EQUALVERIFY OP_EQUAL

This script checks that the output at index 0 is equal to `1BTC`. It also checks there exists at least 1 input that is equal to `1BTC`. The caller of this Script can specify the input index to check that its equal to exactly `1BTC`.

If you have multiple inputs that have exactly `1BTC`, 3rd parties on the network could malleate the witness stack and still have the bitcoin transaction be valid.

-------------------------

Chris_Stewart_5 | 2025-05-02 17:37:51 UTC | #4

# Case study: OP_VAULT


This case study explores how Script opcodes can be used to implement **amount locks**—restrictions that ensure the value of inputs and outputs in a transaction meets certain conditions. The goal is to evaluate the required features and developer ergonomics for opcodes that push input and output amounts onto the stack. Rather than starting from scratch, we build on existing opcode proposals and retrofit them to support amount locks directly in Script

This requires 2 proposals I am working on

1. [64-bit arithmetic in Script](https://github.com/Christewart/bips/blob/79257ba5d7a632fa828208f266fd4f5540ffba7f/bip-XXXX.mediawiki)
2. [`OP_INOUT_AMOUNT`](https://delvingbitcoin.org/t/op-inout-amount/549/3?u=chris_stewart_5)

**Note:** This study does not attempt to implement _destination locks_—restrictions on where funds may be sent. That logic is preserved from the original proposal being examined.

[Here](https://github.com/Christewart/bitcoin/tree/2025-04-16-covtools-softfork-nochange) is a link to the repository that implements everything talked about below - a good place to start reading is the functional test [`feature_vaults.py`](https://github.com/Christewart/bitcoin/blob/4c6c2ecb59a132eeac43d5608a1a1c081940b0e0/test/functional/feature_vaults.py)

## BIP345
[BIP345](https://github.com/bitcoin/bips/blob/3365fb7a7e5e25b95b94d65808e32a02aa684aaa/bip-0345.mediawiki) proposes a mechanism that enforces a delay before certain coins can be spent to arbitrary destinations—unless they're redirected along a predefined "recovery" path. At any time before final withdrawal, the funds can be moved to this recovery path.

The proposal introduces two opcodes—`OP_VAULT` and `OP_VAULT_RECOVER`—that depend on enforcing **two amount locks**. In the original implementation, these were enforced using [deferred checks](https://github.com/bitcoin/bips/blob/3365fb7a7e5e25b95b94d65808e32a02aa684aaa/bip-0345.mediawiki#cite_note-7):

1. `trigger_vout_value` + `revault_vout_value?` = sum(`funding_vault_outputs`)  
2. sum(`vault_recover_outputs`) = `recov_vout_value`

> Note: In (1), `revault_vout_value` is optional and may not be present in every case.

This document explores how these amount locks can be enforced directly in Script using `OP_INOUT_AMOUNT`, eliminating the need for deferred checks for the amount lock portion of `OP_VAULT`/`OP_VAULT_RECOVER`.


## OP_VAULT

Here is what the BIP345 stack looks like when evaluating `OP_VAULT`
```
<leaf-update-script-body>
<push-count>
[ <push-count> leaf-update script data items ... ]
<trigger-vout-idx>
<revault-vout-idx>
<revault-amount>
```

We are interested in `trigger-vout-idx`, `revault-vout-idx`, and `revault-amount`. The `OP_VAULT` opcode checks that the destination lock is satisfied, and the amount lock is satisfied via a combination of [logic in the opcode implementation itself](https://github.com/jamesob/bitcoin/blob/5e59c074703f0913db7ee004b99d086857d10ad6/src/script/interpreter.cpp#L2159), and a [deferred check](https://github.com/jamesob/bitcoin/blob/5e59c074703f0913db7ee004b99d086857d10ad6/src/validation.cpp#L1894) that is run after all inputs are validated in the trigger transaction.

### Trigger Transaction's Amount Lock Logic in Script

Here is what the witness stack looks like when you begin to evaluate in `OP_VAULT` output in a trigger transaction.

```
<leaf-update-script-body>
<push-count>
[ <push-count> leaf-update script data items ... ]
<trigger-vout-idx>
<revault-vout-idx>
<input_indices> # a bitmap for compatible OP_VAULT outputs we are verifying in this transaction
```

The only new field is `input_indices` which corresponds to the compatible `OP_VAULT` outputs that are sending funds to the `trigger-vout-idx` and `revault-vout-idx`. This change removes `<revault-amount>` from the stack as it will be pushed onto the stack by `OP_INOUT_AMOUNT`. As a side note, I don't believe `revault-amount` is required in the original BIP345 proposal as the value can be checked via deferred checks.

Here is what the corresponding `trigger_script` looks like to evaluate this stack

```
OP_6,            # Depth of input bitmap on stack
OP_ROLL,         # Move input bitmap to stack top
OP_6,            # Depth of revault vout index on stack
OP_ROLL,         # Move revault vout index to top
OP_6,            # Depth of trigger vout index on stack
OP_PICK,         # Copy trigger vout index to top (keep original for OP_VAULT)
OP_SWAP,         # Bring revault index to top
OP_DUP,          # Copy revault index (for -1 check)
OP_1NEGATE,
OP_EQUAL,        # Check if revault index == -1

OP_IF,           # Case: No revault output present
  OP_DROP,       # Drop duplicated -1

  # Convert trigger index into bitmap (simulate shift table since we have no OP_LSHIFT)
  OP_DUP,
  OP_0,
  OP_EQUAL,
  OP_IF,
    OP_DROP,
    OP_1,
  OP_ELSE,
    OP_DUP,
    OP_1,
    OP_EQUAL,
    OP_IF,
      OP_DROP,
      OP_2,
    OP_ELSE,
      OP_0,
      OP_VERIFY,
    OP_ENDIF,
  OP_ENDIF,

  # Push amounts: op_vault_input_sum, trigger_vout_value
  OP_INOUT_AMOUNT,
  OP_EQUALVERIFY,  # Require: sum(inputs) == trigger output
OP_ELSE,         # Case: Revault output exists
  OP_DUP,
  OP_0,
  OP_GREATERTHAN,
  OP_VERIFY,      # Require revault index >= 0

  # Convert revault index into bitmap (since we have no OP_LSHIFT)
  OP_DUP,
  OP_0,
  OP_EQUAL,
  OP_IF,
    OP_DROP,
    OP_1,
  OP_ELSE,
    OP_DUP,
    OP_1,
    OP_EQUAL,
    OP_IF,
      OP_DROP,
      OP_2,
    OP_ELSE,
      OP_0,
      OP_VERIFY,
    OP_ENDIF,
  OP_ENDIF,

  OP_2,           # Depth of input bitmap
  OP_ROLL,        # Bring input bitmap to top
  OP_SWAP,        # Reorder: input bitmap, output bitmap
  OP_INOUT_AMOUNT, # Push amounts: op_vault_input_sum, revault_output_value

  # Prepare trigger output lookup
  OP_2,
  OP_ROLL,
  OP_0,           # Dummy input bitmap
  OP_SWAP,

  # Convert trigger index into bitmap (since we have no OP_LSHIFT)
  OP_DUP,
  OP_0,
  OP_EQUAL,
  OP_IF,
    OP_DROP,
    OP_1,
  OP_ELSE,
    OP_DUP,
    OP_1,
    OP_EQUAL,
    OP_IF,
      OP_DROP,
      OP_2,
    OP_ELSE,
      OP_0,
      OP_VERIFY,
    OP_ENDIF,
  OP_ENDIF,

  OP_INOUT_AMOUNT, # Push trigger output amount
  OP_SWAP,
  OP_DROP,         # Drop dummy input amount
  OP_ADD,          # total_outputs = trigger + revault
  OP_EQUALVERIFY,  # Require: sum(inputs) == total_outputs
OP_ENDIF,

OP_VAULT          # Final vault check
```

This Script checks this invariant using only Script

>1. `trigger_vout_value` + `revault_vout_value?` = sum(`funding_vault_outputs`)

[Small modifications were made to the OP_VAULT implementation](https://github.com/Christewart/bitcoin/blob/4c6c2ecb59a132eeac43d5608a1a1c081940b0e0/src/script/interpreter.cpp#L1427). Namely the `OP_VAULT` opcode no longer consumes these stack arguments because the logic is implemented in Script

1. `revault_vout_idx`
2. `revault_amount`

`revault_amount` is removed all together, and `revault_vout_idx` is used by `OP_INOUT_AMOUNT` rather than `OP_VAULT`. 

## OP_VAULT_RECOVER

`OP_VAULT_RECOVER` is an opcode that allows a user to recover a vaulted amount from a trigger transaction. This output can be spent at any time before the withdrawal transaction becomes confirmed on the bitcoin network.

Here is what the witness stack looks like for `OP_VAULT_RECOVER` in BIP345.

```
<recovery-sPK-hash>
<recovery-vout-idx>
```

We are interested in `recovery_vout_idx`. This index represents the output that the recovered funds are sent to in the recovery transaction.

We need all inputs that spend an `OP_VAULT_RECOVER` to sum to the output value at the `recover_vout_idx`. The original implementation in BIP345 uses deferred checks to enforce this invariant - we are going to show how `OP_INOUT_AMOUNT` could be used to replace the deferred check.

### Recovery Transaction's Amount Lock Logic in Script

This is what the `recovery_script` looks like to implement the amount lock on the recovery transaction. 

```
OP_DUP,           # Duplicate recovery_output_idx for lookup

# Simulate a left shift for recovery_output_idx using a lookup table since OP_LSHIFT isn't available
OP_0,
OP_EQUAL,
OP_IF,
  OP_1,
OP_ELSE,
  OP_DUP,
  OP_1,
  OP_EQUAL,
  OP_IF,
    OP_2,
  OP_ELSE,
    OP_DUP,
    OP_2,
    OP_EQUAL,
    OP_IF,
      OP_4,
    OP_ELSE,
      OP_DUP,
      OP_3,
      OP_EQUAL,
      OP_IF,
        OP_8,
      OP_ELSE,
        OP_DUP,
        OP_5,
        OP_EQUAL,
        OP_IF,
          CScriptNum(32),
        OP_ELSE,
          OP_DUP,
          OP_1NEGATE,
          OP_EQUAL,
          OP_IF,
            OP_1NEGATE,
          OP_ELSE,
            OP_0,
            OP_VERIFY,
          OP_ENDIF,
        OP_ENDIF,
      OP_ENDIF,
    OP_ENDIF,
  OP_ENDIF,
OP_ENDIF,

OP_2,             # Stack depth of trigger_input_indices
OP_ROLL,          # Move input bitmap to stack top
OP_SWAP,          # Reorder: input_bitmap, output_bitmap
OP_INOUT_AMOUNT,  # Push input_sum and recovery_output_value
OP_EQUALVERIFY,   # Ensure: sum(trigger_inputs) == recovery_output_value

self.recovery_hash,
OP_VAULT_RECOVER
```

If you are remove the shift table, the Script is relatively compact to enforce the recovery output's amount lock.


## Learnings

1. Any index based opcodes require shift operators to be have nice developer ergonomics in Script. Most of the Script I've written for BIP345's logic comes down to writing a table to figure out what the appropriate input/output indice we are checking.
2. Segregating `OP_INOUT_AMOUNT` into 2 opcodes - `OP_IN_AMOUNT` and `OP_OUT_AMOUNT` would reduce the amount of stack manipulation that needs to be done with `OP_PICK`/`OP_ROLL`.

-------------------------

Chris_Stewart_5 | 2025-05-07 17:16:31 UTC | #5

# Case study: OP_CHECKCONTRACTVERIFY

This case study explores how Script opcodes can be used to implement **amount locks**—restrictions that ensure the value of inputs and outputs in a transaction meets certain conditions. The goal is to evaluate the required features and developer ergonomics for opcodes that push input and output amounts onto the stack. Rather than starting from scratch, we build on existing opcode proposals and retrofit them to support amount locks directly in Script.

This requires two proposals I am working on:

1. [64-bit arithmetic in Script](https://github.com/Christewart/bips/blob/79257ba5d7a632fa828208f266fd4f5540ffba7f/bip-XXXX.mediawiki)  
2. [`OP_IN_AMOUNT` & `OP_OUT_AMOUNT`](https://delvingbitcoin.org/t/op-inout-amount/549/3?u=chris_stewart_5)

**Note:** This study does not attempt to implement _destination locks_—restrictions on where funds may be sent. That logic is preserved from the original proposal being examined.

[Here](https://github.com/Christewart/bitcoin/tree/ccv-core-op-in-out-amount) is a link to the repository that implements everything discussed below. A good place to start reading is the functional test: [`feature_checkcontractverify.py`](https://github.com/Christewart/bitcoin/blob/c83ed810a889e4e69ba8c417ddf4c85c1723f9e5/test/functional/feature_checkcontractverify.py).


## OP_CHECKCONTRACTVERIFY

[OP_CHECKCONTRACTVERIFY](https://github.com/Merkleize/bips/blob/2655b8b88f15ac3ecfaad91f2c2c1a7be3454fc3/bip-ccv.mediawiki) is an opcode enables users to create UTXOs that carry a dynamic commitment to a piece of data. The commitment can be validated during the execution of the script, allowing introspection to the committed data. Moreover, a script can constrain the internal public key and taptree of one or more outputs, and possibly the committed data.

This case study is interested in the [amount locks](https://github.com/Merkleize/bips/blob/2655b8b88f15ac3ecfaad91f2c2c1a7be3454fc3/bip-ccv.mediawiki#user-content-Output_amounts) of the `OP_CHECKCONTRACTVERIFY` proposal. Similar to the [BIP345](https://github.com/bitcoin/bips/blob/3365fb7a7e5e25b95b94d65808e32a02aa684aaa/bip-0345.mediawiki) [case study](https://delvingbitcoin.org/t/op-inout-amount/549/4?u=chris_stewart_5), we are going to replace the current amount locks implemented via [modes](https://github.com/Merkleize/bips/blob/2655b8b88f15ac3ecfaad91f2c2c1a7be3454fc3/bip-ccv.mediawiki#output-amounts) in the [`CheckContract()`](https://github.com/bitcoin/bitcoin/blob/1fca2689866761ee4cd6629c0f3a0deb0fac72ae/src/script/interpreter.cpp#L1908) function in the OP_CCV proposal with the opcodes `OP_IN_AMOUNT` and `OP_OUT_AMOUNT`.


## OP_CCV Witness Stack

Here is what the stack looks like when evaluating `OP_CCV`
```
`<mode>` # is a minimally encoded integer, according to one of the values defined below.
`<taptree>` # is the Merkle root of the taproot tree, or a minimally encoded `-1`, or the empty buffer.
`<pk>` # is called the _naked key_, and it's a valid 32-byte x-only public key, or a minimally encoded `-1`, or the empty buffer.
`<index>` # is a minimally encoded -1, or a minimally encoded non-negative integer.
`<data>` # is a buffer of arbitrary length.
```

We are interested in two values on the stack for implementing amount locks - `index` and `mode`. 

### Modes

There are 4 modes defined in the `OP_CHECKCONTRACTVERIFY` BIP. Unfortuantely the OP_CCV opcode currently has context specific meanings for the `index` value on the stack depending on the `mode` below. This will not work with implement amount locks in Script, so this work changes the meaning of `index` to be the `output_index` we are spending to. 

We add a new value called [`input_indices`](https://github.com/Christewart/bitcoin/blob/c83ed810a889e4e69ba8c417ddf4c85c1723f9e5/test/functional/feature_checkcontractverify.py#L232) that reference all of the funding outputs we are validating.

#### CCV_MODE_CHECK_INPUT

The `CCV_MODE_CHECK_INPUT` checks an input's script; no amount check. This means we do not need to implement an amount lock in Script.

#### CCV_MODE_CHECK_OUTPUT_IGNORE_AMOUNT

The `CCV_MODE_CHECK_OUTPUT_IGNORE_AMOUNT` checks an output's script, but ignores the amount. This means we do not need to implement an amount lock in the Script.

#### CCV_MODE_CHECK_OUTPUT

The `CCV_MODE_CHECK_OUTPUT` checks an output's script; preserve the (possibly residual) amount. 

Here is what the amount lock for this mode looks like when implemented in Script with `OP_IN_AMOUNT` and `OP_OUT_AMOUNT`. You can view the python implementation with working test cases [here](https://github.com/Christewart/bitcoin/blob/c83ed810a889e4e69ba8c417ddf4c85c1723f9e5/test/functional/feature_checkcontractverify.py#L271).

```
OP_3, # output_indices position on stack
OP_PICK, # move output_indices to stack top
OP_DUP, #duplicate output_indices

# shift table for index since we have no OP_LSHIFT
OP_0,
OP_EQUAL,
OP_IF,
  OP_DROP, # drop duplicated index
  OP_1,
OP_ELSE,
  OP_0,
  OP_VERIFY,
OP_ENDIF,
# end shift table

OP_OUT_AMOUNT, # push output_amount onto stack
OP_5, # input indices position on stack
OP_ROLL, # move input_indices to stack top
OP_IN_AMOUNT, # push input amount onto stack
OP_EQUALVERIFY, # make sure input and output amounts are equal
OP_CHECKCONTRACTVERIFY,
OP_TRUE
```

#### CCV_MODE_CHECK_OUTPUT_DEDUCT_AMOUNT

The `CCV_MODE_CHECK_OUTPUT_DEDUCT_AMOUNT` mode checks an output's script and deducts the output amount from the input's residual amount.

Here is what the amount lock for this mode looks like when implemented in Script with `OP_IN_AMOUNT` and `OP_OUT_AMOUNT`. You can view the python implementation with working test cases [here](https://github.com/Christewart/bitcoin/blob/c83ed810a889e4e69ba8c417ddf4c85c1723f9e5/test/functional/feature_checkcontractverify.py#L351).

```

# 1. Push full amount from input_indices
# 2. Push first output index value
# 3. Subract first output index value from input_indices amount
# 4. Push second output index value
# 5. Check (input_value - first_output_value) - second_output_value = 0

OP_3, # input_indices on stack
OP_ROLL, # move input_indices to stack top
OP_IN_AMOUNT, # push full input amount onto the stack
OP_4, # first output index on stack
OP_PICK, # move it to stack top

# shift table for index since we have no OP_LSHIFT
OP_0,
OP_EQUAL,
OP_IF,
  OP_1,
OP_ELSE,
  OP_0,
  OP_VERIFY,
OP_ENDIF,
# end shift table

OP_OUT_AMOUNT, # push first_output_amount onto stack
OP_SUB, # input_amount - first_output_amount
OP_TOALTSTACK, # move input_amount - first_output_amount to alt stack for now to check later

CCV_MODE_CHECK_OUTPUT_DEDUCT_AMOUNT,
OP_CHECKCONTRACTVERIFY,

0, # no data tweaking (hardcoded in author's OP_CCV test case)
1, # index (hardcoded in author's OP_CCV test case)
0, # NUMS pubkey (hardcoded in author's OP_CCV test case)
0, # no taptweak (hardcoded in author's OP_CCV test case)

# check output, all remaining amount must go to this output
OP_2,
OP_PICK,

# shift table for index since we have no OP_LSHIFT
OP_1,
OP_EQUAL,
OP_IF,
  OP_2,
OP_ELSE,
  OP_0,
  OP_VERIFY,
OP_ENDIF,
# end shift table

OP_OUT_AMOUNT, # push second_output_value onto stack
OP_FROMALTSTACK, # move input_amount - first_output_amount back from alt stack
OP_SUB, # (input_amount - first_output_amount) - second_output_value
OP_0,
OP_EQUALVERIFY, # (input_amount - first_output_amount) - second_output_value = 0
CCV_MODE_CHECK_OUTPUT,
OP_CHECKCONTRACTVERIFY,
OP_TRUE
```

### One Big Beautiful Script

The `mode` for OP_CCV could be given as input to this Script and the mode could be matched with `OP_IF` `OP_ELSE` `OP_ENDIF`. The body of each conditional would be the different Script's written above. This could give the spending transaction more control over how the Script is evaluated. 

I'm not sure if it could make sense to add `mode` to the witness stack in certain cases - or have it computed during Script execution. If there is a use case where this makes sense, having "one big beautiful Script" where all of the amount lock's conditionals are included in the Script could be very useful.

## Lessons learned

### Extensibility 

In the case of OP_CCV, amount locks implemented with `OP_IN_AMOUNT` and `OP_OUT_AMOUNT` greatly enhance the extensibility of the proposal. Rather than having to soft fork in a new `mode` every time there is a new feature we want to introduce, Script programmers can just implement the logic themselves. This makes the `OP_CCV` proposal much more flexible.

### Separation of concerns

I think it is very useful to think of "destination locks" and "amount locks" separately. You need both restrictions to implement any covenant proposal. Trying to build them both in the same proposal can lead to sub-optimal design choices.

For Instance, in this proposal it seems like OP_CCV is very close to [`OP_TAPLEAFUPDATEVERIFY`](https://gnusha.org/pi/bitcoindev/20210909064138.GA22496@erisian.com.au/) if it were to add support for adding/remove tap leafs from the taptree. 

The merkle root hash for the taptree is already on the stack for OP_CCV evaluation as its needed for verifying the piece of data is included in the internal key. It seems like it isn't too far of a stretch to adding a parameter to add a new leaf to the tap tree or removing a leaf.

### Separating OP_INOUT_AMOUNT into two opcodes: OP_IN_AMOUNT, OP_OUT_AMOUNT

This did reduce some stack manipulation. Namely, I didn't have to use `OP_SWAP` as much to make sure stack arguments were in the correct place to be consumed by the single opcode `OP_INOUT_AMOUNT`. 

Unfortunately `OP_PICK` and `OP_ROLL` usage is still necessary to make sure stack arguments could be moved to the correct places to be used by the Script implementing the amount locks.

### `OP_LSHIFT`/`OP_RSHIFT`

As expressed on the BIP345 case study, this case study reaffirms the need for `OP_LSHIFT`/`OP_RSHIFT` for a proposal like `OP_IN_AMOUNT`/`OP_OUT_AMOUNT` to be viable in its current form - specifically consuming bitvectors that correspond to input/output indices as an argument.

-------------------------

salvatoshi | 2025-05-08 14:09:29 UTC | #6

Hi Chris,

Thanks for exploring the combination of amount opcodes with covenant opcodes - I think it's interesting and there is some potential synergy.

However, I think removing the amount semantic from `CCV` (or `VAULT`) is problematic.

1. It only works for cases where the structure of the transaction (that is, "what inputs will be spent together") is known in advance and can be hardcoded in Script. This might be true for some use cases, but is certainly false for ergonomic vaults. You might receive a number of transactions to a vault address, and then at spending time (*trigger* transaction) you'll want to select which of the vault UTXOs you want use as part of the withdrawal. The number and the position of those UTXOs can't be known in advance when their script is defined. In order to avoid having to hardcode these bitmaps, you'd need some cross-input logic to somehow make sure that all those inputs are using compatible bitmaps. I believe this is in fact the most interesting part of the amount logic of `CCV` (inspired from `VAULT` rather than from the original applications of `CCV` to MATT and fraud proofs). To the best of my understanding, this is not implemented in your demo.
2. Assuming that a clean solution to (1) is found, since all the related inputs that are being aggregated need to have a matching bitmap, the trivial implementation would require to report this same bitmap for all the inputs. This has $O(n^2)$ cost both in terms of space occupation and computational cost. While $O(n^2)$ bits and $O(n^2)$ additions might not be a huge deal for many common use cases, it seems rather unsatisfactory to do in $O(n^2)$ cost something that can be done optimally in $O(n)$ cost. Without a real, embedded cross-input logic, the only way to achieve the optimal $O(n)$ cost seems to be something like [this demo from burak](https://brqgoo.medium.com/emulating-op-vault-with-elements-opcodes-bdc7d8b0fe71) <small>(TL;DR: one special input performs all the amount checks, while the other inputs merely check the presence of the special input)</small>, which is not exactly ergonomic.

More generally, any covenant opcode that constrains the destination seems to be pointless if the covenant opcode itself doesn't _also_ enforce the *presence* of the amount logic, whether embedded in the same opcode or enforced via some other mechanism. This is something I also [commented about darosior's approach using the annex](https://delvingbitcoin.org/t/op-checkcontractverify-and-its-amount-semantic/1527/7?u=salvatoshi), and I strongly believe that some amount of redundancy is unavoidable for any mechanism that extracts the amount logic out of the covenant opcode.

Because of these reasons, I'm not convinced that ejecting the amount logic leads to improved outcomes or better scripts.

Note that I believe that opcodes like `OP_OUT_AMOUNT` would be very useful in combination with `OP_CCV`, particularly with the `DEDUCT` mode: by simply having equality checks for outputs, one could avoid explicit arithmetic over 64-bit amounts in a Script that performs a withdrawal of one or several users from a shared UTXO.

`OP_IN_AMOUNT` and 64-bit arithmetic might of course also be useful for some applications - for example to implement constructions with velocity limits.

-------------------------

Chris_Stewart_5 | 2025-05-08 18:35:01 UTC | #7

[quote="salvatoshi, post:6, topic:549"]
It only works for cases where the structure of the transaction (that is, “what inputs will be spent together”) is known in advance and can be hardcoded in Script.
[/quote]

Why? By "it" I assume you are referring to `OP_CCV`/`OP_VAULT`. Is there some security consideration I am missing?

[quote="salvatoshi, post:6, topic:549"]
You might receive a number of transactions to a vault address, and then at spending time (*trigger* transaction) you’ll want to select which of the vault UTXOs you want use as part of the withdrawal.
[/quote]

This is tested in the [BIP345](https://delvingbitcoin.org/t/op-inout-amount/549/4?u=chris_stewart_5) case study. I didn't have this issue you are talking about. Both [`test_batch_unvault`](https://github.com/Christewart/bitcoin/blob/4c6c2ecb59a132eeac43d5608a1a1c081940b0e0/test/functional/feature_vaults.py#L362)  and [`test_batch_recovery`](https://github.com/Christewart/bitcoin/blob/4c6c2ecb59a132eeac43d5608a1a1c081940b0e0/test/functional/feature_vaults.py#L251) test cases work with `OP_{IN,OUT}_AMOUNT`.

[quote="salvatoshi, post:6, topic:549"]
The number and the position of those UTXOs can’t be known in advance when their script is defined. In order to avoid having to hardcode these bitmaps, you’d need some cross-input logic to somehow make sure that all those inputs are using compatible bitmaps
[/quote]

Yes, that is why they should be provided on the witness stack at spending time rather than when the utxos are created. I modified your [OP_CCV test code](https://github.com/Christewart/bitcoin/blob/c83ed810a889e4e69ba8c417ddf4c85c1723f9e5/test/functional/feature_checkcontractverify.py#L232) code to do just that.

[quote="salvatoshi, post:6, topic:549"]
I believe this is in fact the most interesting part of the amount logic of `CCV` (inspired from `VAULT` rather than from the original applications of `CCV` to MATT and fraud proofs).
[/quote]

Admittedly I am confused, as far as I can tell your test framework defines the index parameter for OP_CCV in the output script rather than placing it in the witness when the OP_CCV utxo is spent. Here is a link to the code I am looking at: [1](https://github.com/Merkleize/bitcoin/blob/1fca2689866761ee4cd6629c0f3a0deb0fac72ae/test/functional/feature_checkcontractverify.py#L280) [2](https://github.com/Merkleize/bitcoin/blob/1fca2689866761ee4cd6629c0f3a0deb0fac72ae/test/functional/feature_checkcontractverify.py#L303) [3](https://github.com/Merkleize/bitcoin/blob/1fca2689866761ee4cd6629c0f3a0deb0fac72ae/test/functional/feature_checkcontractverify.py#L311) [4](https://github.com/Merkleize/bitcoin/blob/1fca2689866761ee4cd6629c0f3a0deb0fac72ae/test/functional/feature_checkcontractverify.py#L258) [5](https://github.com/Merkleize/bitcoin/blob/1fca2689866761ee4cd6629c0f3a0deb0fac72ae/test/functional/feature_checkcontractverify.py#L234). 

I understand that both of our proposals are works in progress -- is the the goal to remove the hard coded indices in favor of providing them on the witness stack in your OP_CCV test cases?

[quote="salvatoshi, post:6, topic:549"]
Assuming that a clean solution to (1) is found, since all the related inputs that are being aggregated need to have a matching bitmap, the trivial implementation would require to report this same bitmap for all the inputs. This has $O(n^2)$ cost both in terms of space occupation and computational cost. While $O(n^2)$ bits and $O(n^2)$ additions might not be a huge deal for many common use cases, it seems rather unsatisfactory to do in $O(n^2)$ cost something that can be done optimally in $O(n)$ cost.
[/quote]

I think this is a fair critique and something I'm going to look into.

[quote="salvatoshi, post:6, topic:549"]
Without a real, embedded cross-input logic, the only way to achieve the optimal $O(n)$ cost seems to be something like [this demo from burak](https://brqgoo.medium.com/emulating-op-vault-with-elements-opcodes-bdc7d8b0fe71) <small>(TL;DR: one special input performs all the amount checks, while the other inputs merely check the presence of the special input)</small>, which is not exactly ergonomic.
[/quote]

Thank you for sharing this, I wasn't aware of this work. I'll look into it. I'm always interested in alternative designs for amount locks :-).

[quote="salvatoshi, post:6, topic:549"]
More generally, any covenant opcode that constrains the destination seems to be pointless if the covenant opcode itself doesn’t *also* enforce the *presence* of the amount logic,
[/quote]

I think that is right. However I'm not sure the inverse is true. I think amount locks may be useful without destination locks. [See my coinjoin example](https://delvingbitcoin.org/t/op-inout-amount/549/3#p-4521-coinjoin-use-case-3). If privacy is your #1 concern, you want to enforce at the Script level that your utxos follow a specific amount pattern -- you can enforce the destinations with digital signatures as coinjoin protocols do today.

I also think we can make more general purpose primitives to implement amount locks rather than baking them into a single covenant opcode. I like the design of OP_CCV for implementing destination locks from what I've learned so far, but I'm not a big fan of the amount lock side of things.

[quote="salvatoshi, post:6, topic:549"]
I strongly believe that some amount of redundancy is unavoidable for any mechanism that extracts the amount logic out of the covenant opcode.
[/quote]

I'll think about this more.

Thank you for the thoughtful reply, you've given me a lot to think about. :slightly_smiling_face:

-------------------------

salvatoshi | 2025-05-08 19:41:45 UTC | #8

[quote="Chris_Stewart_5, post:7, topic:549"]
Why? By “it” I assume you are referring to `OP_CCV`/`OP_VAULT`. Is there some security consideration I am missing?
[/quote]

By "it" I meant  "a separate amount logic enforced with OP_{IN,OUT}_AMOUNT or similar opcodes".

[quote="Chris_Stewart_5, post:7, topic:549"]
This is tested in the [BIP345](https://delvingbitcoin.org/t/op-inout-amount/549/4) case study. I didn’t have this issue you are talking about. Both [`test_batch_unvault`](https://github.com/Christewart/bitcoin/blob/4c6c2ecb59a132eeac43d5608a1a1c081940b0e0/test/functional/feature_vaults.py#L362) and [`test_batch_recovery`](https://github.com/Christewart/bitcoin/blob/4c6c2ecb59a132eeac43d5608a1a1c081940b0e0/test/functional/feature_vaults.py#L251) test cases work with `OP_{IN,OUT}_AMOUNT`.
[/quote]

Perhaps I'm misunderstanding something, but here's the scenario I have in mind.

If you trigger two Vault UTXOs A and B of 1 ₿ each and spend A, B ==> U (where U is the "staging" or unvaulting output), you want the Script to enforce that U has amount at least 2 ₿.
If the input Script receive the input indices via the witness stack, what prevents "lying" to the inputs, by claiming that the input indices are the set $\{0\}$ in the witness stack of A, and the set $\{1\}$ in the witness stack of B? It seems to me that such transaction would be valid and allow stealing 1 ₿.

You can only prevent this kind of issues if you have some form of "synchronization" among the different input Scripts, that makes sure that the involved indices are the same for all the related inputs.

-------------------------

Chris_Stewart_5 | 2025-07-12 17:51:14 UTC | #9

# Case study: OP_CHECKTEMPLATEVERIFY

This case study explores how Script opcodes can be used to implement **amount locks**—restrictions that ensure the value of inputs and outputs in a transaction meets certain conditions. The goal is to evaluate the required features and developer ergonomics for opcodes that push input and output amounts onto the stack. Rather than starting from scratch, we build on existing opcode proposals and retrofit them to support amount locks directly in Script.

This requires two proposals I am working on:

1.  [64-bit arithmetic in Script](https://github.com/Christewart/bips/blob/79257ba5d7a632fa828208f266fd4f5540ffba7f/bip-XXXX.mediawiki)
2.  [`OP_IN_AMOUNT` & `OP_OUT_AMOUNT`](https://delvingbitcoin.org/t/op-inout-amount/549/3)

**Note:** This study does not attempt to implement _destination locks_—restrictions on where funds may be sent. That logic is preserved from the original proposal being examined.

[Here](https://github.com/Christewart/bitcoin/tree/2025-06-27-ctv-csfs-op-in-out-amount) is a link to the repository that implements everything discussed below. A good place to start reading is the functional test: [`feature_ctv_amount.py`](https://github.com/Christewart/bitcoin/blob/91815443c06b64858c532821d72c4e3a2b33aa52/test/functional/feature_ctv_amount.py).

# OP_CHECKTEMPLATEVERIFY

[`OP_CHECKTEMPLATEVERIFY`](https://github.com/bitcoin/bips/blob/83ac8427e7f81cead035728b9c1d925aceddf0d0/bip-0119.mediawiki) introduces a transaction template, a simple spending restriction that pattern matches a transaction against a hashed transaction specification. `OP_CTV` reduces many of the trust, interactivity, and storage requirements inherent with the use of pre-signing in applications.

This case study is interested in the [`AMOUNTVERIFY`](https://github.com/bitcoin/bips/blob/83ac8427e7f81cead035728b9c1d925aceddf0d0/bip-0119.mediawiki#user-content-OP_AMOUNTVERIFY) section of the `OP_CTV` proposal. We will implement safe [forwarding addresses](https://github.com/bitcoin/bips/blob/83ac8427e7f81cead035728b9c1d925aceddf0d0/bip-0119.mediawiki#user-content-Forwarding_Addresses) for OP_CTV that prevents users from
1. [Accidentally creating unsatisfiable UTXOs](https://delvingbitcoin.org/t/understanding-and-mitigating-a-op-ctv-footgun-the-unsatisfiable-utxo/1809?u=chris_stewart_5)
2. Creating overfunded UTXOs that must pay large miner fees

## Safe Forwarding Addresses with `OP_IN_AMOUNT`

A "safe" forwarding address, in this context, refers to creating additional spending paths within an `OP_CTV` output's script that can be utilized when the amount received by the `OP_CTV` hash lock does **not** match the exact amount committed in the hash. By leveraging a proposed opcode like `OP_IN_AMOUNT` (which allows introspection of input amounts), we can build custom logic to handle such discrepancies.

Let's illustrate with an example: Imagine our `OP_CTV` hash commits to spending an output that is expected to contain exactly 1 BTC. However, due to user error, only 0.9 BTC was actually sent to this `OP_CTV`-locked output. If we relied solely on the exact match specified by `OP_CTV` for a single input, this 0.9 BTC would become an **unsatisfiable UTXO**, permanently frozen.

However, by incorporating an "amount lock guard" using `OP_IN_AMOUNT`, we can define an alternative script path to recover these funds. Here's a conceptual representation of such a script:

```
OP_1,           # Push the index of the relevant input (e.g., the CTV input)
OP_IN_AMOUNT,   # Push the actual amount of the input at that index onto the stack
100000000,      # Push 1 BTC (in satoshis) onto the stack (the expected amount)
OP_EQUAL,       # Check if the actual funding amount equals the expected amount
OP_IF,          # If they are equal, execute the OP_CTV check
  withdrawl_tx_hash, # The hash of the template transaction
  OP_CHECKTEMPLATEVERIFY,
OP_ELSE,        # Otherwise (if amounts don't match)
  pub,          # Push a predefined public key
  OP_CHECKSIG,  # Use a standard OP_CHECKSIG with this pubkey to recover funds
OP_ENDIF

```

This script effectively encodes a "safe forwarding address" (or more accurately, a safe spending condition for the UTXO). If an incorrect amount of money is used to fund the `OP_CTV` output, the `OP_CHECKSIG` clause provides an accessible recovery mechanism. The funds, though misfunded, can still be spent by the holder of the `pub` key, preventing them from being irrevocably lost.

Furthermore, `OP_IN_AMOUNT` allows for more sophisticated recovery logic. For instance, you could differentiate between overfunded vs. underfunded scenarios:

-   **Underfunded:** Trigger a simple `OP_CHECKSIG` for recovery to a known address.
    
-   **Overfunded (by a small amount):** Perhaps apply a slightly higher miner fee to absorb the excess, or route it to a designated "dust" address.
    
-   **Overfunded by a _large_ amount (e.g., 100 BTC instead of 1 BTC):** For such significant sums, you could enforce even more stringent security policies, like requiring a 2-of-3 multisig to recover the funds, rather than a single key, adding an extra layer of protection.
    

The Python test suite demonstrating this concept can be found [here](https://github.com/Christewart/bitcoin/blob/91815443c06b64858c532821d72c4e3a2b33aa52/test/functional/feature_ctv_amount.py#L273).

----------

### Future Work

Next, I will be addressing Salvatoshi's insightful point regarding [amount replay attacks](https://delvingbitcoin.org/t/op-inout-amount/549/8?u=chris_stewart_5) in the context of `OP_IN_AMOUNT` and `OP_OUT_AMOUNT`. Stay tuned!

----------

-------------------------

