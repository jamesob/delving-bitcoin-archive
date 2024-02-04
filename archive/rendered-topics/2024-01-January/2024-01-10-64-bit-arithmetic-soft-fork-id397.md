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

sipa | 2024-01-18 13:09:39 UTC | #6

[quote="Chris_Stewart_5, post:5, topic:397"]
I don’t understand this. Little endian is a pretty conventional encoding system no? I think the goal should be to remove weirdness when possible in favor of conventional number systems.
[/quote]

Sure, little endian is very conventional, and it'd be a reasonable choice if you're building something from scratch. But I don't think there is anything weird with the existing encoding though, it's minimal-length ~~big endian~~ (EDIT: little-endian with sign-magnitude encoding rather than two's complement), which for literals inside the script has the advantage of being more compact than forcing a full length constant. Moreover, it has the enormous benefit of being already implemented, tested, deployed, and in use.

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

sipa | 2024-01-18 13:11:02 UTC | #15

[quote="ajtowns, post:13, topic:397"]
The semantics proposed here copy those from liquid/elements – that is maths opcodes like ADD and SUB tend to leave two two values to the stack, on the top, a boolean TRUE/FALSE indicating whether the operands were in range, and if they were, underneath that, the actual result of the operation. That’s pretty different to the way bitcoin’s existing operations work, so I’m not sure modifying the existing opcodes to such a different new behaviour makes sense.
[/quote]

I was suggesting not changing any semantics at all; only changing the acceptable range of inputs to existing opcodes. If `OP_MUL` or a variant thereof is added, I can see why detecting/dealing with overflows becomes an issue that the existing interface doesn't deal well with.

If the different semantics are actually desirable, then I agree it shouldn't reuse the existing opcodes. Even if so, I don't see the benefit of introducing a different encoding.

[quote="Chris_Stewart_5, post:9, topic:397"]
Simple rules like things like inputs are always 8 bytes in length (not variable) make it much easier to reason about. If you would prefer big endian to be used rather than little endian I can see the value if that - although little endian is used elsewhere in the protocol.
[/quote]

Ok, I'll accept that the variable-length approach complicates things a bit, but I also think having two different encodings is even worse. All things being equal, I prefer ~~little-endian over big-endian~~ (EDIT: two's complement over sign-magnitude), but again, two encodings is worse than one.

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

dgpv | 2024-01-13 15:12:01 UTC | #21

On the second thought, maybe checking for overflows after each 64-bit arithmetic opcode is not that great, if we can have computations in (practically) arbitrary width integers and then only detect overflows on conversion to LE64/LE32.

(but really, adding VERIFY after each 64-bit arith op does not add any mental load - it just increases script size a bit)

(well, forgetting VERIFY after 64-bit arith op in Elements might lead to unexpected behavior in case of actual overflow...)

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

ajtowns | 2024-01-15 04:22:58 UTC | #26

[quote="sipa, post:6, topic:397"]
But I don’t think there is anything weird with the existing encoding though, it’s minimal-length big endian, which for literals inside the script has the advantage of being more compact than forcing a full length constant.
[/quote]

