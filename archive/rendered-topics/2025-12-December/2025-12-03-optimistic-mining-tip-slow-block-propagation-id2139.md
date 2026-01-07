# Optimistic mining tip (slow block propagation)

optout | 2025-12-03 10:41:11 UTC | #1

I’d like to get some comments on a mitigation idea for slow block propagation for miners.
The topic of how slow block propagation can harm miners has been discussed recently in many places (e.g. https://bitcoinops.org/en/podcast/2025/11/25/ ). The topic often came up on the side of compact block propagation delays due to transaction filtering (but that’s not my focus here).

The straightforward-looking idea is to mine optimistically on a new chain tip while its validation is pending. I wouldn’t be surprised if this has been discussed before, or there are some reasons against it, or if my understanding is incorrect, so any feedback is welcome.

When I new (compact) block is received from a peer, the phases are:

* validation of the header (hash, proof of work)
* if not all transactions are available, requesting the missing ones (can cause delay)
* validation of all transaction
* accepting as a new tip.

The idea is to temporarily accept the new tip for mining, while validation is pending.

* After header validation, the block is placed in a special slot for optimistically-accepted-block-pending-validation, let’s call it “optimistic tip” for short.
* If optimistic tip is set, mining template generation considers it
* When validation succeeds, the block becomes the ‘normal’ tip, the slot can be cleared, everything returns to normal.
* When validation fails, the slot is cleared, thus tip revers to the old.
* If the validation is not finished in a give time, e.g. 20 seconds, the slot is cleared (revert to old tip). Note that validation still continues, and it’s result is processed as normal.

The difference is that during the validation period mining is directed to the new tip, reducing the risk of orphan blocks.
The negative outcome is if for some reason the new tip fails validation, which should be a rare case, and the loss in the case is the wasted mining time for the limited time (e.g. max 20 secs).
As the block headers are work-protected, I see no risk of an attack vector (intentionally forcing the node into mining on an invalid block).

-------------------------

mcelrath | 2025-12-03 12:49:27 UTC | #2

Indeed this has been discussed for a long time. Sometimes it's called “headers first mining” or “SPV mining” and it's a [cause of empty blocks being mined](https://bitcoinmagazine.com/business/why-do-some-bitcoin-mining-pools-mine-empty-blocks-1468337739). (When you do this you can't have any txs in your block because if anything in your block conflicts with the block you haven't validated yet, your block will be invalid). You are however taking a risk on the incoming block being valid and may generate orphans if it is invalid.

Braidpool will solve this in a unique way: by using our committed mempool, block templates are deterministic and already known to all Braidpool nodes. Often we will have pre-computed the template for the incoming block and validation is as fast as comparing the Merkle root. When we can't do that it's because we're missing parent shares (beads) of the incoming block. But that validation is still much faster because we only need to validate \~5 txs, not the entire block. Our share relay is also a lot faster than Bitcoin because shares are small and we use UDP/QUIC to avoid a round trip. Every Braidpool node can generate the corresponding block and feed it to Bitcoin once it sees the share.

Another optimization we may implement is to precompute the template for a *second* block, so if the block we expect comes in as a Bitcoin block, we can instantly switch to the second template, full of transactions, maximizing miner profit. This will matter more in a high fee rate environment, and of course it only works for blocks mined by Braidpool. For blocks mined by others we're in the same situation as you describe. But, it's safe for 100% of miners to be on Braidpool. The 51% attack still exists but it's transferred to Braidpool itself.

-------------------------

AntoineP | 2025-12-03 16:01:19 UTC | #3

[quote="optout, post:1, topic:2139"]
The straightforward-looking idea is to mine optimistically on a new chain tip while its validation is pending.
[/quote]

As Bob points out validationless mining is a common bad practice that has been discussed for a long time. It is sometimes referred to as SPV mining (mining on top of a block after validating only its header, not its content) or spy mining (listening for jobs on competitors' pools and using their previous block hash as soon as they switch[^0]).

Validationless mining undermines the security of the system in at least two ways. First it risks extending an invalid chain, creating the opportunity for fake confirmations. This can happen either maliciously, by feeding an invalid block to a non-validating miner, or inadvertently in case of a consensus rules change. It famously occured during the BIP 66 soft fork where numerous invalid blocks were created due to validationless mining, producing invalid branches as long as 6 (!!) blocks [^1]. Secondly, if the cost of trustless validation is too high it incentivizes mining pools to enforce validity through alternative means, such as a state's legal system, in order to make sure they are not wasting work.

There is ways to mitigate validationless mining, but not avoid it completely. The best we can do is to ensure access to full validation is as cheap as possible so pools only send potentially-invalid jobs to hashers for the smallest possible amount of time. This requires optimising block propagation and validation times in the common scenario (compact blocks, mempool synchronisation) as well as preventing exploiting long block validation times in the attack scenario (see [BIP 54](https://github.com/bitcoin/bips/blob/9a30c28574e62e26da77f14e33eb698b81268887/bip-0054.md), Consensus Cleanup).

You may also be interested in [this previous proposal](https://gnusha.org/pi/bitcoindev/CAAS2fgRwfQNYxCmDPAnVudyAti9v8PPXQjxe9M13pmrFxKcSCQ@mail.gmail.com/) for miners to voluntarily signal they only partially validated the block they are building on top, as a gentlemen agreement to provide risk mitigation. The given motivation seems to be what you are after:
>  Although this validation skipping undermines the security assumptions
of thin clients, it also has a beneficial effect: these delays also
make the mining process unfair and cause increased rewards for the
largest miners relative to other miners, resulting in a centralization
pressure.  Deferring validation can reduce this pressure and improve
the security of the Bitcoin system long term.
> 
> This BIP seeks to mitigate the harm of breaking the thin client
assumption by allowing miners to efficiently provide additional
information on their level of validation.  By doing so the
network can take advantage of the benefits of bypassed
validation with minimal collateral damage.

[^0]: See for instance [this issue](https://github.com/bitcoin/bitcoin/issues/3658#issue-27423298) for a public mention of this practice.
[^1]: See [this SE question](https://bitcoin.stackexchange.com/q/38437/101498), although as far as i can tell the currently accepted answer is missing some invalid blocks that were created at the time.

-------------------------

optout | 2025-12-05 09:11:56 UTC | #4

Thanks for the explanation and links, Bob & Antoine (P)!
I have simply overlooked the fact that block validation is more than the act of validation of the block itself, but it also includes the updating of the state: UTXO set, etc.
It’s not surprise that this is an issue that came up often during the history of bitcoin (I was not aware of the details).
It’s a fun fact that Gregory Maxwell’s verification flag proposal was exactly 10 years old yesterday!

-------------------------

optout | 2025-12-05 09:12:48 UTC | #5

Thanks for the explanation! I’ve came across Braidpool earlier, but I have to put some effort to grasp it. It’s good to know that it has such benefits as well.

-------------------------

optout | 2025-12-05 09:48:04 UTC | #6

[quote="AntoineP, post:3, topic:2139"]
The best we can do is to ensure access to full validation is as cheap as possible so pools only send potentially-invalid jobs to hashers for the smallest possible amount of time. This requires optimising block propagation and validation times in the common scenario (compact blocks, mempool synchronisation) as well as preventing exploiting long block validation times in the attack scenario (see [BIP 54](https://github.com/bitcoin/bips/blob/9a30c28574e62e26da77f14e33eb698b81268887/bip-0054.md), Consensus Cleanup).

[/quote]

You gave a nice summary of potential improvements which can help with block propagation times. I expand a bit on that.

\- Reduced relay-filtering. The objective of seemingly minimizing the chance of a transaction being mined by denying its relay is simply not compatible with the aim of fast propagation of any consensus-valid transaction. This has been extensively debated here and elsewhere.
- Recent proposals for next-block-template synchronizations (could help a lot) 
- Faster block validation
- Faster UTXO, mempool state update, template creation – I suppose Cluster mempool can be beneficial here

Anything else to mentioning?

I think it’s also worthwhile to mention that only a small proportion of bitcoin nodes are mining (i.e. generating block templates used by miners), and only these few nodes are directly motivated in lower block propagation times. There are some things such a node can control, such as good peering, large mempools, no filtering, etc.). Most non-mining nodes are not much hindered by block delays (except e.g. node with high transaction generation). However, global transaction propagation is an emergent property of the whole network, and requires cooperation from the non-mining nodes as well.

On a related note, more miner-generated templates (as opposed to pool-generated) helps reduce mining centralization, but may have a drawback of affecting negatively block propagation times, because the set of block-creator nodes becomes larger and more diverse. (Most Ocean-mined blocks are with custom template even today, and with the long-awaited Stratum V2 adoption this may get more widespread).

-------------------------

AntoineP | 2025-12-05 15:10:52 UTC | #7

[quote="optout, post:6, topic:2139"]
Anything else to mentioning?
[/quote]

Optimizing internal operations of mining pools, it's likely where the bottleneck is now. See [this comment](https://delvingbitcoin.org/t/propagation-delay-and-mining-centralization-modeling-stale-rates/2110/3?u=antoinep) on my stale rate modeling thread that shares a blog post which goes into the performance improvement of using Stratumv2 for pools. The fact that some pools appear to be making their own (!!) blocks stale appears to corroborate this (see [here](https://bnoc.xyz/t/antpool-mines-two-blocks-at-height-925051/60?u=antoine) for Antpool mining two blocks at the same height, and [here](https://bnoc.xyz/t/viabtc-mining-pool-mines-multiple-blocks-at-same-height/48?u=antoine) for ViaBTC mining two blocks at the same height).

-------------------------

