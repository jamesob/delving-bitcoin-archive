# Overflow handling in Script

Chris_Stewart_5 | 2025-03-31 15:40:38 UTC | #1

# Overflow handling in Script

## Motivation

There is interest in enhancing Script's functionality. Re-enabling new opcodes one of the main feature enhancements that have been proposed. This overflow limitation has been talked about in the Rusty Russell's [Great Script Restoration](https://rusty.ozlabs.org/2023/12/30/arithmetic-opcodes.html) project, as well as the [64-bit arithmetic delving bitcoin post](https://delvingbitcoin.org/t/64-bit-arithmetic-soft-fork/397).

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

### Bitcoin Cash

Bitcoin cash seems to have re-added disabled opcodes such as [OP_MUL](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/blob/master/src/script/interpreter.cpp#L884) and [OP_DIV](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/blob/master/src/script/interpreter.cpp#L895). They have decided to modify the exception handling behavior by [setting `serror` exception](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/blob/master/src/script/interpreter.cpp#L887) if the result overflows. Note, this is different than how bitcoin works currently as we catch an exception thrown by `CScriptNum`.

### Zcash / Litecoin

Both [Zcash](https://github.com/zcash/zcash/blob/a3435336b0c561799ac6805a27993eca3f9656df/src/script/script.h#L209) and [Litecoin](https://github.com/litecoin-project/litecoin/blob/5fba5ad7c13c59d1e0854dd51ac0c22bea68c8f8/src/script/script.h#L254) have retained the original behavior from bitcoin as far as I can tell.

### Other bitcoin derivatives?

If you know of any interesting design choices made by other forks of bitcoin, please share them below!

## Future research

As mentioned above, CScriptNum currently [throws an exception if an overflow occurs](https://github.com/bitcoin/bitcoin/blob/998386d4462f5e06412303ba559791da83b913fb/src/script/script.h#L249). If this is used in conjunction with an op code that interprets the stack top as a CScriptNum, this propagates the exception in to `EvalScript()` causing the main execution loop to be wrapped in a [`try` `catch` block](https://github.com/bitcoin/bitcoin/blob/998386d4462f5e06412303ba559791da83b913fb/src/script/interpreter.cpp#L1225). Questions I have are

1. What are other opcodes that can result in an exception that are caught by this `try` `catch` block?
2. What are the performance implications of wrapping the meat and potatoes of `EvalScript()` in a `try` `catch` block?

While we will likely never be able to fully migrate from the `try` `catch` block, perhaps future soft forks could avoid having this logic?

-------------------------

