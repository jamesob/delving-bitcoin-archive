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

