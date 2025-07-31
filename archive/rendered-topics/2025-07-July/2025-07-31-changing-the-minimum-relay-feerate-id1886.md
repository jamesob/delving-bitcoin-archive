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

