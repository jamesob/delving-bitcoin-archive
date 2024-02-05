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

harding | 2024-01-05 23:17:45 UTC | #6

In this post, I will attempt to show that Peter Todd's recent [post][pt post] significantly underestimates the time to sign multiple commitment transaction fee variants.  I then directly address the concern about "the danger to decentralization" of building protocols on top of CPFP; I note that a simple as-need soft fork and small change to ephemeral anchor rules makes them into a powerful tool for _protecting_ decentralization in the face of CPFP-based protocols.

## Different parents have different children

To fully spend an HTLC in LN-Penalty, you need to publish three transactions:

1. The commitment transaction.  Money is allocated to an output of the commitment transaction using an HTLC script.

2. An HTLC-Success or HTLC-Timeout transaction.  These implement the classic logic of HTLCs: Alice can claim money at any time with a preimage (HTLC-Success) or Bob can receive a refund after a timeout (HTLC-timeout).

3. HTLC-Penalty or delayed spend.  If either the HTLC-Success or HTLC-Timeout transaction were created in a revoked commitment transaction, these allow the party who didn't publish the revoked state to claim the full amount of the HTLC.  If the current commitment transaction was used, this is delayed by a timelock (to allow an HTLC-Penalty to be broadcast) but can otherwise be a normal spend (although with a large witness).

Bitcoin currently lacks [covenants][], so the only way to ensure an HTLC-Success or HTLC-Timeout transaction pays an output with the HTLC-Penalty conditions is for the HTLC-Success and HTLC-Timeout transactions to require signatures from both parties, allowing each party to only sign payments to the HTLC-Penalty conditions.  To ensure each party can act unilaterally (a requirement for trustlessness), each party must presign their half of the spend and send it to the other party before that party accepts (relies upon) an HTLC in the commitment transaction.

This means that updating an LN-Penalty commitment transaction requires sending 1 signature for the commitment transaction and 1 signature for every HTLC output of that commitment transaction.  Each party can add a maximum of 483 outputs (source: BOLT2) to a commitment transaction, for a total of `2*483=966` outputs, which is also 966 signatures.

In the above-linked post, Peter Todd suggests presigning multiple versions of the commitment transaction at different feerates.  He also notes that HTLC-Success and HTLC-Timeout transactions would need to be signed at different feerates if the same technique was applied to them, resulting in N^2 signatures, although he suggests mitigations.

Not ~~mentioned~~ detailed in that post is the effect of the HTLC-Success and HTLC-Timeout transactions being children of the commitment transaction.  Each different version of the commitment transaction at a different feerate has a different txid.  An HTLC-Success or HTLC-Timeout transaction will only be valid if a particular txid gets confirmed, so for every different version of the commitment transaction, a different version of the HTLC-Success and HTLC-Timeout transaction is needed.  This requires N+N*M signatures, where N is the number of commitment transaction variants and M is the number of HTLCs.

The post suggests N=50 and a points to a Jonas Nick benchmark of 100us per signature (with non-controversial LN upgrades applied). BOLT2 suggests a maximum M=966, for a worst-case of 48,350 signatures that take a bit under 5 seconds to generate.  Even a single hop adding a five second delay would be a significant degradation from the performance we hope for in the _Lightning_ Network.

Using a more likely average M=200 and assuming 10 hops would give a minimum payment forwarding time of 12.5 seconds.

This does not consider other negative consequences of massively increasing signature operations, such as the increase in bandwidth and receiver-side verification operations.

If I haven't messed up my analysis, I think this implies presigned incremental fee bumping does not provide an acceptable substitute to CPFP fee bumping of LN commitment transactions in all reasonable cases.  If CPFP of commitment transactions is to continue to be used, we should encourage P2P protocol developers to improve support for it, e.g.  through package relay, package RBF, v3 relay, and ephemeral anchors.

## Ephemeral anchors can protect decentralization from CPFP-based protocols

