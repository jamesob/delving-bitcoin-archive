# OP_CHECKCONTRACTVERIFY and its amount semantic

salvatoshi | 2025-03-31 12:31:52 UTC | #1

I have done some work on formalizing the semantic of `OP_CHECKCONTRACTVERIFY`.  I also wrote the first draft BIP and an implementation in bitcoin-core:

Links:
- [BIP draft](https://github.com/bitcoin/bips/pull/1793)
- [bitcoin-core implementation draft](https://github.com/bitcoin/bitcoin/pull/32080)

In this post, I will briefly introduce the `OP_CCV` opcode semantic, then expand and discuss on an its amount-handling logic — an aspect that wasn't fully explored (and not properly formalized) in my previous posts on the topic. I will make the argument that it provides a convenient and compelling feature set that would be difficult to implement with more 'atomic' opcodes.

# OP_CHECKCONTRACTVERIFY
I start with a brief intro on `OP_CHECKCONTRACTVERIFY`, for the sake of making this post self-contained.

## in a nutshell: *state-carrying UTXOs*

`OP_CCV` enables *state-carrying UTXOs*: think of a UTXO as a locked box holding data, rules and an amount of coins. When you spend the UTXO, it lets you peek inside your own box (introspect the data), and decide the next box's rules (program), data and amount.

## less in a nutshell

`OP_CCV` works with P2TR inputs and outputs. It allows comparing the public key of an input.output with:
- a public key that we call the *naked key*
- ...optionally tweaked with the hash of some data
- ...optionally taptweaked with the Merkle root of a taproot tree.

Similarly to taproot, *tweaking* allows to create a commitment to a piece of data inside a public key. Therefore, by using a 'double' tweak, one can easily commit to an additional piece of arbitrary data.
An equality check of a double tweak with the input's taproot key is therefore enough to 'introspect' the current input's embedded data (if any), and an equality check with an output's taproot public key can force both the program (*naked key* and *taptree*) and the data of the output to be a desired value.

In combination with an opcode that allows to create a *vector commitment* (like `OP_CAT`, `OP_PAIRCOMMIT` or a hypothetical `OP_VECTORCOMMIT` opcode), it is of course possible to commit to multiple pieces of data instead of a single one.

While certainly not the only way to only way to implement a dynamic commitment inside a Script, this is the best I could come up with, and has some nice properties.

**Pro:**
- Fully compatible with taproot. Keypath spending is still available, and cheap.
- No additional burden for nodes; state is committed in the UTXO, not explicitly stored.
- It keeps the *program* (the taptree) logically separated from the *data*.
- Spending paths that do not require access to the embedded data (if any) do not pay extra witness bytes because of its presence.

**Con:**
- Only compatible with P2TR.
- A tweak is computationally more expensive than other approaches

# Amount logic

What about amounts?
It would be rather unusual to care about the output's Script, and then sending 0 sats to it. In most cases, you want to also specify how much money goes there.

Therefore, when checking an output, `OP_CCV` allows a convenient semantic to specify the amount flow in a way that works for most cases. There are three options:

- *default*: assign to the output the entire unassigned amount of the current input
- *deduct*: assign to the output the portion of current input's amount equal to this output's amount (the rest remain unassigned)
- *ignore*: only check the output's Script but not the amount.

An output can be used with the *default* logic from different inputs, but no output can be used as the target of both a *deduct* check and a *default* check, nor multiple *deduct* checks.

A well formed Script using `OP_CCV` would use:
- 0 or more `OP_CCV` with the *deduct* logic, assigning parts of the input amount to some outputs
- exactly 1 `OP_CCV` with the *default* logic, assigning the (possibly residual) amount to an(other) output.

This ensures that *all the amount of this input is accounted for in the outputs*.

## Examples

### 1-to-1
![A sends the entire amount to X|690x229, 75%](upload://1mlf0AhG7w6GRt5GcMGbZoWedys.png)

**A** uses `CCV` with the *default* logic to send all its amount to **X**.

This is common in all the situations where **A** wants to send the some predefined destination - either e terminal state, or another UTXO with a certain program and data.

In some cases, **X** might in turn be a copy of **A**'s program, after updating its embedded data. In this case, **A**'s Script would first use `CCV` to inspect the input's program and data, then compute the new data for the output, then use `CCV` with the *default* logic to check the output's program and data. This allows long-living smart contracts that can *update* their own state.

### many-to-1 (aggregate)
![**A**, B, C aggregate their amount towards X|690x229, 75%](upload://akbQQz4AQ5TQ4ZMcycOvz23xk16.png)

**A**, **B** and **C** uses `CCV` with the *default* logic to send all its amount to **X**.

### send partial amount

![A sends some 'to itself', the rest to X|690x229, 75%](upload://tbm607iNG8zKeImREfGKabeZKiM.png)

**A** checks the program and data (if any) of the input using `CCV`, then it checks that the first input **A'** has the same program/data using the *deduct* logic, then it uses `CCV` on the second output to send the residual amount to **X**.

This is used in constructions like vaults, to allow immediate partial revaulting. It could also be used in shared-UTXO schemes, to allow one of the users to withdraw their balance from the UTXO while leaving the rest in the pool (however, this would likely require a `CHECKSIG` or additional introspection on the exact amount going to X, which `CCV` alone can't provide).

(In other application, the partial amount could of course be sent to a completely different program, instead of **A'**)

### send partial amount, and aggregate

![Partial send from A, aggregate with B, toward X|690x229, 75%](upload://1v19tHec3yXWqGPxu1c7421DA25.png)

**A** does the same as the previous case, but here a separate input **B** also sends its amount  to **X** using the *default* logic, therefore aggregating its amount to the residual amount of **A** (after deducting the portion going to **A'**).

# Discussion

I've been suggested to separate the Script introspection of `CCV` from the amount logic. That's a possibility, however:

- you virtually *never* want to check an output Script without meaningful checks on their amount; so that would be wasteful and arguably less ergonomic.
- both the *default* and the *deduct* logic would be very difficult to implement with more atomic introspection opcodes.

Even enabling all the opcodes in the [Great Script Restoration](https://youtu.be/rSp8918HLnA) wouldn't be sufficient to easily emulate the amount logic described above. That's because the nature of these checks is inherently transaction-wide, and can't be expressed easily (or at all?) with individual input Script checks. Implementing it with a loop over all the involved inputs, it would result in quadratic complexity if every input performs this check. Of course, one could craft a single special input that checks the amount of all the other inputs (which in turn check the presence of the special input), but that's not quite an ergonomic solution - and to me it just feels wrong.

By incorporating this amount logic where the amount of the input is *assigned* to the outputs, the actual computations related to the constraint amounts are moved out of the Script interpreter.   

I think the two amount behaviors (*default* and *deduct*) are very ergonomic, and cover the vast majority of the desirable amount checks in practice. The only extension I can think of is direct equality checks on the output amounts, which could be added with a simple `OP_AMOUNT` (or [`OP_INOUT_AMOUNT`](https://delvingbitcoin.org/t/op-inout-amount/549)) opcode that pushes an input/output amount on the stack, or with a more generic introspection opcode like `OP_TXHASH`).

I implemented [fully featured vaults](https://github.com/Merkleize/pymatt/blob/e9cd2077880422ff74c3a5817c8affad74a0ed39/examples/vault/vault_contracts.py) using `OP_CCV + OP_CTV` that are roughly equivalent to `OP_VAULT + OP_VAULT_RECOVER + OP_CTV` in the python framework I developed for exploring MATT ideas. Moreover, a reduced-functionality version using just `OP_CCV` is implemented as a functional test in the bitcoin-core implementation of `OP_CCV`.

# Conclusions

I look forward to your comments on the specifications, implementation, and applications of `OP_CHECKCONTRACTVERIFY`,

-------------------------

instagibbs | 2025-03-17 13:42:37 UTC | #2

Thanks for finally writing a real life BIP!

Given the value forwarding piece is the most bike-sheddy part, I'm going to pick on that for now.

[quote="salvatoshi, post:1, topic:1527"]
(however, this would likely require a `CHECKSIG` or additional introspection on the exact amount going to X, which `CCV` alone can’t provide).
[/quote]

An alternative, which IIRC I proposed for OP_VAULT, was having explicit ability to introspect the (aggregate) output amounts.

Something like:
1) OP_IN_AMOUNT: pushes input amount on stack
2) CCV with value introspection: takes value off stack (can be >4 bytes), allocates to an output, and pushes residual of input back on stack, where residual is always the full amount minus the specified amount

This way value forwarding is always explicit when desired, no there's no default/deduct mode split, you can do math checks on the values (below a certain value), rate-limiting / collateral is pretty straight forward. Obviously it has downsides including that Bitcoin Script math is probably more footgun than feature in practice... but muh GSR?

-------------------------

Chris_Stewart_5 | 2025-03-17 16:59:40 UTC | #3

Hey Salvatoshi, looking through the code and BIP right now. I obviously come at this from thinking about handling amounts in Scripts. 

Does this opcode make sense to use in "fan-in" and "fan-out" patterns? Its unclear to me if `X` in the examples could also be `A''` OP_CCV?

### Fan-in

Does it make sense to aggregate multiple OP_CCV funding outputs into a single OP_CCV output?

### Fan-out 

Does it make sense to take a single OP_CCV funding output and split it into multiple OP_CCV outputs? Its unclear to me from the examples if `X` can be an OP_CCV as well


### Single input Single Output 

In the case where you have a transaction that takes a single OP_CCV input and a single OP_CCV output, how is fee handling expected to work? Is the idea that you just add an anchor output and the CPFP the parent? 

Or can you use the "deduct" feature to slice off a portion of the OP_CCV amount to use for a fee?

-------------------------

salvatoshi | 2025-03-17 18:25:33 UTC | #4

[quote="instagibbs, post:2, topic:1527"]
1. OP_IN_AMOUNT: pushes input amount on stack
2. CCV with value introspection: takes value off stack (can be >4 bytes), allocates to an output, and pushes residual of input back on stack, where residual is always the full amount minus the specified amount
[/quote]

Interesting approach, I'll think more about it. Although, having worked with it, I consider the opt-out semantic quite natural and enjoyable to work with, and it works for most cases.

I tend to think that bringing amount on the stack is too error prone to bring in without bignums, especially since you can start with 4-byte amounts and aggregate up to 5-byte ones.

If my conjecture is true that "equality checks is all you need" beyond CCV, you avoid doing math altogether.

-------------------------

salvatoshi | 2025-03-17 18:49:16 UTC | #5


Output Scripts in CCV-based state machines are more often than not completely unrelated scripts; however, you can obtain "same script as the input" by combining CCV on the input with CCV on the output; I use this for example in vaults for the partial revault).

So I think the answer is yes for all the variations you mentioned - except that in the *Fan-out* case, for it to make sense, you'd probably still need to introspect the various output amounts.

[quote="Chris_Stewart_5, post:3, topic:1527"]
In the case where you have a transaction that takes a single OP_CCV input and a single OP_CCV output, how is fee handling expected to work? Is the idea that you just add an anchor output and the CPFP the parent?
[/quote]

CCV doesn't put any limitation on the inputs/outputs that it doesn't introspect, so you can have separate input just for the fees, and a separate output just for the change. Or you could also use anchor outputs, package relay, or whatever is compatible with the relay policies.

[quote="Chris_Stewart_5, post:3, topic:1527"]
Or can you use the “deduct” feature to slice off a portion of the OP_CCV amount to use for a fee?
[/quote]
The amount you can use for fees is by definition not bound by the covenant restrictions, so I think either exogenous fees or anchors are inherent with any covenant construction.

-------------------------

AntoineP | 2025-04-09 16:06:58 UTC | #6

Hi,

Thanks for the concrete BIP draft and implementation. Something akin to `CCV` strikes me as elegant and potentially useful. However i think we should try to avoid the spillover effect across inputs you are introducing with the amount checks. I know there already exist some spillover effects through `CLTV`, but i think adding more such cases is the wrong direction to go.

I think it's possible to avoid by using an indirection akin to that of `CSV` for instance. The constraint(s) set on output amounts could be enforced as a new field in each input. Say in the annex. The field would specify a list of (constraint type, output index, optional amount) per input. For your use here the constraint types would be either sweep or deduct (equivalent to your *default* and *deduct*). A validator would go through the inputs of the transaction before executing the script (again, similarly to how relative timelocks are implemented). For each constraint in the annex (which may be none, your *ignore* option) it would record how much value must be set in the referenced output. The optional amount field is for the deduct constraint type, the amount to be deducted. Then in your Script you could have an operation which enforces the annex of the spending input has a given set of constraints.

I also think it makes sense to have separately from the `CCV` opcode as it seems like a useful primitive on its own to combine with other functionalities.

I have demonstrated this approach in a branch [here](https://github.com/darosior/bitcoin/tree/2504_hack_poc_annex_amounts) on top of Bitcoin Core v29.0. This is just a quick and dirty PoC, nothing like an actual proposal. But i hope it helps convey the idea. I have implemented the various semantics you describe in your post ("1-to-1", "many-to-1", "send partial amount, and aggregate") as a unit test. You can run it like so (after cloning the repo and checking out the branch):
```
cmake -B defbuild
cmake --build defbuild/ -j20
./defbuild/bin/test_bitcoin -t txvalidation_tests
```

-------------------------

salvatoshi | 2025-04-12 15:37:45 UTC | #7

[quote="AntoineP, post:6, topic:1527"]
However i think we should try to avoid the spillover effect across inputs you are introducing with the amount checks. I know there already exist some spillover effects through `CLTV`, but i think adding more such cases is the wrong direction to go.
[/quote]

Can you elaborate on why?
Validity is necessarily meaningful only at transaction-level. So I don't know what you mean by "spill-over" - these new validation rules are by definition at transaction-level, the same way that CHECKSIG is.

On the opposite direction, it might be interesting to attempt a refactoring/simplification of the validation code so that jobs in `ConnectBlock` receive _transactions_ to validate, rather than input scripts. That's where the implementation complexity of any kind of cross-input logic stems from (because of the added synchronization code), and I strongly suspect that input-level parallelism doesn't bring any measurable improvement in performance. Of course, this would need to be validated with benchmarks.


[quote="AntoineP, post:6, topic:1527"]
I think it’s possible to avoid by using an indirection akin to that of `CSV` for instance. The constraint(s) set on output amounts could be enforced as a new field in each input. Say in the annex. The field would specify a list of (constraint type, output index, optional amount) per input. For your use here the constraint types would be either sweep or deduct (equivalent to your *default* and *deduct*).
[/quote]

Thanks for demonstrating the approach. It would of course work, although I do see some downsides, while it's not too yet clear to me what are the benefits.

I think you can remove the optional amount from the constraint (which is not used in CCV, since in the *deduct* mode, the amount must equal exactly the output's amount).

The annex is only enforced by a signature, but signatures already cover output Scripts and amounts.
So this is only meaningful if the constraints are _also_ enforced in full by the opcode (CCV or any other new opcodes that might want to use this feature). Therefore, the opcode has to repeat in full the constraint as it appears in the annex, doubling the number of bytes needed to express it. That can be reduced to just a couple of bytes per constraint, so maybe that's not too bad for CCV in terms of byte cost.

Some care would also need to be taken to make sure that the annex is not malleable (as the input Script could use CCV without any signature).

As a side note, I think a solution using the annex would be very similar in implementation complexity to my previous attempt in [this diff](https://github.com/bitcoin-inquisition/bitcoin/compare/4e23c3a9867eedadb9e20387936ec9f0eca6e918...Merkleize:bitcoin:34f05028661932b417b59bdcdd58f4453f19cec5), on inquisition, based on the *deferred checks framework* from James O'Beirne's OP_VAULT PR (particularly, [this commit](https://github.com/bitcoin-inquisition/bitcoin/commit/32c9b122d72b3748051c979ce2d46f07a48c44cc)). While that avoids the explicit synchronization, it's quite a bit more complex than the code based on the mutex.

For an implementation based on the annex, I'd expect very similar complexity, as what's accumulated in the deferred checks is isomorphic to what would be in the annex in your approach.

-------------------------

Chris_Stewart_5 | 2025-04-12 20:12:21 UTC | #8

> [quote="salvatoshi, post:7, topic:1527"]  
> Can you elaborate on why?  
> [/quote]

I don’t personally agree with this, but the concern stems from the possibility of having *UTXOs with mutually exclusive spend conditions*. Let me walk through an example to illustrate.

It might helpful to revisit [BIP65](https://github.com/bitcoin/bips/blob/8375f71ee64cda848b159fa4a0c3719a48037492/bip-0065.mediawiki) for some background here.

Imagine I have two UTXOs in my wallet:

1. **UTXO A** has an `OP_CLTV` script that uses block height.  
2. **UTXO B** has an `OP_CLTV` script that uses wall clock time.

These UTXOs cannot be spent together in the same transaction because they both rely on the `nLockTime` field for validation. Since a transaction only has a single `nLockTime`, you can’t satisfy both scripts at the same time. (Note: `OP_CLTV` uses the same kind of "indirect validation" mechanism that `OP_CSV` does.)

This creates an awkward scenario for wallets: how do you convey that these two UTXOs are *mutually exclusive* in terms of spendability?

`OP_CSV` avoids this problem because it uses per-input `nSequence` values, so the same conflict doesn't arise. Each input can specify its own relative locktime.

[quote="salvatoshi, post:7, topic:1527"]
Validity is necessarily meaningful only at transaction-level.
[/quote]

I agree with this framing. Transactions are the atomic unit of validation in the Bitcoin network, and we should be able to introspect anything within the transaction.

I don’t think this principle extends to external factors like block time or block height as they aren't directly available in the transaction we are validating. That’s why the indirection used in BIP65/BIP112 is clever: the transaction’s `nLockTime`/`nSequence` values are validated before entering the interpreter, and inside the interpreter, we just read those fields and check them against the stack parameters given to `OP_CLTV`/`OP_CSV`.

However, this indirection *can* result in transactions that are valid in the interpreter (and thus valid under relay policy, I believe) but can never be mined. See this [PR](https://github.com/bitcoin/bitcoin/pull/32229) in Bitcoin Core for examples of `OP_CLTV` transactions that pass relay rules but are unmineable due to [BIP113](https://github.com/bitcoin/bips/blob/8375f71ee64cda848b159fa4a0c3719a48037492/bip-0113.mediawiki)’s `median-time-past` rules.

-------------------------

salvatoshi | 2025-04-13 10:13:16 UTC | #9

Thanks for the explanation. I would consider that a design flaw in CLTV (although there was probably no sane way of repurposing a transaction-level field like nLockTime without such 'bug', other than _not_ using it for both block height and wall time).

I think CCV doesn't have any such pathological situation: the only case when inputs cannot be spent together is when they violate the conditions on the output script or amounts as stipulated in the opcode itself.

CCV _does_ of course allows you to design UTXOs that can't be spent in the same transaction (e.g. a UTXO that requires the first output to be X, together with another UTXO that requires the first output to be Y != X) - but that's explicitly stipulated in the script, and can be avoided (if desired) by writing scripts with more flexible output indexes. (This situation is anyway not related to the cross-input logic)

-------------------------

AntoineP | 2025-04-14 15:06:40 UTC | #10

[quote="salvatoshi, post:7, topic:1527"]
Can you elaborate on why? Validity is necessarily meaningful only at transaction-level. So I don’t know what you mean by “spill-over” - these new validation rules are by definition at transaction-level, the same way that CHECKSIG is.
[/quote]

Sure, but CHECKSIG is precomputed before the execution of the scripts. In a sense, my suggestion achieves exactly the same for the amount constraints.

[quote="salvatoshi, post:7, topic:1527"]
On the opposite direction, it might be interesting to attempt a refactoring/simplification of the validation code so that jobs in `ConnectBlock` receive *transactions* to validate, rather than input scripts. That’s where the implementation complexity of any kind of cross-input logic stems from (because of the added synchronization code), and I strongly suspect that input-level parallelism doesn’t bring any measurable improvement in performance.
[/quote]

I think this is the wrong direction to take. There are a few reasons:
1. On its face, reducing parallelization (and most likely efficiency as a consequence) to match a proposal seems inappropriate if the same goal can be achieved without reducing parallelization.
2. Although this is implementation specific, i think breaking a property the implementation was able to rely upon raises questions about whether this property is desirable to keep when designing extensions to the Script language.
3. Input-level parallelization is key to reducing the validation time for unconfirmed transactions, where a would-be attacker does not have to expend a PoW to incur costs on a node, as well as worst case block validation times which adds up across a block propagation path.

[quote="salvatoshi, post:7, topic:1527"]
It would of course work, although I do see some downsides, while it’s not too yet clear to me what are the benefits.
[/quote]

Benefits include not breaking cross-input parallelization, a cleaner implementation and a more composable primitive (separation of concerns).

[quote="salvatoshi, post:7, topic:1527"]
Therefore, the opcode has to repeat in full the constraint as it appears in the annex, doubling the number of bytes needed to express it.
[/quote]

I agree the repetition isn't ideal, but i don't see a few weight units more as a major downside. Overall i think it's a good tradeoff to make.

[quote="salvatoshi, post:7, topic:1527"]
Some care would also need to be taken to make sure that the annex is not malleable (as the input Script could use CCV without any signature).
[/quote]

If your transaction doesn't have a signature, malleability already goes out the window.

[quote="salvatoshi, post:7, topic:1527"]
As a side note, I think a solution using the annex would be very similar in implementation complexity to my previous attempt in [this diff](https://github.com/bitcoin-inquisition/bitcoin/compare/4e23c3a9867eedadb9e20387936ec9f0eca6e918...Merkleize:bitcoin:34f05028661932b417b59bdcdd58f4453f19cec5)
[/quote]

Why "would"? I [demonstrated it](https://github.com/bitcoin/bitcoin/compare/v29.0...darosior:2504_hack_poc_annex_amounts) already, and it's *much* simpler.

[quote="salvatoshi, post:7, topic:1527"]
based on the *deferred checks framework* from James O’Beirne’s OP_VAULT PR (particularly, [this commit](https://github.com/bitcoin-inquisition/bitcoin/commit/32c9b122d72b3748051c979ce2d46f07a48c44cc)).
[/quote]

In a sense you could see my suggestion as "preemptive" ("eager"?), rather than "deferred", checks. Existing features, for instance absolute locktimes, could have also been designed to have transaction-level checks be deferred to after executing the inputs scripts. Instead they were designed with transaction-level checks performed beforehand (surely because for those the field already existed in the transaction), and i think it is much cleaner.

-------------------------

