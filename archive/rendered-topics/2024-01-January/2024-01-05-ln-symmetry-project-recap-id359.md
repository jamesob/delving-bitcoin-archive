# LN-Symmetry Project Recap

instagibbs | 2024-01-05 18:03:45 UTC | #1

# LN-Symmetry Project

ln-symmetry started with a simple idea: Someone should take the [eltoo](https://blockstream.com/eltoo.pdf)
idea and take it as far as possible to prove out the concept, without requiring community consensus required
actions like softforks before proving it out.

The end result is research-level quality software, with basic functional tests of the common cases:
1) channel opens
2) payments (revamped + simplified channel state machine)
3) payments with hops 
4) unilateral closes
5) Reconnection logic (with tests!)

It did not implement:
1) cooperative closes: There's a lot of spec work to make this better for today's channels, and it would nearly be copy-paste, so I didn't bother
2) Proper persistence of the new keytypes/fields. In other words, if you restart your node, you'll crash. Couldn't quite figure it out
3) anchor spending support: Rusty was working hard at the time on `option_anchors` support; rebasing everything on top now should fix this
4) Gossip for these channels. So if you're playing around with it, you need to use them as private channels

I also got these opens/closes [broadcasted on signet via bitcoin-inquisition](https://mempool.space/signet/tx/9d63375a596ffd2be55624049580267ae07bf464e4d2fddc4043c7c3be79944c#vout=0):


# Key Focuses

1) KISS: I felt almost too stupid to understand the current BOLTs. How much shorter and simpler can I make them with a new channel type using APO-like functionality? Answer is quite a bit, at least within BOLTs 2, 3, and 5.
3) De-risking eltoo technical challenges: What are we missing in our handwaves of an idea? There's always something. Is that something fatal to the idea?

# Key Takeaways

1) Pinning is super hard to avoid. At least 1/3 of the work I bet was designing the system to be as robust against pinning as possible. I think I completed the task, except for HTLC-Success cases which actually need to be pre-signed to be pin-resistant. It's my biggest remaining design bug I know about.
2) eltoo-style channels have even longer than expect htlc expiry deltas to stay secure. No one had actually worked through the state machine before.
3) Much more flexible fee management: All outputs in settlement transactions(except HTLC-Success paths as noted before) are free to spend, allowing endogenous fees from all other output cases of balance and HTLC outputs. Reduces the burden on having a utxo pool for fees to only pay maximum once per unilteral close during the "challenge" period.
4) CTV(emulation) ended up being useful! It removed the necessity of round-trips from the payment protocol, allowing for "fast forwards" that are extremely simple, and would likely reduce payment times if widely adopted. 
5) I'm less convinced than ever that penalties are the way forward in the future. Penalties are, in practice against a competent adversary, only as large as their reserve requirements, so "make sure it works right" seems to be the best deterrent. Make the expected chance of success essentially zero, and the incentive should be to collaboratively close to save fees.

# Key work artifacts

1) Tons of anti-pinning research resulting in many milestones in the [Package Relay Project](https://github.com/bitcoin/bitcoin/issues/27463) including my novel invention of [Ephemeral Anchors](https://github.com/instagibbs/bips/blob/d33cdbd0777700f4fc488d54b90a8795d2c33639/bip-ephemeralanchors.mediawiki) which builds on top of "V3" concept.
2) CLN implementation here: https://github.com/instagibbs/lightning/tree/eltoo_support
3) Bitcoin Core branch with necessary changes for CLN blackbox tests: https://github.com/instagibbs/bitcoin/tree/202207-anyprevout (though this has non-segwit version of ephemeral anchors)
4) or maybe https://github.com/bitcoin-inquisition/bitcoin/releases/tag/v25.1-inq
5) BOLT draft for ln-symmetry(with segwit ephemeral anchors): https://github.com/instagibbs/bolts/tree/eltoo_draft
6) libwally work to support taprooty CLN changes(chunks of this was upstreamed but not all): https://github.com/instagibbs/libwally-core/tree/taproot
7) Notes on PTLCs, where eltoo-style channels get trivial fast-forwards still(aka APO-style fork fixes this): https://gist.github.com/instagibbs/1d02d0251640c250ceea1c66665ec163#single-sig-adaptor-ln-symmetry
8) Backporting anti-pin tech to today's BOLTs with proposed Bitcoin Core policy changes only: https://github.com/instagibbs/bolts/commits/zero_fee_commitment