**Edit: the points in this section are undermined by later replies in the thread.  I still think there's something conceptually useful here, but this solution is broken for now.  You probably want to skip reading this.**

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

Edits:

- 2023-01-05 13:17 HST.  Added note about soft fork ephemeral anchors not being a satisfactory solution after additional feedback.  Struck out comment that PT hadn't mentioned needing to sign N variants per HTLC; his post says, "we will have to sign N HTLC variants rather than a single variant".

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

harding | 2024-01-05 20:59:48 UTC | #10

[quote="instagibbs, post:9, topic:340"]
sometimes the best thing to do is just burn the ephemeral anchor value by itself, which results in a 65 vbyte txn. I think in the end it’s the same risk as people paying transaction accelerators so they can use fewer inputs in an RBF?
[/quote]

I think the expected use of an ephemeral anchor would be:

- Parent: 1 input, x+1 outputs.  The +1 is the ephemeral anchor.
- Child: >=2 inputs, >=1 outputs.  One input is the anchor spend; the other contributes fees.

In the pathological case, the parent is the same but the child is never created.  So we need a requirement that spending the parent can only be done at a cost equal to at least 2-inputs and 1 output.

That requirement doesn't need to be an input requirement; it could also be treating a childless parent as if it had the additional weight of a 2-input, 1-output child.

-------------------------

nettimel | 2024-01-05 21:20:21 UTC | #11

[quote="harding, post:6, topic:340"]
However, it’s easy to turn those rules into consensus rules: a block may only include a parent ephemeral anchor if it also includes the child spend. We could also add an extra rule in the soft fork: the child spend must include at least two inputs. That would mean the amount of block space used by someone paying miners out-of-band would be equal to the (best and expected normal case) of someone paying any miner in a decentralized fashion. In other words, the incentive for paying out of band would be eliminated.
[/quote]

