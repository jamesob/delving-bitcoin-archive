# Timewarp, Miners Harvesting and Vaults

ariard | 2023-11-24 05:39:40 UTC | #1

## The Difficulty Adjustement

One of the cornerstone of Bitcoin security model is keeping the inter-block time (IBT) to 10 min in average, as such allowing the peer-to-peer network of full-nodes participating in block-relay to converge on a common chain tip, and avoid high rate of orphan blocks.

The stability of the inter-block time is guaranteed by the difficulty adjustment algorithm (`CalculateNextWorkRequired` in `src/pow.cpp`), every 2016 blocks, the actual interval between the timestamp of the latest block and the timestamp ancestor of the first block of a 2015-period is computed. The timestamps are expressed in UNIX epoch time. The interval is multiplied against the target of the latest block of the period and the result is divided by the expected target timestamp of 2 weeks.

The period of 2015 blocks to compute the interval is considered by some Bitcoin protocol developers as an off-by-one error (`GetNextWorkRequired` in `src/pow.cpp`), here since the early days. This bug explains why the Bitcoin re-targeting does not happen exactly every 600 seconds.

## The Timewarp Attack

The block timestamps constraints can be played with where the latest blocks in the 2015-blocks period is pinned in the future with a bogus timestamp to arbitrarily adjust difficulty adjustment, still in the minimum and maximum bounds allowed by the difficulty adjustment algorithm.

A mining attacker can leverage this flaw in the difficulty adjustment algorithm to permanently accelerate the effective block rate issuance under the interval block-time of 10 min. This attack is hard to exploit as it assumes a mining attack with more than 51% of the total hashrate network to mine the latest block of the period with superior chance of success over the "honest" hashrate. Without 51%, the "honest" hashrate should correct the effective difficulty target to the natural level of difficulty.

As a known effect of a timewarp attack, the coins inflation scheduled is accelerated and the limit of 21 millions of Bitcoin in circulation would be reached out faster than its natural schedule of emission.

## Vaults and Timelocked Wallets

Vaults are a set of proposed protocols design to reduce the risk of Bitcoin theft and enable fine-grained custody policy rules. Basically, a vault works as a multi-step chain of transactions, where the spends are constrained with (deleted-key or multi-party) signatures or covenant-enabled hash.

Most of designs have emergency paths, with a special transaction that can be broadcast to clawback the unvaulted funds to a "cold" address, normally at anytime during the custody period of the funds. Vaults or timelocked wallets can also knows inheritance paths, where the long-term (e.g 1 > month) spends might be activated by another stakeholder of the vaulted funds, if the wallet UTXO is not re-spend before a certain expiration time.

Vaults or timelocked wallets can have Script paths automatically spent by watchtower or network monitor, without custody infrastructure team manual actions. This automatic reaction might happen when some conditions are met (e.g low-block rate which might be a hint of an ongoing eclipse attack) or omitted (e.g the usual vault transaction is not spent to a "hot wallet" address after X).

## Miners Harvesting Attacks by Exploiting Timewarp

Let's assume a wallet with a recovery path timelocked under an absolute height-based time lock of chain tip T + 52560 blocks (1 year). The wallet initialisation transaction is broadcast and confirms at chain tip T. Transactions can be pre-signed or covenanted with a high-feerate (e.g 500 sat/vbyte), or connected to a highly-provisioned fee-bumping reserve.

The protocol fingerprint of the initialisation transaction might be deanonymized, e.g looking on the chain of ancestors transactions and seeing an older inheritance path played out with a characteristic time lock height duration and amount.

Once the initialization transaction has been deanonymized, miners know there are odds of a recovery path which might be automatically swept by a watchtower, with a spending transaction with a high-feerate. This spending transaction is played out as soon as some chain tip height is reached.

A 51% coalition of miners can engage in a timewarp attack to accelerate the block rate issuance after few difficulty periods, therefore coming to an event where the recovery path is automatically spent, and the high-fee pocketed in. Attacks might be batched against multiple timelocked wallets, massively bumping all miners income, including the one in the coalition.

Assuming multi-signature and multi-stakeholders redemption path, wallets owners might have operational and time constraints interferring with their abilities to manually sweep the wallet funds to a harvesting-safe address.

## Miners and Vaults Owners Misalignment of Incentives

In a post-subsidy world, when the mining income is expected to come in majority from on-chain transaction fees and only marginally from coinbase reward, miners might have the incentive to coordinate at a 51% level and target some class of Bitcoin use-cases, e.g time-sensitive L2s.

On the other hand, vaults and timelocked wallets have safety properties to provision their on-chain operations with very high-level of fee-bumping reserves, or commit their transactions at a feerate mitigating the worst-case of network mempools congestion, as lack of timely confirmation might be a major jeopardy of their funds safety.

In sustained period of low-fees, i.e when the majority of miners are at risk of seeing business discontinuity in face of their hardware and operational investment, there is a game-theory incentive to attack long-term timelocked use-cases, extract more on-chain fees, while only disrupting marginally or negligibly the remaining majority of use-cases, like scarce collectibles transfers or off-to-on-chain settlements.

## Open Problems

From an attacker perspective, this is uncertain if the 51% threshold might be lowered with techniques tampering with the block-relay network (e.g selfish attacks) or if some advanced manipulations of the headers timestamps might allow other minors miners harvesting attacks.

From a defender perspective, this is uncertain what is the design of vaults presenting vulnerable paths to miner harvesting or deployment configurations lowering the bar for exploitations.

## Ressources

- "Blocking the time warp attacks" https://bitcointalk.org/index.php?topic=114751.0
- "Vaults and Covenants" https://jameso.be/vaults.pdf
- "Forward Blocks" http://freico.in/forward-blocks-scalingbitcoin-paper.pdf
- "BIP proposal: The Great Consensus Cleanup" https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2019-March/016714.html
- "On massive channel closing and fee bumping" https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-February/002569.html

-------------------------

jamesob | 2023-12-18 19:57:52 UTC | #2

[quote="ariard, post:1, topic:218"]
Once the initialization transaction has been deanonymized, miners know there are odds of a recovery path which might be automatically swept by a watchtower, with a spending transaction with a high-feerate. This spending transaction is played out as soon as some chain tip height is reached.
[/quote]

No vault schemes that I'm aware of - and certainly none of the concrete examples I've done with `OP_VAULT` - automatically transfer coins after some height, relative or otherwise.

As far as I can tell, the summary of this post is "using timewarp, miners are incentivized to speed up block issuance to claim high fee transactions associated with automatic vault recoveries." But I haven't actually ever seen a vault scheme that is vulnerable to this, because the recovery path (in every scheme I've seen) consists of both a relative timelock _and_ a recovery key.

Miners might be able to speed up the chain, but they can't sign with keys they don't have, so I don't think the timewarp attack represents some kind of new problem for vaults.

-------------------------

