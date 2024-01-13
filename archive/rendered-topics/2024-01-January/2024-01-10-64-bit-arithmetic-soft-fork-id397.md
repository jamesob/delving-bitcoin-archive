# 64 bit arithmetic soft fork

Chris_Stewart_5 | 2024-01-11 13:10:10 UTC | #1

Hi all, when reviewing OP_TLUV mailing list posts it became clear that 64bit arithmetic would be a subset of necessary changes to enable something like TLUV. I've heard from other protocol developers that 64bit arithmetic operations would be useful to their protocols as well. 

Here is a live link to my BIP proposal: https://github.com/bitcoin/bips/pull/1538

and the implementation: 

https://github.com/bitcoin/bitcoin/pull/29221

I figured I would try posting here to see if there is any early feedback before sending to the mailing list. Apologies if this isn't the right format for this venue.

-------------------------

moonsettler | 2024-01-10 23:10:38 UTC | #2

ACK 64 bit arithmetics in general.

aside from TLUV, it would also be useful for any more detailed introspection, like TXHASH, CAT, elements opcodes, MATT.

-------------------------

halseth | 2024-01-11 14:07:42 UTC | #3

Concept ACK. I think most proposed covenant opcodes would really benefit from being able to do arithmetics on satoshi values.

-------------------------

sipa | 2024-01-11 14:11:33 UTC | #4

No opinion on the value of such a consensus change, but I'm confused why this is introducing a whole new encoding for numbers, as opposed to just expanding how big the input values to the existing opcodes can be through e.g. an `OP_ENABLE64BIT`? That would avoid duplicating all the opcodes (whose namespace isn't infinite), and reduce possible confusion.

-------------------------

Chris_Stewart_5 | 2024-01-11 14:24:52 UTC | #5

> t I’m confused why this is introducing a whole new encoding for numbers, as opposed to just expanding how big the input values to the existing opcodes can be through e.g. an `OP_ENABLE64BIT`

I don't understand this. Little endian is a pretty conventional encoding system no? I think the goal should be to remove weirdness when possible in favor of conventional number systems. Could you link me to more information on this so I can read about it?

>That would avoid duplicating all the opcodes (whose namespace isn’t infinite), and reduce possible confusion.

IMO, on the next witness version we should remove OP_ADD/OP_SUB/comparison ops and just force people to use 64 bit versions. That would remove confusion.

-------------------------

sipa | 2024-01-11 14:54:55 UTC | #6

[quote="Chris_Stewart_5, post:5, topic:397"]
I don’t understand this. Little endian is a pretty conventional encoding system no? I think the goal should be to remove weirdness when possible in favor of conventional number systems.
[/quote]

Sure, little endian is very conventional, and it'd be a reasonable choice if you're building something from scratch. But I don't think there is anything weird with the existing encoding though, it's minimal-length big endian, which for literals inside the script has the advantage of being more compact than forcing a full length constant. Moreover, it has the enormous benefit of being already implemented, tested, deployed, and in use.

[quote="Chris_Stewart_5, post:5, topic:397"]
Could you link me to more information on this so I can read about it?
[/quote]