But as per [Peter Todd's tweet](https://twitter.com/peterktodd/status/1743376014154121374) miners can just efficiently spend all the anchor channel outputs together in a block. More efficient than any normal CPFP. It's still more efficient to pay out of band too because UTXOs for the ephemeral txs need to be created and spent. Vast majority of channels will be stuff like Phoenix in the future, which want one UTXO per user. Can't anchor that.

-------------------------

instagibbs | 2024-01-05 21:22:26 UTC | #12

Backing up a bit:

How is this decentralization argument not generalizable to any BringYourOwnFunds(BYOF) scheme?

For example, presigned HTLC-X transactions?

These are batchable. They are cheaper to be paid out of band.

Are we really arguing against all exogenous fees in smart contracts? I reject the premise entirely.

-------------------------

nettimel | 2024-01-05 21:32:42 UTC | #13

[quote="instagibbs, post:12, topic:340"]
For example, presigned HTLC-X transactions?
[/quote]

Link to what a HTLC-X transaction is?

-------------------------

harding | 2024-01-05 21:59:28 UTC | #14

[quote="nettimel, post:13, topic:340"]
Link to what a HTLC-X transaction is?
[/quote]

@instagibbs is talking about HTLC-Success and HTLC-Timeout transactions as defined in [BOLT3](https://github.com/lightning/bolts/blob/8a64c6a1cef979b3f0cecb00ba7a48c2d28b3588/03-transactions.md)

These are presigned spends of HTLCs made with `SIGHASH_SINGLE|SIGHASH_ANYPREVOUT` that allow multiple of them to be combined into a single transaction.  The don't contribute any fee (or any meaningful fee), so we would normally expect them to be combined with an extra contributed input that adds fees (and likely an extra output for change).  However, the threat-to-decentralization argument, you could pay a miner out-of-band fees to mine them without adding an extra input or output, saving block space.

I made this same argument to PT last night (last paragraph): https://twitter.com/hrdng/status/1743293516992876561

-------------------------

instagibbs | 2024-01-05 22:00:07 UTC | #15

[quote="harding, post:14, topic:340"]
I made this same argument to PT last night (last paragraph): https://twitter.com/hrdng/status/1743293516992876561
[/quote]

He almost doesn't even address it.

"As for SIGHASH_ANYONECANPAY, obviously, it is still a problem, which is why I mentioned, among other mitigations, that in the future SIGHASH_ANYPREVOUT would help the situation."

Again, does he believe that all future smart contracts should have endogenous fees, otherwise we're risking decentralization? Is that really a tenable reason to stop any proposal(aside from the ~50vb that ephemeral anchors costs)?

-------------------------

nettimel | 2024-01-05 22:42:16 UTC | #16

[quote="instagibbs, post:15, topic:340"]
Again, does he believe that all future smart contracts should have endogenous fees, otherwise we’re risking decentralization?
[/quote]

["Obviously scale matters. We can't prevent every single circumstance where someone might want to pay a transaction out of band. But we can and should design protocols to minimize it."](https://twitter.com/peterktodd/status/1743397111268274600)

That seems to cover it. His article that started all this proposes a few solutions for HTLC-X problem too: https://petertodd.org/2023/v3-transactions-review#htlcs-and-replace-by-fee

-------------------------

harding | 2024-01-05 23:11:33 UTC | #17

[quote="nettimel, post:11, topic:340"]
miners can just efficiently spend all the anchor channel outputs together in a block. More efficient than any normal CPFP.
[/quote]

That's a fair point; it does undermine my claimed advantage of a soft-fork version of ephemeral anchors.

I haven't thought of an alternative way to obtain the property of in-band CPFP fee bumping pay the same weight cost as out-of-band transaction acceleration, at least not without preventing batching that would be beneficial to users.

-------------------------

instagibbs | 2024-01-06 13:53:10 UTC | #18

[quote="nettimel, post:16, topic:340"]
That seems to cover it.
[/quote]

That's an argument for *specs*  to avoid exogenous fees when possible, not how to make sensible mempool designs. *if* your smart contract can have endogenous fees, you should consider it. That's appropriate! But this is not the case in *many*(most?) protocols outside of ln-penalty, and it's completely inappropriate to gatekeep good mempool design to a specific instantiation of a specific idea.

Please see motivation/use-cases sections for more details on what kind of smart contracts V3 and ephemeral anchors is useful for: 

https://github.com/instagibbs/bips/blob/527b007dbf5b9a89895017030183370e05468ae6/bip-ephemeralanchors.mediawiki#motivation

Lastly, if "scale matters", then we should be doing everything in our power to make fee bidding useful. No one has put forward an alternative proposal that makes sense for many RBF like V3. This has been years of discussions, and based on Peter's writeup I don't think he bothered to read my BIP draft.

[quote="nettimel, post:16, topic:340"]
His article that started all this proposes a few solutions for HTLC-X problem too:
[/quote]

This stuff IIUC is only talking about signature complexity and providing fees? As a humerous aside, HTLCs are endogenous fees(decentralization hit!), and would be more pin resistant using ephemeral anchors,  but I'm not pushing for that in my proposal spec because of relative spec diff(HTLC-Success paths on both commit txs would have to be pre-signed, really hairy given the duplex updates it has now) :slight_smile: 

[quote="harding, post:17, topic:340"]
That’s a fair point; it does undermine my claimed advantage of a soft-fork version of ephemeral anchors.
[/quote]

Note that similar OOB benefits are obtained by batching and `SIGHASH_SINGLE|ACP`-based smart contracts today, and would be similar if we had any good introspection opcodes.

-------------------------

moonsettler | 2024-01-07 18:16:21 UTC | #19

is there a reason to necessarily allow more than 1 unconfirmed ancestor for an ephemeral anchor type fee bump transaction?

if only the anchor itself is unconfirmed, what mempool pinning tactics are possible?

-------------------------

instagibbs | 2024-01-07 20:27:40 UTC | #20

[quote="moonsettler, post:19, topic:340"]
is there a reason to necessarily allow more than 1 unconfirmed ancestor for an ephemeral anchor type fee bump transaction?
[/quote]

I mean, it'd be nice for batched CPFP! There are no known designs to do that that are pin-resistant though. 

[quote="moonsettler, post:19, topic:340"]
if only the anchor itself is unconfirmed, what mempool pinning tactics are possible?
[/quote]
not entirely sure what the question is, mind rewording?

-------------------------

moonsettler | 2024-01-07 21:00:40 UTC | #21

[quote="instagibbs, post:20, topic:340"]
I mean, it’d be nice for batched CPFP! There are no known designs to do that that are pin-resistant though.
[/quote]

that's what i thought. so first we could just allow for a more restricted and less efficient way of doing ephemeral anchors. but at least we would have something usable that is easy to implement and reason about. then expand later to more useful and more optimal ways of doing it?

-------------------------

harding | 2024-01-08 00:46:00 UTC | #22

I'm trying to better understand the claimed [v3 pinning vulnerability](https://petertodd.org/2023/v3-txs-pinning-vulnerability).  Do I understand correctly that:

1. The worst case, which is what PT analyzed, is a commitment transaction with no pending HTLCs.
2. The attacker reduces the feerate of the package/cluster containing the commitment transaction by the attacker using their own money to pay the fees.
3. A commitment transaction with no pending HTLCs has no special urgency (i.e., no timelocks that will expire to the detriment of the broadcasting party).

In other words, the worst case form of this attack would be that Bob will have to wait a bit longer to respend his channel funds but Mallory will pay his fees?

Obviously, the attack also works against commitment transactions with pending HTLCs, but for each additional HTLC output, the attack quickly becomes less effective due to the decrease in relative size difference.  Perhaps more interestingly, if Bob pays out of band to fee bump both the commitment transaction and Mallory's pin (ephemeral anchor spend), Bob will possibly pay less fees than he would've without the pinning attack and Mallory will end up losing the funds she spent on the attack.

Of course, paying to defeat an attack out of band is still bad for mining decentralization, but I think all of the above points to this pinning attack being possibly ineffective.

-------------------------

moonsettler | 2024-01-08 10:39:32 UTC | #23

yeah, protocol and wallet devs should be reluctant to rely on oob payments to resolve mempool conflicts or timeout situations. such an api is a single point of failure and a huge centralization vector.

not gonna pretend i fully understand the various pinning attacks, i expect the major pain would be when you try to settle some contract by adding fees through the anchor, and someone else outbids your tx with higher fees (which is not a problem for you in itself) but with dependence on other unconfirmed ancestors, which ancestors they can double spend, resulting in the eviction of both fee bumps and the anchored tx from the mempools.

-------------------------

ajtowns | 2024-01-08 10:53:07 UTC | #24

[quote="harding, post:6, topic:340"]
However, it’s easy to turn those rules into consensus rules: a block may only include a parent ephemeral anchor if it also includes the child spend.
[/quote]

I don't think that works: if you get to the point where miners offer an API for out of band fee payments that's trustworthy enough and discounted enough that it sees wide adoption, leading to a centralisation risk, then if you did have a soft fork to require ephemeral anchors be spent, then those miners could work around your soft fork as follows:

 * create a "nouveau ephemeral anchors" BIP, with the same behaviour as before the soft-fork, but with a different `scriptPubKey` pattern
 * flood the network with nodes that relay according to the new BIP, have those nodes preferentially peer with each other to ensure there aren't disconnected subgraphs
 * get this implemented by the devs that had already integrated with their API
 * push the patch to core noting that it's in wide use on the network
 * profit

I think the sort of soft-fork you be okay if it was in line with economic incentives (ie, the only time an ephemeral anchor is in a block but not immediately spent is people testing things on regtest/signet/testnet, or due to bugs), but if the economic incentives are strongly pushing the other way (miners, wallet devs and users all collaborating to save a buck despite the centralisation risk), I don't think a soft fork here would actually help.

(The other soft-fork approach would be: "an ephemeral anchor output can only be spent in the same block that it was created; it's removed from the utxo set once the block is completely processed". That resolves the "bugs lead to dust in the utxo set" issue, but doesn't touch out-of-band-payment incentives, and introduces the potential for the child tx to become invalid in a reorg, if for some reason it isn't included in the same block as its parent)

-------------------------

instagibbs | 2024-01-08 16:26:01 UTC | #25

[quote="moonsettler, post:23, topic:340"]
resulting in the eviction of both fee bumps and the anchored tx from the mempools.
[/quote]

I think you're mostly describing the "cycle attack", where the child is RBF'd out of the mempool, and the new child no longer spends the ephemeral anchor, causing the parent transaction to be evicted.

https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-October/021999.html

Mitigation is, in general, to aggressively re-broadcast, since each cycle costs the attacker fees(unlike pins).

You can think of it in another, imo more principled way: Someone is paying toe censor you, but they have to continuously bid block on block(and inbetween blocks if you're doing it right + mempool synchrony assumptions), where you only pay once at the end of the game at most, and at your expected rate.

I don't want to get too deep into cycle discussion though; it's been done to death.

-------------------------

rustynail | 2024-01-09 01:41:58 UTC | #26

[quote="harding, post:22, topic:340"]
In other words, the worst case form of this attack would be that Bob will have to wait a bit longer to respend his channel funds but Mallory will pay his fees?
[/quote]

Just checked. At the moment the fees required to get into the next block are 30sat/VB while the min relay fee is 22.2sat/VB. The mempool feerate is super flat. So an attacker could easily force you to wait basically forever becuase those min fee txs aren't gonna get mined, unless you pay the 50% extra. Not good. And the attacker's txs will get pushed out of the mempool as fees go up so Mallory isn't paying anything to attack.

HTLC outputs aren't much more space. I don't think Peter's numbers would change that much if you did the computation for that case too. You could do the same attack on the HTLC as well I think?

@glozow How are V3 transactions supposed to work with HTLCs anyway? Going to have V3 HTLC-success/failure transactions too?

-------------------------

rustynail | 2024-01-09 02:31:45 UTC | #27

Hmm, so Peter's thing made assumptions about the attackers fee margin of error that aren't valid right now. His analysis assumed the attacker paid 1/2.5th of the victim fees, which would be less than minrelayfee. You could probably get away with just 25% less right now, which would force the victim to pay 2.8x more fees. Big difference!

-------------------------

instagibbs | 2024-01-09 14:57:31 UTC | #28

The exact pin will indeed rely on exact mempool conditions, which is why I mostly just said "5x max" in my BIP text: https://github.com/instagibbs/bips/blob/527b007dbf5b9a89895017030183370e05468ae6/bip-ephemeralanchors.mediawiki#motivation

If a 5x theoretical pin, down from 500x, with practical limits depending on factors generally considered out of the attacker and defenders power. The only direct way to further lower this is to lower the CPFP max size, without effecting "honest" CPFPs. Originally the child size was something like 4kvB, we already bikeshedded it once!

@0xB10C made some nice charts which may inform better as well: https://twitter.com/0xB10C/status/1743626031070630038

-------------------------

instagibbs | 2024-01-09 14:46:24 UTC | #29

[quote="rustynail, post:26, topic:340"]
@glozow How are V3 transactions supposed to work with HTLCs anyway? Going to have V3 HTLC-success/failure transactions too?
[/quote]

Spec for HTLCs are identical to today, pretty much. Making those non-pinnable is out of scope for now as it would be a significantly larger spec change, with clear drawbacks. I think we can do better later without bothering spec people too much :slight_smile:

-------------------------

rustynail | 2024-01-09 17:13:30 UTC | #30

[quote="instagibbs, post:28, topic:340"]
The exact pin will indeed rely on exact mempool conditions, which is why I mostly just said “5x max” in my BIP text
[/quote]

But then why would you use an ephemeral anchor which can be maliciously pinned by **anyone** instead of a normal anchor with a checksig? Why make the attackers able to attack more stuff?

-------------------------

rustynail | 2024-01-09 17:14:59 UTC | #31

[quote="instagibbs, post:29, topic:340"]
Spec for HTLCs are identical to today, pretty much. Making those non-pinnable is out of scope for now as it would be a significantly larger spec change, with clear drawbacks. I think we can do better later without bothering spec people too much
[/quote]

@glozow Is this true? Why aren't we figuring out this stuff now while the spec is developing? I thought V3 was supposed to fix all these pinning issues.

-------------------------

instagibbs | 2024-01-09 19:42:52 UTC | #32

[quote="rustynail, post:30, topic:340"]
But then why would you use an ephemeral anchor which can be maliciously pinned by **anyone** instead of a normal anchor with a checksig? Why make the attackers able to attack more stuff?
[/quote]

Assuming we're talking specifically about LN, every choice has tradeoffs. From what I can see we have these shorter term choices:  
1) 2 Keyed anchors: Relies on CPFP carveout([which is going away eventually due to incoherence in cluster mempool world](https://delvingbitcoin.org/t/an-overview-of-the-cluster-mempool-proposal/393#the-cpfp-carveout-rule-can-no-longer-be-supported-12)). Not really viable.  
2) 1 Keyed Anchor(with no sharing): Relies V3 + package RBF which means you have to pay for absolute fee of counter-party's package + incremental bytes (slightly worse than replacing just child with adversarial counterparty).  
3) 1 Shared Keyed Anchor: Relies on V3 + package RBF. Both parties can independently spend the same anchor. Pinning bounded by the child tx size rule. Requires all other outputs be relative timelocked for at least one block, and costs the additional vbytes for a keyspend. Output must be above dust value. Allows for package RBF or direct RBF against anchor spends. Allows for "theft" of base anchor value.
4) Keyless anchor: In benign cases, strictly cheaper to spend, and results in smaller commitment transactions. Allows using any outputs previously encumbered by `1 CSV` to be unlocked for potential CPFP. Downside is it allows general adversaries to try and pin in the same reduced way as the highly motivated counter-party could in (3).

For more general smart contracting, it has plenty of benefits over CPFP-carveout based solutions: https://github.com/instagibbs/bips/blob/527b007dbf5b9a89895017030183370e05468ae6/bip-ephemeralanchors.mediawiki#motivation

[quote="rustynail, post:31, topic:340"]
@glozow Is this true? Why aren’t we figuring out this stuff now while the spec is developing? I thought V3 was supposed to fix all these pinning issues.
[/quote]

It was previously investigated by me, presented to LN spec group last year in NYC in person, and shelved because all HTLC-Success paths must be pre-signed to commit to some sort of opt-in policy, and either:
1) If using V3+ephemeral anchors, additional overhead bytes for HLTCs and protocol changes (icy reception to additional bytes)
2) Some sort of "V4" transaction which helps directly with 1-input-1-output case (Investigated this twice; not many people seemed to like this either).

