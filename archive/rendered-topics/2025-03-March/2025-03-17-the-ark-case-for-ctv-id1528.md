# The Ark case for CTV

stevenroose | 2025-03-17 17:23:04 UTC | #1

I initially made this layout as part of [another thread](https://delvingbitcoin.org/t/ctv-csfs-can-we-reach-consensus-on-a-first-step-towards-covenants/1509/50?u=stevenroose), but to avoid derailing the original conversation, I'll repeat the post here so that questions and suggestions can be posted here.


Because it was requested, let me make the [Ark case for CTV](https://x.com/stevenroose3/status/1865144252751028733) in a bit more detail, because I think it is significant. One can also refer to [ark-protocol.org](https://ark-protocol.org/intro/vtxos/index.html) for more detail.

TL;DR: send-to-others, automatic vtxo reissuance, lightning receive and mass payouts

For those that don't know, in Ark users hold off-chain utxos that we call virtual utxos or vtxos. In the original design, which uses CTV, these vtxos are constructed using a tree of transactions where each transaction uses a CTV covenant to get into the next transaction. Much like congestion control.

Generalized, a vtxo can be any construction that has:

1. a chain or one or more txs with policy `ctv OR (pk(S) AND after(T))` (`S` being the Ark server's key and `T` being the vtxo's expiry time). Each `ctv` comitting to the next tx in the chain

2. followed by a tx with the following policy `(pk(A) AND pk(S)) OR (pk(A) AND older(delta))` (`A` being the key of the vtxo's owner and `delta` being a relative timelock which gives the other policy enough time to react (something like 24h or 28h).

Using CTV, this construction can be constructed entirely non-interactively, meaning that knowing the Ark's parameters `S` and `delta`, anyone can issue their own vtxo that expires at time `T`. This is how "onboarding" works, i.e. entering the Ark.

When we developing `clArk` (covenant-less Ark), we replace the `ctv` policy with a MuSig2 pre-signature of `S` and all the users of the leaves below that node. The intermediate nodes from step 1. above become then `pk(S+A+B+C+..) OR (pk(S) AND after(T))`. Apart from additional signing and storage overhead, this has one strong limitation:

**Co-signed (clArk) vtxos cannot be issued without the presence of the eventual owner**, otherwise the construction is not safe. This has led our team (at [Second](second.tech)) to only designate Ark rounds to be used for users that have expiring vtxos and need them to be refreshed, in effect "sending them to themselves". Because senders always have to be present in either Ark construction (they need to sign [forfeit txs](https://ark-protocol.org/intro/connectors/index.html#forfeit-transactions)), if the receivers are also the senders, they are already present during the process.

But being able to issue vtxos for others has several significant benefits:

- The most obvious is simply being able to **send vtxos to others without requiring the receiver to participate in Ark rounds**. We currently only support users sending to each other using [out-of-round (Arkoor)](https://ark-protocol.org/intro/oor/index.html) transactions, which has an additional security assumption. With CTV we could again support sending vtxos to others in rounds.

- It allows the server to issue vtxos for users. We currently want to do this in two occasions:
  - A good server will want to **re-issue expired vtxos automatically**. This is obviously not secure, but the server can continuously provide proofs that it hasn't claimed any expired vtxos and for some smaller-value vtxos, this can be enough security for certain users.
  - **Receiving Lightning payments** in Ark (without having virtual channels) is not easy. One way it could be done is by having the user notify the server that it is expecting an inbound payment, the server will accept the HTLC as a hodl HTLC and issue an HTLC for the user in a vtxo in the next round. The user can then reveal the preimage which grants him the vtxo fully and allows the server to claim the hodl HTLC. (With various anti-abuse measures in place.) Without CTV the receiver would have to participate in a round in order to receive his HTLC.

- It would allow non-interactive onboards, but since for an onboard, the onboarder is usually the receiver anyway this seems quite uninteresting, but it equivalently **allows any party to non-interactively issue any number of vtxos with a single on-chain output**. We think this can be a very powerful tool for exchanges or DCA providers that want to regularly payout amounts to their users. Using CTV, they could independently construct a tree of vtxos (using the Ark's parameters), fund the root tx and inform all their users. Since these vtxos are no different than vtxos created in Ark rounds, the recipients can use them as if they were any other type of vtxo in this Ark.

At Second we believe these benefits are quite significant for both the user experience more broadly and the viability of participating in Ark rounds a the mobile setting in the first place. We have little data currently to back up this latter claim, but we are working hard to put up our implementation of co-signed Ark to the test on mobile clients.

-------------------------

instagibbs | 2025-03-17 17:26:03 UTC | #2

[quote="stevenroose, post:1, topic:1528"]
We currently only support users sending to each other using [out-of-round (Arkoor)](https://ark-protocol.org/intro/oor/index.html) transactions, which has an additional security assumption. With CTV we could again support sending vtxos to others in rounds.
[/quote]

Can you go into that security model a bit, with and without CTV?

I wasn't under the expectation that Arkoor would be statecoins forever, but "routinely" settled via a real swap to a new Ark (which gets block confirmation and resets the coin)

-------------------------