# Current Status
On hold while I focus on the mempool side of things. I hope this work helps derisk conversations about future softforks and channel designs.

-------------------------

ProofOfKeags | 2024-01-05 18:09:14 UTC | #2

Incredible work and post. This kind of content makes me glad I'm subscribed to emails.

[quote="instagibbs, post:1, topic:359"]
eltoo-style channels have even longer than expect htlc expiry deltas to stay secure. No one had actually worked through the state machine before.
[/quote]

Can you elaborate on precisely why this is the case. I'm not sure I intuit why this is necessary. In theory the injunction period in the existing protocol is two rounds max: breach and justice. In eltoo, my understanding is that it is still two rounds: breach and fast-forward.

[quote="instagibbs, post:1, topic:359"]
CTV(emulation) ended up being useful! It removed the necessity of round-trips from the payment protocol, allowing for “fast forwards” that are extremely simple, and would likely reduce payment times if widely adopted.
[/quote]

I'm thoroughly unsurprised by this as CTV/Covenant schemes can, in principle, give you state-graph cut-through on any state machine encoded into Bitcoin transactions. However, I do have a question as to whether alternative covenant schemes also give you this property? I suspect but am not certain that they can.

-------------------------

instagibbs | 2024-01-05 18:41:17 UTC | #3

[quote="ProofOfKeags, post:2, topic:359"]
Can you elaborate on precisely why this is the case. I’m not sure I intuit why this is necessary. In theory the injunction period in the existing protocol is two rounds max: breach and justice. In eltoo, my understanding is that it is still two rounds: breach and fast-forward.
[/quote]

Fairly simple to explain and obvious in retrospect:
1) Alice and Bob do some channel ops
2) Alice then sends `update_signed` which is a MuSig2 partial sig
3) Bob smirks now that he's the only one with the full signature, goes silent
4) Alice goes to chain with prior state, waits however long the "update phase" has a timelock for the "settle transaction" to be valid
5) Right before timelock expires, Bob publishes the final state
6) wait for this timelock to expire... publish settle tx
7) finally can settle HTLCs

that's 2x the update phase timelock, not 1x. I know there's some "sandwiching" you can do to compress it a bit(conversations between LN engineers I didn't understand!), but it's north of my once expected 1x.

This game also shows up in the "layered eltoo" design from https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-January/002448.html which was supposed to get rid of the additional delta values altogether.

[quote="ProofOfKeags, post:2, topic:359"]
I’m thoroughly unsurprised by this as CTV/Covenant schemes can, in principle, give you state-graph cut-through on any state machine encoded into Bitcoin transactions. However, I do have a question as to whether alternative covenant schemes also give you this property? I suspect but am not certain that they can.
[/quote]

OP_CTV is probably the ideal here? I never validated it end to end or carefully looked though. I emulated it for a few more vbytes using APO.

-------------------------

instagibbs | 2024-01-05 19:18:51 UTC | #4

Ah, forgot one more issue left:

In spec/implementation, we only pass around a single nonce per round. This is correct *unless* you use the "optimistic" updates from "simplified update" protocol proposed originally by Rusty. In other words, you can never send updates out of turn, with the associated latency. You have to prod your counter-party to give up their turn first and wait for them to respond. To fix the out of turn optimistic updates, you would need another nonce and associated logic to manage it safely.

I never implemented the optimistic sending of updates, but didn't ban it outright and it's still in the spec: https://github.com/instagibbs/bolts/blob/a17b60f42077a785c625430e8f6e8e2828d4d898/XX-eltoo-peer-protocol.md#eltoo-simplified-operation

-------------------------

