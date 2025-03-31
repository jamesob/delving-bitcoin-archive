# Overflow handling in Script

Chris_Stewart_5 | 2025-03-31 20:04:35 UTC | #1

# Overflow handling in Script

## Motivation

There is interest in enhancing Script's functionality. Re-enabling opcodes is one of the main feature enhancements that have been proposed. This overflow limitation has been talked about in the Rusty Russell's [Great Script Restoration](https://rusty.ozlabs.org/2023/12/30/arithmetic-opcodes.html) project, as well as the [64-bit arithmetic delving bitcoin post](https://delvingbitcoin.org/t/64-bit-arithmetic-soft-fork/397).

Here are some highlights

[quote="sipa, post:15, topic:397"]
If `OP_MUL` or a variant thereof is added, I can see why detecting/dealing with overflows becomes an issue that the existing interface doesn’t deal well with.
[/quote]

[quote="sipa, post:15, topic:397"]
I think you’d want to add overflow-detecting versions for your use, but otherwise it already does everything.
[/quote]

[quote="dgpv, post:20, topic:397"]
it definitely feels that there is value in being able to do computations using the same format as the numbers that are stored within the transaction, and with clearly defined ways to detect overflows etc.
[/quote]

[quote="dgpv, post:21, topic:397, full:true"]
On the second thought, maybe checking for overflows after each 64-bit arithmetic opcode is not that great, if we can have computations in (practically) arbitrary width integers and then only detect overflows on conversion to LE64/LE32.

(but really, adding VERIFY after each 64-bit arith op does not add any mental load - it just increases script size a bit)

(well, forgetting VERIFY after 64-bit arith op in Elements might lead to unexpected behavior in case of actual overflow…)
[/quote]

I would like to consolidate discussion on the _overflow handling_ part of the Great Script restoration project here.

While not an overflow exactly, you can imagine a world where `OP_DIV` is re-enabled. This error handling logic could also be used for the case where the user attempts to divide by 0.


## Background

[CScriptNum](https://github.com/bitcoin/bitcoin/blob/770d39a37652d40885533fecce37e9f71cc0d051/src/script/script.h#L226) is the data type used in bitcoin core to handle numbers in Script. Here is what happens when a stack element is interpreted as a CScriptNum that results in an overflow

>operands must be in the range [-2^31 +1 to 2^31 -1]

This means inputs to opcodes such as `OP_ADD`, `OP_SUB`, `OP_1ADD` etc must fall within the stated range. However

>results may overflow (and are valid as long as they are not used in a subsequent numeric operation). CScriptNum enforces those semantics by storing results as an int64 and allowing out-of-range values to be returned as a vector of bytes but throwing an exception if arithmetic is done or the result is interpreted as an integer.

Simply put, if my Script is 

> 2^31-1 OP_1ADD 

the result on the stack would be `2^31`, which is a valid result. If I tried to do an `OP_1ADD` on `2^31` an [this exception would occur](https://github.com/bitcoin/bitcoin/blob/770d39a37652d40885533fecce37e9f71cc0d051/src/script/script.h#L249)

which then would result in this error being propagated to the user

>mandatory-script-verify-flag-failed (unknown error)

This is because the exception thrown in `CScriptNum`'s constructor is caught in `EvalScript()`'s `catch` clause

https://github.com/bitcoin/bitcoin/blob/770d39a37652d40885533fecce37e9f71cc0d051/src/script/interpreter.cpp#L1227

## What design options are out there?

Here is the design space as I see it, please comment below if I've missed any designs that have been deployed

### Elements project
[In 2022](https://github.com/ElementsProject/elements/blob/811d8359600477b38088d12f7a686291fdad211f/doc/tapscript_opcodes.md#new-opcodes-for-additional-functionality) the Elements project introduced a new set of opcodes to handle the case of overflowing numbers in Script. This upgrade also added 64 bits of precision, 64 bit specific opcodes, and a new encoding format -- all of which we will ignore for the purposes of this post. 

This soft fork introduces _overflow handling_ when arithmetic computations are performed with the new 64 bit opcodes in elements.

>When dealing with overflows, we explicitly return the success bit as a `CScriptNum` at the top of the stack and the result being the second element from the top. If the operation overflows, first the operands are pushed onto the stack followed by success bit. [`a_second` `a_top`] overflows, the stack state after the operation is [`a_second` `a_top` `0`] and if the operation does not overflow, the stack state is [`res` `1`].

>This gives the user flexibility to deal if they script to have overflows using `OP_IF\OP_ELSE` or `OP_VERIFY` the success bit if they expect that operation would never fail. When defining the opcodes which can fail, we only define the success path, and assume the overflow behavior as stated above.

AJ Towns provided a critique to this design in the 64-bit arithmetic thread

[quote="ajtowns, post:50, topic:397"]
FWIW, the big concern I have with this is people writing scripts where they don’t think an overflow is possible, so they just do an OP_DROP for the overflow indicator, and then someone thinks a bit harder, and figures out how to steal money via an overflow, and then they do exactly that. That’s arguably easily mitigated: just use `OP_VERIFY` to guarantee there wasn’t an overflow, but I noticed that an example script in [review club](https://bitcoincore.reviews/29221#l-86) used the more obvious DROP:

```
<Chris_Stewart_5> Script: 0x000e876481700000 0x000e876481700000 OP_ADD64 OP_DROP OP_LE64TOSCRIPTNUM OP_SIZE OP_8 OP_EQUALVERIFY OP_SCRIPTNUMTOLE64 0x001d0ed902e00000 OP_EQUAL
```

Worries me a bit when the obvious way of doing something (“this won’t ever overflow, so just drop it”) is risky.

You could imagine introducing two opcodes: “OP_ADD64” and “OP_ADD64VERIFY” the latter of which does an implicit VERIFY, and hence fails the script if there was overflow; but that would effectively be the existing behaviour of OP_ADD. So I guess what I’m saying is: maybe consider an approach along the lines that sipa suggested:
[/quote]


### Bitcoin Cash

Bitcoin cash seems to have re-added disabled opcodes such as [OP_MUL](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/blob/master/src/script/interpreter.cpp#L884) and [OP_DIV](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/blob/master/src/script/interpreter.cpp#L895). They have decided to modify the exception handling behavior by [setting `serror` exception](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/blob/master/src/script/interpreter.cpp#L887) if the result overflows. Note, this is different than how bitcoin works currently as we catch an exception thrown by `CScriptNum`.

### Zcash / Litecoin

Both [Zcash](https://github.com/zcash/zcash/blob/a3435336b0c561799ac6805a27993eca3f9656df/src/script/script.h#L209) and [Litecoin](https://github.com/litecoin-project/litecoin/blob/5fba5ad7c13c59d1e0854dd51ac0c22bea68c8f8/src/script/script.h#L254) have retained the original behavior from bitcoin as far as I can tell.

### Other bitcoin derivatives?

If you know of any interesting design choices made by other forks of bitcoin, please share them below!

### Ethan Heilmans suggested design

[quote="EthanHeilman, post:36, topic:397"]
The BIP states: “If the operation results in an overflow, push false onto the stack”.

I would propose altering this to so that the result and the overflow amount are pushed onto the stack.

**Case 1, no overflow:**

1, 1, OP_ADD64 → 2, 0

where 2 is the result and 0 is the overflow amount.

**Case 2, overflow:**

2^64 - 1, 5, OP_ADD64 → 4, 1

where 4 is the result (mod 2^64), and 1 is the overflow amount. You can think of these two stack values as representing the 128 bit number 2^64+4 broken into two 64 bit chunks.

Push values on the stack in this fashion makes it esimple use these 64 opcodes to do math on numbers larger than 64 bits by chunking them into 64-bit stack elements. The overflow amount tells you how much to carry into the next chunk.

You’d still get the benefit of having a flag to check, if overflow amount is not 0, overflow occurred.

I think the BIP as written lets you add numbers larger than 64-bits using a similar chunking approach, but it is less straight forward and requires IF statements and substructions operations.
[/quote]

with suggestions from AJ Towns

[quote="ajtowns, post:40, topic:397"]
I think there’s a few choices here:

* what happens with “overflows” ?
  * they’re not possible, everything happens modulo 2^n2n2^n
  * if the inputs are “too large”, the script aborts
  * you get an overflow indicator in every result
* what is 2^n2n2^n or what constitutes “too large” ?
  * BTC’s max supply is about 2^{51}2512^{51} satoshis, so 64bit is a fine minimum
  * 2^{256}22562^{256} would let you manipulate scalars for secp256k1 which might be useful
  * 2^{4160}241602^{4160} would let you do numbers of to 520 bytes which matches current stack entry limits
* unsigned only, or signed
  * if we’re doing modular arithmetic, unsigned seems easier
  * signed maths probably makes some contracts a bunch easier though
* what serialization format?
  * if signed, use a sign-bit or 2’s complement?
  * fixed length or variable length – if fixed length, that constrains the precision we could use
  * if variable length, does every integer have a unique serialization, or can you have “0000” and “0” and “-0” all as different representations of 0?

I guess 64-bit unsigned modular arithmetic would be easiest (`uint64_t` already works), but 4160-bit signed numbers (or some similarly large value that benchmarks fast enough) with abort-on-overflow behaviour and stored/serialized like CScriptNum format might be more appropriate?

Abort-on-overflow seems kind of appealing as far as writing contracts goes: we’ve already had an “overflow allows printing money” bug in bitcoin proper, so immediately aborting if that happens in script seems like a wise protection. If people want to do modular arithmetic, they could perhaps manually calculate `((x % k) * (y % k)) % k` provided they pick k \le \sqrt{2^{n}}k≤√2nk \le \sqrt{2^{n}} or similar.
[/quote]


## Future research

As mentioned above, CScriptNum currently [throws an exception if an overflow occurs](https://github.com/bitcoin/bitcoin/blob/998386d4462f5e06412303ba559791da83b913fb/src/script/script.h#L249). If this is used in conjunction with an op code that interprets the stack top as a CScriptNum, this propagates the exception in to `EvalScript()` causing the main execution loop to be wrapped in a [`try` `catch` block](https://github.com/bitcoin/bitcoin/blob/998386d4462f5e06412303ba559791da83b913fb/src/script/interpreter.cpp#L1225). Questions I have are

1. What are other opcodes that can result in an exception that are caught by this `try` `catch` block?
2. What are the performance implications of wrapping the meat and potatoes of `EvalScript()` in a `try` `catch` block?

While we will likely never be able to fully migrate from the `try` `catch` block, perhaps future soft forks could avoid having this logic?

-------------------------

