# V3 transaction policy for anti-pinning

AntoineP | 2024-01-02 09:36:47 UTC | #1

"V3" is an opt-in constrained transaction relay policy proposed by Gloria Zhao to mitigate pinning. It is often discussed in conjunction with Greg Sanders' "[ephemeral anchors](https://github.com/instagibbs/bips/blob/7d79c5692bb745bf158f2d8f8e4979d80ad07e58/bip-ephemeralanchors.mediawiki)". With the upcoming phasing out of the mailing list, i'm opening this thread to discuss v3 at a high level.

Ressources:
- Original mailing list thread: https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-September/020937.html
- Original gist with discussions which led to the above mailing list post: https://gist.github.com/glozow/25d9662c52453bd08b4b4b1d3783b9ff
- Pull request to Bitcoin Core implementing the "version 3" policy: https://github.com/bitcoin/bitcoin/pull/28948. This PR contains the v3 [design specifications](https://github.com/glozow/bitcoin/blob/1dd62c3df4856c36bfc610f700684852772dd9f7/doc/policy/version3_transactions.md).
- Where it fits in the bigger package relay picture: https://github.com/bitcoin/bitcoin/issues/27463

-------------------------

AntoineP | 2024-01-02 09:42:08 UTC | #2

For context Peter Todd recently wrote a [blog post](https://petertodd.org/2023/v3-transactions-review) with the following conclusion:
> V3 transactions should not be be shipped at the moment. It has no compelling use-cases right now without ephemeral outputs, which should be discouraged, and in its current form is still vulnerable to transaction pinning.

Discussion about it [is happening on the Bitcoin Core PR](https://github.com/bitcoin/bitcoin/pull/28948#issuecomment-1873490509), and i think this thread is a more appropriate forum to discuss the merits of the v3 proposal at a high level.

-------------------------

glozow | 2024-01-02 11:44:31 UTC | #3

Thanks, I've linked this in the PR description

-------------------------

instagibbs | 2024-01-02 20:46:40 UTC | #4

moving discussion here...

> Again, I am not denying that there is a 100x reduction in the hypothetical situation of transaction pinning. What I am pointing out is that Lightning already fixed that transaction pinning problem with the design of anchor channels, making your 100x reduction irrelevant.

If an adversary splits the view of which commitment transaction is broadcasted, the remote copy can become the pin since the defender is unable to propagate their spend of the remote commit tx anchor(as their peers do not have the remote commit tx)

> But *even that* is actually incorrect: the highest feerate you could ever possibly need is the one where all of your balance in the channel goes to fees.

I don't think that's correct. As a simplified motivating example, if we set reserve amounts to 0, if you want to send "all" your funds, your top feerate will be zero, even if the total funds recoverable in timeout would be all your remaining liquidity.

I'm sure there are ways to mitigate this, like capping total HTLC exposure as a % of liquidity, but it's pretty clearly a reduction in functionality? Possibly completely broken as it implies that you can never drain your liquidity completely to reserve levels?

-------------------------

harding | 2024-01-03 21:10:57 UTC | #5

In his [post](https://petertodd.org/2023/v3-transactions-review), Peter Todd says:

> For example, if you wanted to cover 10satvB to 1000satvB, with 10% increments, you’d only need 50 fee variants, for a total size of 3200 bytes, taking just 5ms to sign (single core). Given the very good scalability of Lightning, this overhead is reasonable.

I think this ignores a possible effect of [stuckless payments](https://bitcoinops.org/en/topics/redundant-overpayments/) on the LN network.  If Alice wants to pay Zed 100,000 sats using a refundable overpayment of 20 parts each at 10,000 sats, Zed is likely to claim the first 10 parts that arrive and decline the remaining 10 parts.  The nodes that forwarded the claimed parts will receive forwarding fees; the other nodes will not be compensated.

Nodes that forward payments faster will, when all other things are equal, be more likely to receive payment than nodes that forward payments slower.  In a regime with stuckless payments, I think 5 ms (plus the extra time to transmit several KB of musig2 partial signatures) is large enough that it may result in a significant reduction in forwarding node revenue compared to forwarding nodes that can send just a single partial signature.  That would push forwarding nodes towards a lower-latency single commitment scheme, such as those based on CPFP.

-------------------------

harding | 2024-01-05 20:23:25 UTC | #6

In this post, I will attempt to show that Peter Todd's recent [post][pt post] significantly underestimates the time to sign multiple commitment transaction fee variants.  I then directly address the concern about "the danger to decentralization" of building protocols on top of CPFP; I note that a simple as-need soft fork and small change to ephemeral anchor rules makes them into a powerful tool for _protecting_ decentralization in the face of CPFP-based protocols.

## Different parents have different children

To fully spend an HTLC in LN-Penalty, you need to publish three transactions:

1. The commitment transaction.  Money is allocated to an output of the commitment transaction using an HTLC script.

2. An HTLC-Success or HTLC-Timeout transaction.  These implement the classic logic of HTLCs: Alice can claim money at any time with a preimage (HTLC-Success) or Bob can receive a refund after a timeout (HTLC-timeout).

3. HTLC-Penalty or delayed spend.  If either the HTLC-Success or HTLC-Timeout transaction were created in a revoked commitment transaction, these allow the party who didn't publish the revoked state to claim the full amount of the HTLC.  If the current commitment transaction was used, this is delayed by a timelock (to allow an HTLC-Penalty to be broadcast) but can otherwise be a normal spend (although with a large witness).

Bitcoin currently lacks [covenants][], so the only way to ensure an HTLC-Success or HTLC-Timeout transaction pays an output with the HTLC-Penalty conditions is for the HTLC-Success and HTLC-Timeout transactions to require signatures from both parties, allowing each party to only sign payments to the HTLC-Penalty conditions.  To ensure each party can act unilaterally (a requirement for trustlessness), each party must presign their half of the spend and send it to the other party before that party accepts (relies upon) an HTLC in the commitment transaction.

This means that updating an LN-Penalty commitment transaction requires sending 1 signature for the commitment transaction and 1 signature for every HTLC output of that commitment transaction.  Each party can add a maximum of 483 outputs (source: BOLT2) to a commitment transaction, for a total of `2*483=966` outputs, which is also 966 signatures.

In the above-linked post, Peter Todd suggests presigning multiple versions of the commitment transaction at different feerates.  He also notes that HTLC-Success and HTLC-Timeout transactions would need to be signed at different feerates if the same technique was applied to them, resulting in N^2 signatures, although he suggests mitigations.

Not mentioned in that post is the effect of the HTLC-Success and HTLC-Timeout transactions being children of the commitment transaction.  Each different version of the commitment transaction at a different feerate has a different txid.  An HTLC-Success or HTLC-Timeout transaction will only be valid if a particular txid gets confirmed, so for every different version of the commitment transaction, a different version of the HTLC-Success and HTLC-Timeout transaction is needed.  This requires N+N*M signatures, where N is the number of commitment transaction variants and M is the number of HTLCs.

The post suggests N=50 and a points to a Jonas Nick benchmark of 100us per signature (with non-controversial LN upgrades applied). BOLT2 suggests a maximum M=966, for a worst-case of 48,350 signatures that take a bit under 5 seconds to generate.  Even a single hop adding a five second delay would be a significant degradation from the performance we hope for in the _Lightning_ Network.

Using a more likely average M=200 and assuming 10 hops would give a minimum payment forwarding time of 12.5 seconds.

This does not consider other negative consequences of massively increasing signature operations, such as the increase in bandwidth and receiver-side verification operations.

If I haven't messed up my analysis, I think this implies presigned incremental fee bumping does not provide an acceptable substitute to CPFP fee bumping of LN commitment transactions in all reasonable cases.  If CPFP of commitment transactions is to continue to be used, we should encourage P2P protocol developers to improve support for it, e.g.  through package relay, package RBF, v3 relay, and ephemeral anchors.

## Ephemeral anchors can protect decentralization from CPFP-based protocols

In the post, Peter Todd argues:

> if you have an anchor-using transaction that you need to get mined, it
> costs twice as much blockspace to get the transaction mined via the
> intended mechanism — the anchor output — as it does by getting a miner
> to include the transaction without the anchor output spend. [...] a
> large miner could easily offer out-of-band fee payments at, say, a 25%
> discount, giving them a substantial 25% premium over their smaller
> competitors.

I find this to be a compelling argument about the dangers of building protocols that depend on CPFP fee bumping.  However, ephemeral anchors are different than other forms of CPFP fee bumping in that we can easily turn their policy rules into consensus rules, plus make one small change to eliminates the advantage of out-of-band payments.

The _policy_ rules for ephemeral anchors are basically:

- If a transaction pays a certain specified scriptPubKey (the parent's ephemeral anchor)

- It will only be considered policy-valid if it is packaged with a spend of that output (the child)

Miners following that policy will only include transactions with ephemeral anchor outputs if the same block includes the spend of that output.  As described in the post, that's just a policy rule and any miner can choose not to include the spend.

However, it's easy to turn those rules into consensus rules: a block may only include a parent ephemeral anchor if it also includes the child spend.  We could also add an extra rule in the soft fork: the child spend must include at least two inputs.  That would mean the amount of block space used by someone paying miners out-of-band would be equal to the (best and expected normal case) of someone paying any miner in a decentralized fashion.  In other words, the incentive for paying out of band would be eliminated.

Whether as policy or consensus, ephemeral anchors are totally opt-in, don't affect any other part of the Bitcoin protocol, don't add significant resource costs, and don't complex code or new primitives.  Implementing them as a soft fork is almost as easy a soft fork as could be.  (Forbiding 64 byte transactions is easier, but we have no excuse for not having done that yet IMO.)

If we start with policy-only ephemeral anchors, then any protocol that depended on the policy version of them would continue working without changes if a soft fork activated.  Only people who expected to pay fees out of band would be affected, and they would simply have to use the CPFP method already implemented in stock software.

This leads me personally to the opposite conclusion of that section of the original post.  The post says,

> We must not build protocols that put decentralized mining at a
> disadvantage. On this basis alone, there is a good argument that V3
> transactions and ephemeral anchor outputs should not be implemented.

If we expect people to continue to build protocols based on CPFP fee bumping, and I think there's a compelling case for that in the previous section, then ephemeral anchors is the **best** way I know of to prevent a CPFP-based reduction in mining decentralization.  We should start with policy-only ephemeral anchors and, if it gains traction and we don't discover anything better, eventually switch to consensus-enforced ephemeral anchors.

[pt post]: https://petertodd.org/2023/v3-transactions-review
[covenants]: https://bitcoinops.org/en/topics/covenants/

-------------------------

orangesurf | 2024-01-05 20:40:59 UTC | #7

[quote="harding, post:6, topic:340"]
it’s easy to turn those rules into consensus rules: a block may only include a parent ephemeral anchor if it also includes the child spend. We could also add an extra rule in the soft fork: the child spend must include at least two inputs. That would mean the amount of block space used by someone paying miners out-of-band would be equal to the (best and expected normal case) of someone paying any miner in a decentralized fashion. In other words, the incentive for paying out of band would be eliminated.
[/quote]

Is it a given that there would be widespread consensus for such a soft fork?

-------------------------

harding | 2024-01-05 20:49:20 UTC | #8

[quote="orangesurf, post:7, topic:340"]
Is it a given that there would be widespread consensus for such a soft fork?
[/quote]

If enough people are paying fees out-of-band for ephemeral anchors that we believe it's threatening mining decentralization, our choice would be to either activate an easy and laser-focused soft fork or to lose mining decentralization.  Given that choice, I think there would be widespread consensus.

-------------------------

instagibbs | 2024-01-05 20:50:02 UTC | #9

[quote="harding, post:6, topic:340"]
the child spend must include at least two inputs. That would mean the amount of block space used by someone paying miners out-of-band would be equal to the (best and expected normal case) of someone paying any miner in a decentralized fashion.
[/quote]

hmmm not sure about this extension, sometimes the best thing to do is just burn the ephemeral anchor value by itself, which results in a 65 vbyte txn. I think in the end it's the same risk as people paying transaction accelerators so they can use fewer inputs in an RBF?

-------------------------

