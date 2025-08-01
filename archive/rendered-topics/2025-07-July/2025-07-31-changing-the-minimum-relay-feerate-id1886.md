# Changing the minimum relay feerate

glozow | 2025-07-31 20:07:11 UTC | #1

Yoohoo, big summer blowout - 90% off cheap transactions!!!!

Here’s a PR to Bitcoin Core to lower the minimum relay feerate from 1000sat/kvB to 100sat/vB: https://github.com/bitcoin/bitcoin/pull/33106 
There's a bit of a writeup there, but a lot of it is Bitcoin Core-specific. Let's use this thread for discussion that is more conceptual or applicable to the wider ecosystem.

This is motivated by the [recent trend](https://x.com/mononautical/status/1949452586391855121) of sub-1sat/vB transactions being relayed and mined and a [PR](https://github.com/bitcoin/bitcoin/pull/32959) opened to Bitcoin Core. Blocks with lots of sub-1sat/vB transactions don't propagate as quickly to nodes that rejected or didn’t hear about those transactions earlier.

I think we should try to avoid block relay problems, but also shouldn't make it too cheap for an attacker to use a huge amount of network resources. Sending 1 valid transaction means tens of thousands of nodes using bandwidth to download/broadcast it.

The minimum relay feerate is a DoS protection rule, representing a price per unit bandwidth on this network-wide resource. The incremental relay feerate is similar: it's used to make replacement transactions pay "new fees" for relaying themselves. It's also used to set the dynamic mempool minimum feerate, which is like making new transactions cover the cost of the evicted ones, even though they aren't replacements.

Since the feerate represents a ratio between tx fees and real world resources, I think it's not entirely crazy to look at the price of BTC in USD. For example, the rationale given [here](https://github.com/bitcoin/bitcoin/pull/32959#issuecomment-3095260286) makes sense to me:

> 1kB of data across 100k nodes is at least 100MB of data, at 10c/GB (contemporary ec2 bandwidth pricing) is 1c total; and 1000sats at $250/BTC 0.25c total. Today, ec2 prices don't seem much different, and node counts aren't much higher, so at current prices, paying for relay would be closer to 0.01sat/vb. A rate of 0.1sat/vb would cover growing the node count to 1M nodes, and 1sat/vb would cover growing the node count to 10M nodes. So from a "free relay" perspective, I think 0.1 sat/vb is likely fine, both for min-fee and min-incremental-fee. If the p2p network grows to much more than 1M nodes, that might not be true though.

Curious whether others have thoughts on how the minimum feerates are used, what the minimum feerate should be, how to formalize this, what wallets/applications are using, etc.

-------------------------

garlonicon | 2025-08-01 06:27:52 UTC | #2

> Blocks with lots of sub-1sat/vB transactions don’t propagate as quickly to nodes that rejected or didn’t hear about those transactions earlier.

The same is true with any transaction, which is considered non-standard by existing nodes. Does it mean, that standardness rules should be discarded, if miners will start confirming free transactions, and using future Segwit versions for random data pushes, up to 4 MB per transaction? Because in this way, any existing limit can be lifted.

> The minimum relay feerate is a DoS protection rule, representing a price on the network bandwidth used to relay transactions that have no PoW.

Does it mean, that it would be different, if transactions would contain Proof of Work? Because they can, and it can be enforced inside Script, by checking the size of DER signature.

> block minimum feerate: shouldn’t be above min relay feerate, otherwise the node accepts transactions it will never mine

Which can be seen as a benefit, when it comes to batching: many low-fee transactions could be shared in P2P network, and they could contain sighashes, which would make them open for modifications and adjustments. And then, nodes can keep batching their transactions, and pushing them through full-RBF, until reaching 1 sat/vB, when the final transaction can land in a block.

Also, it is about scaling as well: by keeping on-chain fees always low enough, so any user can make cheap on-chain transactions on its own, usage of second layers is discouraged. Because if everyone can do everything on-chain, and it will always land in produced blocks, then why do we need Lightning Network, sidechains, or any other second layer at all? In this case, we could allow paying one satoshi per transaction, because other limits like 300 MB default mempool size limit, or 4 MB maximum block size limit will still protect us.

> dust feerate: I don't think these arguments apply in the same way

What if that limit will be lifted by miners as well? Would we have yet another discussion about changing default settings again, just because miners did it?

-------------------------