I don't understand this comment -- `CScriptNum` is little endian in the first place, isn't it? Both my understanding of [the code](https://github.com/bitcoin/bitcoin/blob/3ba8de1b704d590fa4e1975620bd21d830d11666/src/script/script.h#L357-L361) and [definition](https://en.wikipedia.org/wiki/Endianness) suggests it is, and the [wiki](https://en.bitcoin.it/wiki/Script) seems to agree, at least.

The differences between `CScriptNum` and this proposal aiui are fixed vs variable length encoding, and using twos-complement vs a sign bit to handle negative numbers.

-------------------------

jamesob | 2024-01-16 17:43:34 UTC | #27

1. I think it would be a (potentially big) waste of chainspace to move from minimally encoded numbers to fixed-length 64 bit numbers. Minimal encodings may be somewhat cumbersome to deal with, but they're there for a reason!
2. The only thing worse than dealing with one weird encoding is having to implement another one alongside it. `CScriptNum` parsing will probably always be a part of wallet software, if not just to validate legacy scripts.

-------------------------

Davidson | 2024-01-17 22:31:06 UTC | #28

I wonder how unfeasible is it to bring back the 256 bits arithmetic. Now that we have taproot, we can have some fancy crypto inside script, but that wouldn't go onchain most of the time due to the happy-path aspect of taproot.

Imagine if we could make some convoluted sigma protocol onchain, but it's only there in case of disputes. We can make those cost some extra sigops to discourage abusing that.

-------------------------

Chris_Stewart_5 | 2024-01-19 21:27:03 UTC | #29

> I think it would be a (potentially big) waste of chainspace to move from minimally encoded numbers to fixed-length 64 bit numbers

I don't think this is the case. First off - literals (`OP_0`,`OP_1`,`OP_2`..) can just be re-interpreted based on sig version. That means the space they consume will remain at 1 byte, however when they are pushed onto the stack they will no longer be 1 byte - rather 8 bytes. This increases _memory consumption_, not disk space.

EDIT: Found a bug in my results of historical scanning of the blockchain, will come back and update this section after my bug is fixed and my results are accurate.

We can speculate what future blockchain usage patterns will look like, but lets be honest that is just speculation.

>but they’re there for a reason!

Yes! They are. But not for the reason you mentioned. 

Lets examine the ill fated [BIP62](https://github.com/bitcoin/bips/blob/deae64bfd31f6938253c05392aa355bf6d7e7605/bip-0062.mediawiki#user-content-Number). BIP62 was _intended to solve transaction malleability_, not optimizing for disk space.

Wrt to script numbers:

> **Zero-padded number pushes** Any time a script opcode consumes a stack value that is interpreted as a number, it must be encoded in its shortest possible form. 'Negative zero' is not allowed. See reference: [Numbers](https://github.com/bitcoin/bips/blob/deae64bfd31f6938253c05392aa355bf6d7e7605/bip-0062.mediawiki#numbers).

This is meant to solve malleability that was occurring on the network that would cause bitcoin businesses to lose track of withdrawals from their business (IIUC - MtGox suffered from this, again why i'm suspicious of my results).

If we [examine the commit](https://github.com/bitcoin/bitcoin/commit/698c6abb25c1fbbc7fa4ba46b60e9f17d97332ef) that introduced the `fRequireMinimal` flag in the interpreter, it specifically cites BIP62 rule #3 and #4 as for the reasoning for introducing it. It does not cite disk space usage.

As a by product of this proposed change, I actually believe we could potentially simplify `interpreter.cpp` for future soft forks by removing the need for `SCRIPT_VERIFY_MINIMALDATA` - how great is that?! Admittedly, this proposal does not cover encoding for push operations. I'm going to research expanding this proposal to cover these to see what effects it would have.

I'm also suspicious that the only reason we have `CScriptNum` is because it wrapped the openSSL number implementation. I've tried to track this down on github history to verify, but i'm unable to. Perhaps an OG can comment on this to see if this is correct. [Here is as far as I got in github archaeology](https://github.com/bitcoin/bitcoin/commits/7cd0af7cc222d0694ce72e71458aef460698ee2c/src/bignum.h?browsing_rename_history=true&new_path=src/test/bignum.h&original_branch=2b2ddc558e1cddb5ff54fd2d9e375793021a908e)

> The only thing worse than dealing with one weird encoding is having to implement another one alongside it.

Yes, so why are you advocating for keeping the 2nd (arguably 3rd, if you count `CompactSize` ints) format? I'm advocating for consolidating to a single format. As it exists today, we have two number encodings in the bitcoin protocol. The traditional int64_t one (which satoshi values are encoded in, and is of interest for a lot of BIP proposals) and our exotic one.

It is my understanding we were given the 2nd format by satoshi (I haven't got to the bottom of this myself to confirm this) and have had to modify it multiple times to remove malleability.

>`CScriptNum` parsing will probably always be a part of wallet software, if not just to validate legacy scripts.

I think 'validate' is a bit of a strong word as I'm not aware of a lot of external implementations of `interpreter.cpp` (although there are a few! [bitcoin-s has one](https://github.com/bitcoin-s/bitcoin-s/blob/e6ceda44f2ab54d3cc9cfd45303010ed6e95660a/core/src/main/scala/org/bitcoins/core/script/interpreter/ScriptInterpreter.scala)), but they definitely need to be able to build Scripts for their wallet software to receive/withdraw funds. 

I don't think this proposal would affect wallets that do standard signing logic (`p2pkh`, `p2wpkh`, `p2trkp`). I believe if your Script does not use `OP_PUSHDATAx` this holds true for `p2sh`,`p2wsh`, `p2trsp`, although I would like others to think about this make sure my mental model is correct. `OP_PUSHDATAx` is a relatively infrequent opcode, so I suspect that there isn't wide usage of these (although I haven't looked too much into the NFT world, where it might make sense to use `OP_PUSHDATAx`)

My view is that we should move away from exotic things like this in the bitcoin protocol in favor of standard encodings in the rest of the software ecosystem. While there will be an adjustment period, in 10 years people would look back on these changes say 'remember when you had a numbering system in bitcoin's programming language that was _different_ than other protocol number representations? What a nightmare! Glad someone decided to consolidate that.' 

Lets relegate this numbering system to a fun historical fact best talked about over a :beer: to establish your OG cred :slightly_smiling_face:.

-------------------------

dgpv | 2024-01-20 05:01:16 UTC | #30

[quote="Chris_Stewart_5, post:29, topic:397"]
we could potentially simplify `interpreter.cpp` for future soft forks by removing the need for `SCRIPT_VERIFY_MINIMALDATA`
[/quote]

The interpreter needs to support all the historical scripts, so I doubt very much it will be possible removing such code parts. At most this flag will be consensus-enforced.

-------------------------

Chris_Stewart_5 | 2024-01-20 12:57:31 UTC | #31

Yes, which is why I said future soft forks. A better way to state this would be: 

Future Script programmers would no longer need to consider `SCRIPT_VERIFY_MINIMAL_DATA` when writing their Scripts. "Legacy" Script programmers will always need to consider them for policy reasons - unless for some reason policy changes (and I don't see why it would).

Perhaps that was worded a bit clunky, apologies.

-------------------------

Chris_Stewart_5 | 2024-01-20 13:16:16 UTC | #32

[quote="Chris_Stewart_5, post:29, topic:397"]
I believe if your Script does not use `OP_PUSHDATAx` this holds true for `p2sh`,`p2wsh`, `p2trsp`
[/quote]

I'm still exploring what the scope of this proposal _could include_ while trying to adhere to [my rule about soft forks](https://twitter.com/Chris_Stewart_5/status/1748361651651289305) :-). @jamesob got me thinking about the 2 paths forward via a DM.

I'm trying to decide where to draw the line between requiring all existing op codes to take 8 byte inputs (in a backwards compatible way via a `SigVersion`), or just adding these arithmetic op codes and allowing people to use the conversion op codes to 'cast' stack tops to the appropriate input size (`OP_LE64TOSCRIPTNUM`, `OP_SCRIPTNUMTOLE64`, `OP_LE32TOLE64`) for op codes that pre-date this soft fork proposal. 

This proposal currently does the latter, would like to hear others input on this to see if the juice is worth the squeeze with requiring all inputs to be 8 bytes to existing op codes (i.e. `OP_CLTV`, `OP_CSV`, `OP_WITHIN`, `OP_DEPTH`...)

This comment also is a bit confusing as of course legacy Scripts will not need to be rewritten (`p2sh`, `p2wsh`, `p2trsp`). 

If you want to upgrade to use this new proposed soft fork that require 8 byte inputs for operations such as `OP_CLTV`, this would require Script programmers to upgrade their Scripts to use the soft fork. 

If we don't require 8 byte inputs for anything besides the new numeric op codes (`OP_ADD64`, `OP_SUB64`, ...) the upgrade story is pretty easy, but we retain the 2nd encoding.

-------------------------

dgpv | 2024-02-02 16:37:03 UTC | #33

[quote="rustyrussell, post:19, topic:397"]
In case you missed it, please consider: [Arithmetic Opcodes: What Could They Look Like? | Rusty Russell’s Quiet Corner of The Internet ](https://rusty.ozlabs.org/2023/12/30/arithmetic-opcodes.html)
[/quote]

I liked the ideas in that article. Dealing only with non-negative numbers might simplify some things.

Also removing "normalization" from LSHIFT and RSHIFT is the right thing - Elements implementations remove leading zeroes, and coupled with sign bit that is not actually considered after the shift, it makes modelling these opcodes with Z3 needlessly complex, and also normalization can introduce malleability: `5 LSHIFT` can take 0x01, 0x0100, 0x010000 with the same result.

Relying on `__uint128_t` support in GCC and clang is a bit iffy... Would that mean that other compilers will be somehow discouraged ?

I think there's still need for a way to easily convert to fixed-with numbers. Maybe a generic opcode that would take number of bytes, and zero-pads or truncates if necessary ? Pobably truncation should succeed only if removed bytes are all zero, and script should fail otherwise

For example, `4 FIXNUM` would make LE32 from the argument, `8 FIXNUM` will make LE64, `32 FIXNUM` will make a 256-bit number

-------------------------

Chris_Stewart_5 | 2024-01-23 16:23:25 UTC | #34

>I think it would be a (potentially big) waste of chainspace...

Ok finally have some results from this. 

The proposal to shift from minimal encodings to 8 byte encodings for ScriptNums would result in roughly `1,176,417,034` (~1GB) larger blockchain. My current local blockchain size is

```
du -sh ~/.bitcoin/blocks
576G    /home/chris/.bitcoin/blocks
```

This means this proposal would increase the blockchain size by is `0.17%` since the genesis block.

I can share my json file used to calculate these, its large (13GB). Here is an example of a few txs, along with [a link to my source code for calculating these json values](https://github.com/Christewart/bitcoin-s-core/tree/2024-01-19-count-scriptnums)

```
{"txIdBE":"3ef4fe026fbea7160f6bc55288db14df2889f088f5a5654d21acaff3e1bb7197","scriptConstants":["04ec091a","18","05","1d"],"sizeIncrease":25,"comment":"ASM"},
{"txIdBE":"91d7d2c1cae734481e621f33c226f23bad257ea28f47cfb45f076f44c732887e","scriptConstants":["04ec091a","17","05"],"sizeIncrease":18,"comment":"ASM"},
{"txIdBE":"cfc7b512ec29507ea5f576ff6185f580047f72e3edfef30f783a6d3fec236d03","scriptConstants":["456c6967697573","74657374","b630"],"sizeIncrease":11,"comment":"ASM"},
{"txIdBE":"fc61cbe788267b631448d001c98bdb45ac70bf8bfd0a1df2b634e4255fcd6664","scriptConstants":["456c6967697573","74657374","d6fd00"],"sizeIncrease":10,"comment":"ASM"},
{"txIdBE":"e3853d6faa369b20c31369526af6c6c2d25686260805185d2b5afa8dbd98602a","scriptConstants":["456c6967697573","fe","bea800"],"sizeIncrease":13,"comment":"ASM"},
``` 

I'm working on testnet3 results, i'll edit those in when I'm done with them.

-------------------------

halseth | 2024-01-23 20:36:15 UTC | #35

[quote="Chris_Stewart_5, post:29, topic:397"]
I don’t think this is the case. First off - literals (`OP_0`,`OP_1`,`OP_2`…) can just be re-interpreted based on sig version. That means the space they consume will remain at 1 byte, however when they are pushed onto the stack they will no longer be 1 byte - rather 8 bytes. This increases *memory consumption*, not disk space.
[/quote]

I think this is only true for the 0-16 numbers. The moment you want to push > 16 onto the stack, you have to add the full 8-byte representation to your script.

This leads me towards thinking we should keep variable length encoding the default, as we could move to the new format without incurring extra cost.

That being said, having worked with RISC-V emulation in Bitcoin Script lately (see [Eltrace](https://github.com/halseth/elftrace)), I see a real need for the ability to get the 32/64 bit LE representation of a number during script execution. That we could easily add independently of the underlying number format (`OP_SCRIPTNUM[TO/FROM]LE64`), and perhaps give us all what we need without introducing full arithmetic support for another format.

[quote="dgpv, post:20, topic:397"]
In complex covenants the computed values are often used to compare/combine with introspected values taken from the transaction, and the encoding of that values are usually LE64 or LE32.
[/quote]
I think this is an important point. The reason we want 64-bit arithmetics in the first place is to enforce values on the next transaction, and the interesting values are indeed (AFAIK) encoded using fixed-length LE (notable exception is number of inputs/outputs).

How to handle this comes down to the introspection opcodes themselves, as they can be made to put `ScriptNum` on the stack if that's what we want. 

[quote="rustyrussell, post:19, topic:397"]
In case you missed it, please consider: [Arithmetic Opcodes: What Could They Look Like? | Rusty Russell’s Quiet Corner of The Internet ](https://rusty.ozlabs.org/2023/12/30/arithmetic-opcodes.html)
[/quote]
Thanks for this writeup, Rusty! I agree moving to unsigned-only values could simplify a lot of things, especially if the format is backwards compatible with the existing `ScriptNum` representation for positive numbers.

By bumping the leaf version you could also re-use all existing opcodes (no `OP_ADDV` etc)?

-------------------------

EthanHeilman | 2024-02-01 22:23:40 UTC | #36

@Chris_Stewart_5 

Continuing my question from BIP. 

The BIP states: "If the operation results in an overflow, push false onto the stack".

I would propose altering this to so that the result and the overflow amount are pushed onto the stack.

**Case 1, no overflow:**

1, 1, OP_ADD64 -->  2, 0 

where 2 is the result and 0 is the overflow amount.

**Case 2, overflow:**

2^64 - 1, 5, OP_ADD64 --> 4, 1 

where 4 is the result (mod 2^64), and 1 is the overflow amount. You can think of these two stack values as representing the 128 bit number 2^64+4 broken into two 64 bit chunks.

Push values on the stack in this fashion makes it esimple use these 64 opcodes to do math on numbers larger than 64 bits by chunking them into 64-bit stack elements. The overflow amount tells you how much to carry into the next chunk.

You'd still get the benefit of having a flag to check, if overflow amount is not 0, overflow occurred. 

I think the BIP as written lets you add numbers larger than 64-bits using a similar chunking approach, but it is less straight forward and requires IF statements and substructions operations.

-------------------------

dgpv | 2024-02-02 05:25:46 UTC | #37

[quote="EthanHeilman, post:36, topic:397"]
I would propose altering this to so that the result and the overflow amount are pushed onto the stack.
[/quote]

The overflow amount would be LE64,  but then the zero in case of 'no overflow' would also need to be LE64 (otherwise more-than-64bit calculations would still require branching), and that would complicate the common case of checking for 'no overflow', because you cannot just do `NOT VERIFY` - `NOT` expects a scriptnum

-------------------------

dgpv | 2024-02-02 16:50:27 UTC | #38

If more-than-64bit arithmetic is desired, then I would say rustyrussell's idea of using only-positive variable-length integers where larger lengths are allowed is more elegant, and coupled with `FIXNUM` (or maybe `TOFIXNUM`/`FROMFIXNUM`) opcode to convert between variable and fixed encodings, and `BYTEREV` to convert between low/big endian, that would cover a large range of practical needs

-------------------------

EthanHeilman | 2024-02-02 18:46:32 UTC | #39

> If more-than-64bit arithmetic is desired, then I would say rustyrussell’s idea of using only-positive variable-length integers where larger lengths are allowed is more elegant,

I agree variable length would be more elegant. My worry was about fee pricing since, depending on the math operations allowed, really big nums require more computation than small numbers. You can put big numbers on the stack with single byte opcodes like HASH256. 64-bit chucks provides a very nice way to capture that increase in cost since need more opcodes to perform operations on bigger numbers. 

The more I think about my fee cost argument here, the more convinced I am that it doesn't matter and I am wrong. It seems perfectly fine to have operations on very large numbers in Bitcoin because if one wanted to maximize computational resources spent and minimize fee code that are much better opcodes than arithmetic. 

The computational cost of performing arithmetic on even very large numbers, say 520 byte numbers, is probably much smaller than many single opcode instructions like HASH160 or CHECKSIG. I don't have actual performance numbers on this though.

-------------------------

ajtowns | 2024-02-03 12:02:57 UTC | #40

[quote="EthanHeilman, post:39, topic:397"]
The computational cost of performing arithmetic on even very large numbers, say 520 byte numbers, is probably much smaller than many single opcode instructions
[/quote]

I think there's a few choices here:

 * what happens with "overflows" ?
    * they're not possible, everything happens modulo $2^n$
    * if the inputs are "too large", the script aborts
    * you get an overflow indicator in every result
 * what is $2^n$ or what constitutes "too large" ?
     * BTC's max supply is about $2^{51}$ satoshis, so 64bit is a fine minimum
     * $2^{256}$ would let you manipulate scalars for secp256k1 which might be useful
     * $2^{4160}$ would let you do numbers of to 520 bytes which matches current stack entry limits
 * unsigned only, or signed
    * if we're doing modular arithmetic, unsigned seems easier
    * signed maths probably makes some contracts a bunch easier though
 * what serialization format?
    * if signed, use a sign-bit or 2's complement?
    * fixed length or variable length -- if fixed length, that constrains the precision we could use
    * if variable length, does every integer have a unique serialization, or can you have "0000" and "0" and "-0" all as different representations of 0?

I guess 64-bit unsigned modular arithmetic would be easiest (`uint64_t` already works), but 4160-bit signed numbers (or some similarly large value that benchmarks fast enough) with abort-on-overflow behaviour and stored/serialized like CScriptNum format might be more appropriate?

Abort-on-overflow seems kind of appealing as far as writing contracts goes: we've already had an "overflow allows printing money" bug in bitcoin proper, so immediately aborting if that happens in script seems like a wise protection. If people want to do modular arithmetic, they could perhaps manually calculate `((x % k) * (y % k)) % k` provided they pick $k \le \sqrt{2^{n}}$ or similar.

-------------------------

dgpv | 2024-02-03 16:04:35 UTC | #41

[quote="ajtowns, post:40, topic:397"]
if variable length, does every integer have a unique serialization, or can you have “0000” and “0” and “-0” all as different representations of 0?
[/quote]

I thought that this is not an option, since it introduces malleability that MINIMALDATA was added to eliminate

-------------------------

ajtowns | 2024-02-04 07:30:50 UTC | #42

If your goal is to allow varying size big nums so that you can do maths in the secp scalar field, you don't want to enforce minimaldata -- otherwise if you take the hash of a blockheader to generate a scalar, and naturally end up with leading zeroes, you have to strip those zeroes before you can do any maths on that hash. Stripping an arbitrary number of zeroes is then also awkward with minimaldata rules, though not impossible.

(I think a minimaldata variant that only applies to values pulled directly from the witness stack, but not from program pushes or the results of operations) would be interesting)

-------------------------

dgpv | 2024-02-04 07:44:12 UTC | #43

[quote="ajtowns, post:42, topic:397"]
you have to strip those zeroes before you can do any maths on that hash
[/quote]

if there is `FROMFIXNUM` opcode that takes the size of fixed-size integer as one argument and byte blob of that size as another argument and turns it into a variable-size integer it will be easy (and also `BYTEREV` to deal with endianness)

-------------------------

