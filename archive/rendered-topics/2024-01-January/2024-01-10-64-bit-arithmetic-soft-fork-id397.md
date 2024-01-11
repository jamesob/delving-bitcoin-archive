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

