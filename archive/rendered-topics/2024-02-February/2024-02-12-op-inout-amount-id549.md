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

Chris_Stewart_5 | 2025-03-13 18:49:13 UTC | #3

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

This script checks that the output at index 0 is equal to 1BTC. It also checks there exists at least 1 input that is equal to 1BTC. The caller of this Script can specify the input index to check that its equal to exactly 1BTC.

If you have multiple inputs that have exactly 1BTC, 3rd parties on the network could malleate the witness stack and still have the bitcoin transaction be valid.

-------------------------