In summation, we're trying to offer something people can use to safely replace CPFP carveout, whatever that is, while also improving the situation for wallets who would prefer some anti-pin features.

-------------------------

ajtowns | 2024-01-22 10:53:08 UTC | #33

In my head, the idea here is that opting into v3 is a collaboration between (some) node operators and transaction creators to ignore potentially valid transactions in order to more readily relay high feerate alternatives.

So currently you might see txs:

 * :white_check_mark: P = small parent, low feerate
 * :white_check_mark: C1 = huge child, low feerate, high fee
 * :x: C2 = small child, high feerate, modest fee, conflicts with C1

and end up with P and C1 stuck in your mempool and refuse to accept or relay C2, even though due to its high fee, it might well even be included in the next block. That remains a reasonable decision by individual nodes, as simply replacing C1 by C2 would make nodes vulnerable to relay spam: broadcast many C1's with lots of data, get that distributed across the network for free, then replace those txs with C2's, making sure that you're only actually paying a small amount in fees for all that relay spam. This is known as the "free relay" problem.

The workaround that v3 makes available is simply that C1 is now rejected in the first place in many cases. This prevents free relay (C1 isn't relayed at all), and solves the incentive compatibility problem where a tx that would be acceptable in the next block (C2) isn't relayed at all.

This works okay even if only adopted by a proportion of the network: C2 is still able to propagate over the subgraph of nodes that implement this policy eventually reach miners that run a policy compatible with this policy, who will eventually mine it.

It is in no way a perfect solution to pinning -- some systems will be designed in a way that a large child is sometimes necessary and those systems won't be able to opt-in to v3 rules. Also, even relatively small children can create a fee amount/rate pin, and there are other pinning vectors than high-fee/low-feerate. That's fine: making things incrementally better for some people is still making things better.

The limitations with v3 are quite annoying: one ancestor / one descendant means you can only have a pair of related v3 txs in the mempool, and nothing more complicated. In particular you can't do batch CPFP where a single v3 child pays for a bunch of v3 parents. But again, slightly better is still better, and relaxing policy rules if we figure out better ways of doing things is less problematic than restricting policy rules.

With just the v3 constraints, I think you'd want to design your protocol such that either parent transactions only have a single output that's spendable immediately -- that way any CPFP spends will naturally conflict with each other and RBF rules will apply, and the size limits applying to the child tx will limit the maximum fees those children will need to pay. However if there are multiple immediately spendable outputs that all have an n-of-n multisig arrangement, and all the txs that spend those outputs spend some common output, that could work as well. That approach seems unlikely to be useful in practice though?

When you add the ephemeral anchor rule, in particular that the EA output must be spent for the tx that creates the EA output to remain in the mempool, it becomes okay to allow other outputs to be immediately spendable: each child must spend the EA output, so they must conflict with each other, and RBF rules are applied. That seems to me like a significant bonus in flexibility -- allowing lightning commitment tx balance outputs to be spent unilaterally by the same tx that spends the EA output, which I think would be problematic otherwise.

-------------------------

instagibbs | 2024-01-22 13:34:33 UTC | #34

[quote="ajtowns, post:33, topic:340"]
With just the v3 constraints, I think you’d want to design your protocol such that either parent transactions only have a single output that’s spendable immediately – that way any CPFP spends will naturally conflict with each other and RBF rules will apply,
[/quote]

If you are expected to be able to conflict the parent this isn't necessarily required.

e.g., exchange doing a batched payout can sign multiple versions of withdrawals at different feerates and confidently replace them with up to 1kvB overhead if customers attempt sweeps. Locking customerss addresses with `1 CSV` isn't a think you can really do.

-------------------------

Chris_Stewart_5 | 2024-02-05 14:49:55 UTC | #35

Hi all, 

Adding to the use case pile for packages, DLCs would use packaging to guarantee [broadcast of refund transactions](https://github.com/discreetlogcontracts/dlcspecs/blob/9cd9148938c616690c79d99ec6f330e213c246c5/Transactions.md#refund-transaction).  

Refund transactions are signed at contract funding time. They become valid after a negotiated `refund_locktime` in the offer message. As the fees are calculated at contract funding, when it comes time to refund the funds because the oracle didn't do their job, fees on the network could be drastically different.

With packages, you can now submit a package of `(refund_tx, CPFP_refund_tx)` to get your refund tx confirmed. Either Alice or Bob could submit this package as both would have valid outputs on `refund_tx`

-------------------------

