# Non-confiscatory Transaction Weight Limit

vostrnad | 2025-05-28 19:01:24 UTC | #1

Hi everyone,

as you might have heard, several mining pools announced today that they will partner up in supporting the mining of non-standard transactions. News article here: https://www.coindesk.com/business/2025/05/27/bitlayer-joins-forces-with-antpool-f2pool-and-spiderpool-to-supercharge-bitcoin-defi

I haven't been able to confirm for sure, but since the article specifically mentions BitVM, it seems one standardness rule they have in mind is the 100 kvB weight limit for transactions, as some BitVM transactions are supposedly larger than that.

The rule has multiple benefits, such as limiting the impact of quadratic hashing in pre-SegWit inputs, but the [Consensus Cleanup](https://github.com/bitcoin/bips/blob/72af87fc72999e3f0a26a06e6e0a7f3134236337/bip-0054.md) proposal already addresses that. The other primary benefit is that it makes building more optimal block templates easier, reducing the need for centralized template providers.

Ideally we'd want to make this limit a consensus rule, but that risks both impacting future uses of large transactions that could be tremendously valuable (in the case of BitVM that remains to be seen) and confiscating already existing UTXOs that require a large transaction to be spent. A weaker version of this rule might go something like this:

**A non-coinbase transaction weighing more than 400,000 WU must be the only transaction in a block apart from the coinbase transaction.**

This would preserve the ability to create oversized transactions for protocols that are valuable enough to afford a whole block for themselves from time to time, and it wouldn't make it any more difficult to create optimal templates. However, it has that one nasty drawback: MEV. Miners could still add more outputs to the coinbase transaction, i.e. space they could sell for data storage, and smaller miners would be at a disadvantage trying to provide this service.

Potential solutions that come to mind include limiting the size of the coinbase transaction in a block with an oversized transaction (however that would affect decentralized coinbase payout schemes) or allowing just one oversized transaction per block without disallowing smaller transactions (though I'm not sure how much that would actually help in building optimal block templates).

If you have other ideas, I'd be happy to hear about them. I don't think that long-term we want a Bitcoin without any sort of consensus check against oversized transactions. If they become at least somewhat common, miners could be forced to use centralized template providers to remain profitable ([as is the case today with Ethereum](https://www.mevwatch.info/)).

-------------------------

instagibbs | 2025-05-29 13:23:38 UTC | #2


I don't think softforking to make max size the current relay size is off the table. The relay limit has been in place since 41e1a0d7663d479f437c779df90775fc2bbc4087 in Bitcoin Core (about 12 years now). Relay/pinning/etc are significantly harder to reason about in the presence of oversized relay, and it would be better if we just didn't. 

Obviously it would be better if BitVM transitioned away from those transactions, and there may be hope that it's avoided anyways long term with some newer cryptographic work out there.

-------------------------

sipa | 2025-05-29 18:03:45 UTC | #3

Does anyone know if the protocols that "need" these giant transactions could instead make a deal with miners to have the transactions/data stored in their coinbases completely? Because if so, this proposal has no effect.

-------------------------

gmaxwell | 2025-05-30 01:07:45 UTC | #4

Another tool for the non-confiscatory belt to consider is that the height of inputs is available at validation time.  So one could, for example, except txn where all inputs are below some height to reduce the risk of breaking something.

One could also have a rule that sunsets and needs to be continually renewed to be maintained if you're concerned about future risk.  E.g. have it expire N years out and resoftfork in an extension every N/2 years.  This does have some political risk from the repeated consensus changes but could still be superior if the alternative is doing nothing at all.

As far as the actual objective, it's not clear to me that 1/10th is enough to make block construction sensible (though I'd be open to hearing an argument otherwise). ... and pretty clear to me that much smaller than that would be a non-starter, not just because of the historical behavior but also because big transactions can be encountered in ordinary usage.

-------------------------

vostrnad | 2025-05-30 07:38:02 UTC | #5

I believe protocols that need giant transaction generally don't just store data, they really do need large scripts. If they only needed to store data they could easily split it across multiple smaller transactions.

Regarding storing data in the coinbase, it occurs to me that if CTV is activated, we might be able to get away with severely restricting the size of the coinbase, as mass payouts can then just be trustlessly moved off to a subsequent, non-coinbase transaction.

-------------------------

light | 2025-05-30 15:47:03 UTC | #6

[quote="vostrnad, post:5, topic:1732"]
If they only needed to store data they could easily split it across multiple smaller transactions.
[/quote]

This is not strictly true; recall that in the case of the watchtower light-client protocol that ignited the recent OP_RETURN debate, the issue is that the protocol _needs_ to embed 144 bytes _in a single transaction_. Although I am not aware of any protocol that _needs_ to embed >100 kvB in a single tx, this example goes to show that we cannot assume that even in the simple _data storage_ case that it is sufficient to use multiple txs.

[quote="vostrnad, post:1, topic:1732"]
The other primary benefit is that it makes building more optimal block templates easier, reducing the need for centralized template providers.
[/quote]

I will note that since, as you say, "the [Consensus Cleanup](https://github.com/bitcoin/bips/blob/72af87fc72999e3f0a26a06e6e0a7f3134236337/bip-0054.md) proposal already addresses" the first rationale for your proposal, that this is the only rationale you have left to support this proposal. Therefore I think it's worth interrogating this further rather than accepting it as a given:

- How much "easier" does the current policy limit make building optimal block templates, exactly? (Assuming txs actually comply with the limit.)
- How do you quantify how much easier the rule makes building optimal block templates?
- How would a centralized template provider make building optimal block templates any easier? What are they doing that the miner themselves cannot?

(btw I am aware of the knapsack problem arguments and have validated it myself in private research, but I think it's worth going beyond a passing mention about it to justify forking for it to people unfamiliar with the problem.)

Taking a step back: maybe the news article about the miners agreeing to enable the larger BitVM-related txs was only the spark for this idea that is actually intended to address a larger concern, but as far as the BitVM-related txs are concerned, I want to point out that these large nonstandard txs only need to go onchain in the (expected to be[^1]) rare case that a bridge operator actually tries to submit an incorrect claim, is challenged, and finally has their response to the challenge (the "assertion") disproven. It is the `disprove` transaction that tends to be the large nonstandard tx, and it is the last tx in the challenge-response game that needs to be put onchain.[^2]

I mention this about BitVM to ask: how common have >100 KB txs actually been, and how common do we anticipate them to be in the future? Enough to harm mining enough that a soft fork is warranted? At least as far as BitVM is concerned, imo we have nothing to worry about there: even if such txs do end up needed, they will be infrequent enough that their impact on block construction will be negligible.

[^1]: I say it is "expected to be" rare because the bridge operator is heavily disincentivized from putting an incorrect claim onchain, let alone getting so far into the challenge-response process that their assertion is disproven: the bridge operator must put up collateral (currently in the range of multiple BTC) to make a claim, and if their claim is incorrect, this entire collateral could be seized from the operator, and if seized then part of it will be burned to prevent them from profiting by disproving themselves.

[^2]: This description is only applicable to [BitVM2](https://bitvm.org/bitvm_bridge.pdf) implementations.

-------------------------

instagibbs | 2025-05-30 15:40:29 UTC | #7

[quote="light, post:6, topic:1732"]
At least as far as BitVM is concerned, imo we have nothing to worry about there: even if such txs do end up needed, they will be infrequent enough that their impact on block construction will be negligible.
[/quote]

I believe this is likely to be true.

-------------------------

sipa | 2025-05-30 15:48:59 UTC | #8

I think the concern about block template construction is perhaps a bit too narrow. It is a complication on itself, and it gets worse as transaction sizes approach the block size, but it doesn't need too much hand-waving to assume software can be developed that can do block template building near-optimally *given a reasonable pool of candidate transactions*, and made available to anyone to use in mining operations. I think it would be far better if this was not needed, but there is a much more serious related issue.

The issue, as I see it, is that as block template building for miners becomes more complex, reasoning *by relay nodes* (who possibly do not mine) about profitability becomes harder too, and far more so than actual block building, because it is a **decision that needs to be made ahead of time**, possibly long before the transaction might make it into a block, while other intervening transactions may arrive between relay and the block being found. Without information about miner-incentive-compatibility at relay time, and barring trivially-DoSable "just relay everything" policies, an honest user who wishes to use giant transactions will not be able to rely on the public P2P network and anonymous miners to get the transaction confirmed anyway. And without the ability to rely on this, these users will be forced to make agreements with mining businesses anyway to get guaranteed block space. The goal of any proposal to deal with giant transactions must be **to make users able to get their giant transactions confirmed, without private agreements with miners, and without significant extra cost**. If a proposal fails at this, it does not matter how good block building available to the common denominator of miners is, because users and miners will bypass all of that anyway.

To illustrate this problem, first consider a (intentionally unrealistically) extreme example: a user who wants a consensus-valid 999800 vB transaction to be confirmed. This is a legal size, in that consensus-valid blocks could exist that contain such a transaction. But due the size of a block header plus the minimum size of a coinbase transaction that includes a witness commitment, no miner will ever include it without a private agreement, as it would require them to burn all block income: there is no space for any secure normal transaction output in the coinbase anymore. That is no problem if the user pays the miner out of band for an amount above the displaced income they would get from the block otherwise, but I don't see how it could be done in a manner that only uses the public network.

Okay, so we do need some limit on transaction sizes below that value, but how much? For exposition purposes, assume a hypothetical consensus limit of 990000 vB on transactions, assuming that no header+coinbase will realistically want to exceed 10000 vB. Does this solve our problem? Imagine this transaction pays a feerate that places it 20000 vB away from the top of the mempool (i.e., there exist 20000 vB worth of transactions that pay more). For DoS purposes (including relay rate limiting, RBF, and eviction decisions), the giant transaction will be treated as being better than **all** mempool transactions with lower feerate (possibly after chunking, though we can ignore that effect for the purpose of this discussion, as long as the giant transaction itself does not have big dependencies, or involved in CPFPs), including the 980000 vB worth of transactions that follow it. However, due to bin-packing effects, it may well be the case - even to an optimal knapsack solver - that this transaction stays outside of the optimal block template for a long time, or it may not be the case. It depends on the steepness of the fee-size diagram in the 10000 vB before it. This can result in bad relay decisions, bad replacement decisions, opening up the network to free relay attacks, transactions staying in the mempool which realistically won't actually be mined, and in the other direction if it happens closer to the bottom of the mempool, transactions which may be evicted despite being fairly reasonable for inclusion later. This effect is much less for small transactions, as they generally need a much bigger mempool change to be bumped to the next/previous block.

In short: for small transactions, mining quality is a static decision that can be made once (e.g. at relay time). **The larger a transaction gets, the more its mining quality will depend on other transactions available at block finding time.** Uncertainty about these means that reasoning for DoS purposes breaks down.

I'm not convinced about the approach suggested here - it feels like it should lead to similar problems still, but thinking practically I can't really see many issues. At least the "only 1 large transaction + coinbase in a block" rule is effectively equivalent to "treat every transaction as if it had ~1 MvB in size", block building becomes "try normal block building, only considering non-giant transactions, and compare the fee of that with the highest absolute fee giant transaction". For everything else, the giant transaction looks like it can pretty much be treated as having `tx.fee / 1 MvB` as feerate. Due to only one transaction fitting in a block, CPFP and other dependency issues are largely gone. Pinning might be a problem, but only if those who need such transactions require replaceability in adverserial settings.

(thanks to @sdaftuar for discussing this offline)

-------------------------

AntoineP | 2025-06-06 17:50:58 UTC | #9

[quote="instagibbs, post:7, topic:1732, full:true"]
[quote="light, post:6, topic:1732"]
At least as far as BitVM is concerned, imo we have nothing to worry about there: even if such txs do end up needed, they will be infrequent enough that their impact on block construction will be negligible.
[/quote]

I believe this is likely to be true.
[/quote]

Even if true, i think there is still a cause for concerns. Even if there is not substantial demand for large transactions, the mere fact that there is *some* induced miners to provide a publicly-accessible direct submission mechanism. This means that at any time now someone may release an application leveraging them which might become popular. Such an application becoming popular would make it significantly harder to cap transaction sizes by consensus. In other words the cat is out of the bag, if we really think it's a concern we should be proactive in fixing it.

-------------------------

ajtowns | 2025-12-11 23:37:47 UTC | #10

[quote="sipa, post:8, topic:1732"]
I’m not convinced about the approach suggested here - it feels like it should lead to similar problems still, but thinking practically I can’t really see many issues. At least the “only 1 large transaction + coinbase in a block” rule is effectively equivalent to “treat every transaction as if it had ~1 MvB in size”,
[/quote]

Having every tx greater than 100kvB be treated as being ~1000kvB seems like it would encourage stuffing -- "I need a 120kvB tx, but I have to pay for 880kvB extra anyway, so might as well find some garbage to fill that up with". Perhaps you could tweak this somewhat so that other people's txs could still what's used to fill up the block. Rough idea:

## Consensus rules

 * both the coinbase and the last tx in a block can be arbitrarily large, but every other tx must have a weight less than 400000
 * if the last tx in a block has weight more than 400,000:
   * it cannot spend an output that was created in the block (ie, it's treated as if it had an relative lock time of 1 block)
   * its weight is rounded up to 5000 below the next multiple of 100k, ie $w' \equiv 95000 \pmod{100000}$. This makes the max tx size 3,995,000 weight units or 998,750vB.

## Mempool acceptance, storage

When accepting large txs to the mempool, they cannot have in-mempool ancestors or descendants, so always have a cluster size of one.

Because their weights are rounded up for consensus purposes, each large tx in the mempool can be put into one of 36 buckets matching its rounded up weight (495,000 weight units through 3,995,000), and only the highest fee tx in each of those buckets needs to be considered at any point in time. Further, if the best tx in a higher weight unit bucket has lower fee than the best tx in a lower weight unit bucket, it can be ignored. Keep track of these ~36 non-ignored txs.

## Mining

When mining, take the ~36 non-ignored large txs from your mempool ordered from largest to smallest, put them in a `large_buckets` list, and run something like this algorithm:

```python
    block = []
    ignored_txs = 0
    large_bucket = 0
    while ignored_txs < 1000:
        chunk = mempool.get_next_chunk()
        if not chunk: break
        if block.weight() + chunk.weight() > MAX_WEIGHT:
            mempool.ignored_chunk()
            ignored_txs += 1
            continue
        while large_bucket < large_buckets.size()
            if block.weight() + chunk.weight() + large_buckets[large_bucket].weight() > MAX_WEIGHT:
                large_buckets[large_bucket].block = block.copy()
                large_bucket += 1
            else:
                break
        block.add(chunk)
        mempool.accepted_chunk()
        ignored_txs = 0
    if large_bucket < large_buckets.size():
        large_buckets[large_bucket].block = block.copy()
    for large_bucket in range(large_buckets):
        if not lb.block: break
        lb = large_buckets[large_bucket]
        if lb.block.fee + lb.tx.fee > block.fee:
             block = lb.block + lb.tx
    return block
```

That misses out on getting the best possible final 25kvB into a block that includes a large tx, but otherwise seems fairly feasible?

-------------------------

murch | 2025-12-12 17:58:55 UTC | #11

I haven’t reread the whole thread, but I just had a quick thought skimming AJ’s comment: if a large transaction is pre-signed and low feerate, limiting it to a cluster of one would mean that it cannot be CPFPed. So I was wondering whether it would be reasonable to limit it to a cluster of two transactions instead, where the second transaction is similarly limited as the child in TRUC transactions. Otherwise, the only way to get a low feerate pre-signed large transaction mined would be an out-of-band payment.

-------------------------

ajtowns | 2025-12-13 00:14:53 UTC | #12

You could RBF it, if you were a signer of it, of course.

-------------------------

