# OP_CHECKCONTRACTVERIFY and its amount semantic

salvatoshi | 2025-03-17 12:29:05 UTC | #1

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

In combination with an opcode that allows to create a *vector commitment* (like `OP_CAT`, `OP_PAIRCOMMIT` or an hypothetical `OP_VECTORCOMMIT` opcode), it is of course possible to commit to multiple pieces of data instead of a single one.

While certainly not the only way to only way to implement a dynamic commitment inside a Script, this is the best I could come up with, and has some nice properties.

**Pro:**
- Fully compatible with taproot. Keypath spending is still available, and cheap.
- No additional burden for nodes; state is committed in the UTXO, not explicitly stored.
- It keeps the *program* (the taptree) logically separated from the *data*.
- Spending paths that do not require access to the embedded data (if any) do not pay extra witness bytes because of its present.

**Con:**
- Only compatible with P2TR.
- A tweak is computationally more expensive than other approaches

# Amount logic

What about amounts?
It would be rather unusual to care about the output's Script, and then sending 0 sats to it. In most cases, you want to also specify how much money goes there.

Therefore, when checking an output, `OP_CCV` allows a convenient semantic to specify the amount flow in a way that works for most cases. There are three options:

- *default*: assign to the output the entire unassigned amount the current input
- *deduct*: assign to the output the portion of current input's amount equal to this output's amount (the rest remain unassigned
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

