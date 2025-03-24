# The Ark case for CTV

stevenroose | 2025-03-22 19:14:08 UTC | #1

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
  - **Receiving Lightning payments** in Ark (without having virtual channels) is not easy. One way it could be done is by having the user notify the server that it is expecting an inbound payment, the server will accept the HTLC as a hodl HTLC and issue an HTLC for the user in a vtxo in the next round. The user can then reveal the preimage which grants him the vtxo fully and allows the server to claim the hodl HTLC. (With various anti-abuse measures in place.) Without CTV the receiver would have to participate in a round in order to receive his HTLC, but **receivers participating in rounds is vulnerable to DoS attacks**, so this is not possible. It would mean that only receivers with already existing VTXOs could receive LN VTXOs and they would have to proof ownership of an existing VTXO so that we can slash their VTXO if they don't provide their signatures in the round.

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

neha | 2025-03-21 16:49:31 UTC | #3

I want to make sure I'm understanding this correctly, because if so I find it really interesting and striking:

Requiring interactivity (the receiver to be online) was such a strong requirement from a UX perspective, that instead you chose to introduce the (time-limited) security assumption of relying on the server and sender not to collude.

I think this is a really important data point for understanding the tradeoffs between interactivity and trust!

-------------------------

stevenroose | 2025-03-22 19:02:45 UTC | #4

Exactly, so arkoor vtxo have what we call the "statechain model", but maybe we should find a better name for it, like "server cosign assumption" or something.

The assumption is that it can't be double spent as long as the server doesn't collude with a previous owner of the VTXO (meaning anyone since its inception in a round). A user can "exit" this reduced trust model by refreshing their VTXO during a round. A VTXO that is the direct result of a round does not have such reduced trust assumption.

This means that in the worst case, any VTXO in such an arkoor model will be refreshed back into "trustless mode" every interval of the expiry time (1 or 2 months). Users who don't feel comfortable holding an arkoor VTXO can always submit their VTXOs for refresh directly after receiving them. But since the liquidity fee that has to be paid for refreshes is proportional with the time until the expiry of the VTXO, the fee will be higher when a VTXO is refreshed right after receipt, compared to a lower fee when the receiver holds on to the arkoor VTXO until it almost expires.

-------------------------

stevenroose | 2025-03-22 19:12:19 UTC | #5

Yes. But only partially yes. There are other benefits from the arkoor model. Like I mention in my previous reply, the liquidity cost for the server (and hence the liquidity fee paid by the user) to refresh a VTXO is proportional with the time remaining until the expiration of the VTXO. Hence, the arkoor model gives the receiver of a VTXO the choice which side of the trade-off he wants to take. 

If he wants full trustless ownership of the VTXO, but pay a higher liquidity fee, he can refresh the VTXO. If he is ok with the arkoor trust assumption for another 15-30 days (on average half the expiration time), he can save on fees.

We imagine that people that are holding significant amounts in their VTXOs, and receive them from unknown third parties, will be more inclined to refresh their VTXOs early. While users that are doing smaller day-to-day payments will probably prefer to save on fees.

But to reply to your root question: yes, the interactivity requirement of the round is pretty strong. Moreso, it is also **very vulnerable to denial-of-service attacks**. The round can only finish if all the participating users do their part correctly, so f.e. it is possible for an attacker to participate in the round with multiple "users", and each attempt fail a single one, hence forcing the entire round to be retried as many times as he has identities.

The way we mitigate these DoS problems is first of all banning non-cooperative users from the next attempt, this is an obvious step. Every time a round needs to be re-attempted, only the participants that provided their signatures are allowed in the next attempt.

But second, we also penalize the VTXOs of the participants. And here there is a very big difference between a sender and a receiver. A sender submits a VTXO as input to the round. If he abandons the round, we can slash his VTXO a little bit (basically reducing it's value in our books by a few sats). This means that each time he abandons a round, he loses some money. (They can always unilaterally exit to get the full amount but exits also have an on-chain cost.)

Receivers, on the other hand, are not associated with an existing VTXO. So they can't be penalized and have nothing to lose in performing a DoS attack on the round. Therefore, even if the interactivity would be minimal, we can't actually have receivers participate in rounds. (That's why LN receives are difficult without CTV, a receiver would only be able to receive a LN payment if they would already have an existing VTXO they can sign overship for and we'd slash that one.)

-------------------------

instagibbs | 2025-03-24 14:32:20 UTC | #6

[quote="stevenroose, post:4, topic:1528"]
A user can “exit” this reduced trust model by refreshing their VTXO during a round. A VTXO that is the direct result of a round does not have such reduced trust assumption.
[/quote]

Ah I see, users can "send to self" aka refresh, but not send to others. I can see why that is possibly attractive; you only have to get self-sending right for the client :slight_smile:

-------------------------