There is a [footnote](https://github.com/bitcoin/bips/blob/deae64bfd31f6938253c05392aa355bf6d7e7605/bip-0342.mediawiki#cite_note-1) in BIP324:

> `OP_SUCCESSx` may also redefine the behavior of existing opcodes so they could work together with the new opcode. For example, if an `OP_SUCCESSx`-derived opcode works with 64-bit integers, it may also allow the existing arithmetic opcodes in the *same script* to do the same.

This seems strictly better to me. It needs fewer new opcodes, doesn't require a new integer encoding, and leaves the ability to use short literals.

[quote="Chris_Stewart_5, post:5, topic:397"]
IMO, on the next witness version we should remove OP_ADD/OP_SUB/comparison ops and just force people to use 64 bit versions. That would remove confusion.
[/quote]

Sorry, that's just baffling to me. You want to push the entire ecosystem to switch to a different number encoding because you think the existing one is a little strange?

-------------------------

Chris_Stewart_5 | 2024-01-11 15:08:42 UTC | #7

ok just to make sure I understand your concerns

1. You are skeptical of a new encoding - strict 64 bit little endian requirements that would increase witness sizes rather than requiring minimal encoding (I believe `fRequireMinimal` is the flag in `interpreter.cpp`). You also don't want to create more confusion by switching from BE -> LE.
2. IIUC - you are NOT skeptical about expanding arithmetic support from 51 bit support to 64 bit support.

> `OP_SUCCESSx` may also redefine the behavior of existing opcodes so they could work together with the new opcode. For example, if an `OP_SUCCESSx`-derived opcode works with 64-bit integers, it may also allow the existing arithmetic opcodes in the *same script* to do the same.

How would this be deployed with existing v1 tapscripts if we redefine the semantics of the arithmetic op codes? Wouldn't this run into issues of potentially breaking Scripts already deployed with v1? 

I understand that OP_SUCCESSx was intended to allow state modification of the stack, I just don't get the compatibility story with existing v1 Scripts.

-------------------------

sipa | 2024-01-11 15:23:39 UTC | #8

[quote="Chris_Stewart_5, post:7, topic:397"]
You are skeptical of a new encoding - strict 64 bit little endian requirements that would increase witness sizes rather than requiring minimal encoding (I believe `fRequireMinimal` is the flag in `interpreter.cpp`). You also don’t want to create more confusion by switching from BE → LE.
[/quote]

Yes. I also think it's better not to encroach too much on available opcode space.

[quote="Chris_Stewart_5, post:7, topic:397"]
IIUC - you are NOT skeptical about expanding arithmetic support from 51 bit support to 64 bit support.
[/quote]

Eh, current script arithmetic opcodes are limited to 4-byte inputs, which means 32 bits signed integers.

I have no comment on whether we should pursue changing that.

[quote="Chris_Stewart_5, post:7, topic:397"]
How would this be deployed with existing v1 tapscripts if we redefine the semantics of the arithmetic op codes? Wouldn’t this run into issues of potentially breaking Scripts already deployed with v1?
[/quote]

One possibility is having an `OP_ENABLE64BIT`, which has no effect on the stack when executed, but its presence in a script makes all arithmetic opcodes accept up to 8-byte inputs for example. This will not affect any existing scripts because existing scripts don't have `OP_ENABLE64BIT` in them. It has the advantage of, even if `OP_ENABLE64BIT` is present, not changing semantics of *valid* existing scripts. And it is softfork-safe because the mere presence (not even execution) of `OP_ENABLE64BIT` makes a script anyone-can-spend according to the existing consensus rules.

Alternatively, a new taproot leaf version could be used too. That's even more compact.

-------------------------

Chris_Stewart_5 | 2024-01-11 17:19:48 UTC | #9

> One possibility is having an `OP_ENABLE64BIT`...

Ok I understand what you are proposing but honestly seems like introducing even more complexity into Script.

>Alternatively, a new taproot leaf version could be used too. That’s even more compact.

This seems like a better idea IMO. This hasn't been done before, so implementing this would mean having access to `leaf_version` in `EvalScript()` and then building conditional logic inside of these arithmetic opcodes based on `leaf_version`?

>Sorry, that’s just baffling to me. You want to push the entire ecosystem to switch to a different number encoding because you think the existing one is a little strange?

Coming back to this, yes. **I say this with the utmost respect**, I understand why the numbering system isn't confusing to you or other long time bitcoin developers. You've been working with it for a very long time. For newer developers, it is much easier to reason about things they have learned elsewhere in their software development career. Simple rules like things like inputs are always 8 bytes in length (not variable) make it much easier to reason about. If you would prefer big endian to be used rather than little endian I can see the value if that - although little endian is used elsewhere in the protocol.

My understanding is the alternative implementation you are suggesting means modifying `CScriptNum` to support 64 bits. This introduces a ton of consensus risk for prior Scripts deployed. I was specifically recommended **not to touch CScriptNum** as it is hard to reason about.

Perhaps I am not understanding what this alternative implementation looks like, so please correct me if I am wrong.

IIUC - you do not have malleability concerns with this 8 byte proposal as 8 byte sizes would be required.

>it’s minimal-length big endian, which for literals inside the script has the advantage of being more compact than forcing a full length constant.

Just to make sure we are talking about the same thing, by literals you mean `OP_1,OP_2..` etc right? I think this is a fair critique as -- IIUC -- now you would have to have `OP_1` and `OP_1_64` or something like that I believe? Or else you would have to have special interpretation logic for pushing 8 byte values or 1 byte values onto the stack based on what the witness/leaf version is?

-------------------------

ProofOfKeags | 2024-01-11 16:46:12 UTC | #10

[quote="sipa, post:8, topic:397"]
One possibility is having an `OP_ENABLE64BIT`, which has no effect on the stack when executed, but its presence in a script makes all arithmetic opcodes accept up to 8-byte inputs for example
[/quote]

I'm generally skeptical of increasing the surface area of the state variable of the script VM. I'll admit I don't know script backwards and forwards but all of my experience with it so far points me to the fact that the only thing opcodes operate over is the main and alt stacks as well as halting execution of the VM. As such it *seems* that the existing VM *can* be modeled by a state transition function:

```
exec_op : (MainStack, AltStack) -> (MainStack, AltStack)
```

Doing something like `OP_ENABLE64BIT` would introduce some third `VMInterpreterState` and I think that will dramatically increase the surface area of potential consensus issues. Correct me if I'm wrong but this would be a fundamentally new structural change to the VM formalism.

-------------------------

Chris_Stewart_5 | 2024-01-11 17:19:08 UTC | #11

[quote="ProofOfKeags, post:10, topic:397"]
Doing something like `OP_ENABLE64BIT` would introduce some third `VMInterpreterState` and I think that will dramatically increase the surface area of potential consensus issues. Correct me if I’m wrong but this would be a fundamentally new structural change to the VM formalism.
[/quote]

I think Scripts like this would be confusing imo

```
OP_ADD OP_SUB OP_ENABLE64BIT OP_ADD OP_SUB ...
```

So `ENABLE64BIT` would essentially be changing interpretation of "local" op codes (rather than setting how they should be interpreted at a global level).

In my example we would be combining these numbering systems within the same Script. If the `leaf_version` path is taken, we can reason at a global level in the interpreter that all arithmetic ops should be interpreted as 64 bit ops.

-------------------------

ajtowns | 2024-01-11 17:39:55 UTC | #12

[quote="Chris_Stewart_5, post:11, topic:397"]
I think Scripts like this would be confusing imo

```
OP_ADD OP_SUB OP_ENABLE64BIT OP_ADD OP_SUB ...
```
[/quote]

You could define `ENABLE64BIT` to result in script failure unless it is the first opcode (apart from any other `OP_SUCCESSx` ops -- that way future upgrades that introduced `ENABLExyz` opcodes could then relax that restriction to allow ENABLE* opcodes to be in any order at the start of the script). You could have that requirement enforced even if `ENABLE64BIT` occurred in an unexecuted `IF/ELSE` branch.

-------------------------

ajtowns | 2024-01-11 17:42:51 UTC | #13

[quote="sipa, post:8, topic:397"]
One possibility is having an `OP_ENABLE64BIT`, which has no effect on the stack when executed, but its presence in a script makes all arithmetic opcodes accept up to 8-byte inputs for example.
[/quote]

The semantics proposed here copy those from liquid/elements -- that is maths opcodes like ADD and SUB tend to leave two two values to the stack, on the top, a boolean TRUE/FALSE indicating whether the operands were in range, and if they were, underneath that, the actual result of the operation. That's pretty different to the way bitcoin's existing operations work, so I'm not sure modifying the existing opcodes to such a different new behaviour makes sense.

-------------------------

ProofOfKeags | 2024-01-11 17:55:35 UTC | #14

[quote="Chris_Stewart_5, post:11, topic:397"]
I think Scripts like this would be confusing imo
[/quote]

It's not just the scripts themselves. Writing a correct Script interpreter in this regime becomes more complicated too. Script interpreters have to match each other *exactly* if we don't want consensus splits.

We should be moving towards formally specifying the Script VM and things like a hypothetical `OP_ENABLE64BIT` make that harder than just putting new 64bit opcodes into the VM. I do recognize there is a desire to not eat up extra opcode space, but I feel the need to state that introducing this new invisible interpreter altering semantic is probably a pandoras box we don't want to open.

-------------------------

sipa | 2024-01-11 20:40:26 UTC | #15

[quote="ajtowns, post:13, topic:397"]
The semantics proposed here copy those from liquid/elements – that is maths opcodes like ADD and SUB tend to leave two two values to the stack, on the top, a boolean TRUE/FALSE indicating whether the operands were in range, and if they were, underneath that, the actual result of the operation. That’s pretty different to the way bitcoin’s existing operations work, so I’m not sure modifying the existing opcodes to such a different new behaviour makes sense.
[/quote]

I was suggesting not changing any semantics at all; only changing the acceptable range of inputs to existing opcodes. If `OP_MUL` or a variant thereof is added, I can see why detecting/dealing with overflows becomes an issue that the existing interface doesn't deal well with.

If the different semantics are actually desirable, then I agree it shouldn't reuse the existing opcodes. Even if so, I don't see the benefit of introducing a different encoding.

[quote="Chris_Stewart_5, post:9, topic:397"]
Simple rules like things like inputs are always 8 bytes in length (not variable) make it much easier to reason about. If you would prefer big endian to be used rather than little endian I can see the value if that - although little endian is used elsewhere in the protocol.
[/quote]

Ok, I'll accept that the variable-length approach complicates things a bit, but I also think having two different encodings is even worse. All things being equal, I prefer little-endian over big-endian, but again, two encodings is worse than one.

[quote="Chris_Stewart_5, post:9, topic:397"]
My understanding is the alternative implementation you are suggesting means modifying `CScriptNum` to support 64 bits.
[/quote]

It's a wrapper around `int64_t` with serialization, deserialization, and arithmetic operations that assert on overflow. I think you'd want to add overflow-detecting versions for your use, but otherwise it already does everything. I don't think there would be a need to touch any of the existing functions/operators on `CScriptNum`.

There is a restriction on the input length when converting a Script stack element to a `CScriptNum`; my suggesting was to just relax that restriction from 4 bytes to 8 bytes when in 64-bit mode (whether that's through an `OP_SUCCESSx`, through a leaf version, or through a separate opcode). Note that `OP_CHECKLOCKTIMEVERIFY` also uses `CScriptNum`, but permits arguments up to 5 bytes rather than 4.

[quote="Chris_Stewart_5, post:9, topic:397"]
IIUC - you do not have malleability concerns with this 8 byte proposal as 8 byte sizes would be required.
[/quote]

Fair point. So far, I have seen few use cases for integer values that are script *inputs*, but if you envision that changing, that would be a point in favor of a strict encoding (that could still be minimally-encoded integers in a variable-length regine, which is effectively already a policy rule).

[quote="Chris_Stewart_5, post:9, topic:397"]
Just to make sure we are talking about the same thing, by literals you mean `OP_1,OP_2..` etc right? I think this is a fair critique as – IIUC – now you would have to have `OP_1` and `OP_1_64` or something like that I believe?
[/quote]

Yeah, the `OP_n` opcodes, plus direct pushes of integer encodings (e.g. the stack element for encoding the number 20 has no `OP_n`, but you can push the 0x14 byte using a direct push instruction). Duplicating all the `OP_n` opcodes seems like a pain, so a conversion opcode after the literal would make more sense. Alternatively, don't introduce a separate encoding, so the semantics of `OP_n` remains the same in both worlds.

[quote="ProofOfKeags, post:10, topic:397"]
As such it *seems* that the existing VM *can* be modeled by a state transition function:
[/quote]

It cannot. There are at least also:
* The position of the last executed `OP_CODESEPARATOR`, as it affects the sighashes.
* The if/then/else conditional stack (which branches are we in)
* In tapscript, the remaining checksig budget

[quote="ProofOfKeags, post:10, topic:397"]
Doing something like `OP_ENABLE64BIT` would introduce some third `VMInterpreterState` and I think that will dramatically increase the surface area of potential consensus issues. Correct me if I’m wrong but this would be a fundamentally new structural change to the VM formalism.
[/quote]

It's certainly an increase; I don't think it is dramatic at all. But fair enough, I'm convinced that a separate leaf version is cleaner than an `OP_ENABLE64BIT` here.

-------------------------

ProofOfKeags | 2024-01-11 21:01:22 UTC | #17

[quote="sipa, post:15, topic:397"]
It’s certainly an increase; I don’t think it is dramatic at all.
[/quote]

ACK. Fair enough. Thanks for the clarification.

-------------------------

halseth | 2024-01-12 13:20:15 UTC | #18

[quote="sipa, post:6, topic:397"]
Moreover, it has the enormous benefit of being already implemented, tested, deployed, and in use.
[/quote]

I find the arguments pretty convincing; if we can enable 64-bit arithmetics using _the existing_ `CSScriptNum` implementation, that sounds desirable. 


I do see the benefits of having a more approachable number format available for new developers, but `CScriptNum` is already there so they kinda have to deal with it in some form anyway.

[quote="sipa, post:6, topic:397"]
it’s minimal-length big endian, which for literals inside the script has the advantage of being more compact than forcing a full length constant.
[/quote]

If we introduce fixed-size LE version in addition to the existing minimal-length BE, this could possibly give developers incentives to convert between them and only use the fixed-length types when needed to save on space. Not saying this is a dealbreaker, but it could slow the clean transition to a 64 bit number version.  

[quote="Chris_Stewart_5, post:9, topic:397"]
Perhaps I am not understanding what this alternative implementation looks like, so please correct me if I am wrong.
[/quote]

There might be some edge cases in using the existing `CScriptNum` type for 8 byte values, but if I understand @sipa correctly, it could be as easy as this: https://github.com/halseth/bitcoin/pull/1/commits/13c1848edf66410517b3cb6d47d80874438abb1f

(This includes 64-bit support for all the numeric opcodes, including OP_WITHIN, OP_1ADD etc)

We already have `leaf_version` available from the interpreter, so it's just about defining a new one.

[quote="sipa, post:15, topic:397"]
If `OP_MUL` or a variant thereof is added, I can see why detecting/dealing with overflows becomes an issue that the existing interface doesn’t deal well with.
[/quote]
If we want to re-enable these opcodes, it would be nice indeed to have them be backwards compatible with the existing number format (still with a `leaf_version` bump of course).

-------------------------

rustyrussell | 2024-01-12 16:22:29 UTC | #19

I think the modal hack is awkward, and mildly prefer a new set of opcodes, but I prefer them to be general rather than require conversion codes. This can be done most simply by having new opcodes only deal with unsigned numbers.

I also have a preference for larger precision: 64 bits can be limiting (though not as limiting as 31 in a protocol with larger amounts!).

Finally, it's worth considering carefully what interactions would occur with reenabling MUL, LSHIFT, SUBSTR and the like.

In case you missed it, please consider: https://rusty.ozlabs.org/2023/12/30/arithmetic-opcodes.html

-------------------------

dgpv | 2024-01-13 14:26:46 UTC | #20

I'd like to note on 'adding extra encoding for numbers'. While extra encoding might add some burden in one place, it might lessen the burden in another place.

In complex covenants the computed values are often used to compare/combine with introspected values taken from the transaction, and the encoding of that values are usually LE64 or LE32.

In my experience designing covenant scripts on Elements, it definitely feels that there is value in being able to do computations using the same format as the numbers that are stored within the transaction, and with clearly defined ways to detect overflows etc.

-------------------------

dgpv | 2024-01-13 15:11:03 UTC | #21

On the second thought, maybe checking for overflows after each 64-bit arithmetic opcode is not that great, if we can have computations in (practically) arbitrary width integers and then only detect overflows on conversion to LE64/LE32.

(but really, adding VERIFY after each 64-bit arith op does not add any mental load - it just increases script size a bit)

(well, but forgetting VERIFY after 64-bit arith op in Elements might lead to unexpected behavior in case of actual overflow...)

-------------------------

Chris_Stewart_5 | 2024-01-13 14:53:41 UTC | #22

[quote="dgpv, post:20, topic:397"]
I’d like to note on ‘adding extra encoding for numbers’. While extra encoding might add some burden in one place, it might lessen the burden in another place.
[/quote]

Dovetailing this, if a new developer comes to the project and wants to do some simple numerical computations in Script they currently have to learn about policy (`SCRIPT_VERIFY_MINIMALDATA`). How great would it be to be able to remove policies (reduce complexity, imo) with this change. 

Perhaps this could also include removing the `SCRIPT_VERIFY_MINIMALIF` as well.

'Removing' is probably a strong word, but no longer having to take these policy flags into account for future soft forks.

-------------------------

Chris_Stewart_5 | 2024-01-13 14:59:37 UTC | #23

[quote="Chris_Stewart_5, post:9, topic:397"]
Just to make sure we are talking about the same thing, by literals you mean `OP_1,OP_2..` etc right? I think this is a fair critique as – IIUC – now you would have to have `OP_1` and `OP_1_64` or something like that I believe? Or else you would have to have special interpretation logic for pushing 8 byte values or 1 byte values onto the stack based on what the witness/leaf version is?
[/quote]

[I've added another post to talk about altering interpretation of op codes based on leaf version](https://delvingbitcoin.org/t/deploying-new-taproot-leaf-versions/406). My suggestion would be to alter the interpretation of `OP_0,OP_1` etc to push 8 byte values onto the stack (instead of minimal encodings) if a specific leaf version is found. 

This would maintain our invariant that numeric values are always 8 bytes when they are pushed onto the stack, but not necessarily 8 bytes (the exception is our literals, `OP_0,OP_1..`) when in the script.

-------------------------

dgpv | 2024-01-13 15:00:18 UTC | #24

[quote="Chris_Stewart_5, post:22, topic:397"]
Perhaps this could also include removing the `SCRIPT_VERIFY_MINIMALIF` as well.
[/quote]

MINIMALIF is already consensus-enforced for tapscript

-------------------------

Chris_Stewart_5 | 2024-01-13 15:03:10 UTC | #25

Could this check be modified to take into account leaf versions or would that be a hard fork? I'm still learning about what consensus rules are possible to modify with tapscript leaf versions

https://github.com/bitcoin/bitcoin/blob/3ba8de1b704d590fa4e1975620bd21d830d11666/src/script/interpreter.cpp#L613

-------------------------

