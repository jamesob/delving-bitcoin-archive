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

1440000bytes | 2025-08-03 02:22:45 UTC | #3

[quote="garlonicon, post:2, topic:1886"]
The same is true with any transaction, which is considered non-standard by existing nodes. Does it mean, that standardness rules should be discarded, if miners will start confirming free transactions, and using future Segwit versions for random data pushes, up to 4 MB per transaction? Because in this way, any existing limit can be lifted.
[/quote]

This was [discussed](https://bitcoin-irc.chaincode.com/bitcoin-core-dev/2025-05-22#1123266;) on IRC and nobody really has a convincing answer. I wonder if [non-standard transactions](https://mempool.space/tx/bb41a757f405890fb0f5856228e23b715702d714d59bf2b1feb70d8b2b4e3e08) that take a long time to validate would become standard when a percentage of miners include them in blocks.

-------------------------

glozow | 2025-08-04 17:09:18 UTC | #4

[quote="garlonicon, post:2, topic:1886"]
Does it mean, that standardness rules should be discarded, if miners will start confirming free transactions, and using future Segwit versions for random data pushes, up to 4 MB per transaction? Because in this way, any existing limit can be lifted.
[/quote]

[quote="1440000bytes, post:3, topic:1886"]
I wonder if [non-standard transactions](https://mempool.space/tx/bb41a757f405890fb0f5856228e23b715702d714d59bf2b1feb70d8b2b4e3e08) that take a long time to validate would become standard when a percentage of miners include them in blocks.
[/quote]

No. If the intention was to just loosen policy rules completely, we wouldn't be having a discussion about what was appropriate for DoS protections. The argument being made is: if policy is stricter than necessary and prevents nodes from hearing about transactions that are likely to be mined, we should loosen it to only what is necessary.

> Does it mean, that it would be different, if transactions would contain Proof of Work?

Yes. It sounds like a nice idea, but I'm not sure it's reasonable to impose something like this on signing devices. The work requirement would need to be fairly high to be effective.

-------------------------

garlonicon | 2025-08-05 03:36:29 UTC | #5

> It sounds like a nice idea, but I’m not sure it’s reasonable to impose something like this on signing devices.

1. It can be fully optional, and used only when needed, by wrapping it in `OP_IF <proof_of_work> OP_ELSE <other_conditions> OP_ENDIF`.
2. It is already deployed on-chain, and it works as a faucet, giving satoshis to puzzle solvers: https://bitcointalk.org/index.php?topic=5551080.0
3. It is possible to combine Proof of Work with any existing Script, except TapScript. Which means, that for example LN transactions or things from other second layers (like sidechains) could contain optional Proof of Work, to limit transaction replacements.

> The work requirement would need to be fairly high to be effective.

Yes, but many users could work on the same transaction. Then, one of them would broadcast the solution, so all second layer users can benefit from it. Which means, that if honest second layer miners will have more power than any attacker, then it would work. And the main requirement is to get enough Proof of Work, to make it confirmed in the next block, which means, that if Proof of Work requires many days or months of grinding, and the transaction is confirmed after 10 minutes, then attackers have very low chances to double-spend it in practice (and by double-spending, it would mean things like "sharing the old state of some LN channel").

So, to sum up: any given transaction can use two ways to deal with spam: one is transaction fee, another is Proof of Work. And then, it is all about finding an equilibrium, and deciding, if a given user wants to put more coins in, or more Proof of Work instead. Personally, I have nothing against allowing even free transactions, as long as they will have significant Proof of Work requirements, to prevent it from being abused.

-------------------------

1440000bytes | 2025-08-05 07:40:00 UTC | #6

[quote="glozow, post:4, topic:1886"]
The argument being made is: if policy is stricter than necessary and prevents nodes from hearing about transactions that are likely to be mined, we should loosen it to only what is necessary.
[/quote]

A few suggestions for the open pull request if changing the minimum relay feerate is necessary:

1. Sub 1 sat/vbyte transactions don't deserve to be in the pool with other transactions. A subpool should be possible if these transactions follow a different `maxmempool` and `mempoolexpiry`. *Example: 32 MB and 36 hours*
2. If the subpool idea makes sense and does not introduce [free relay](https://bitcoinops.org/en/topics/free-relay/) then `minrelaytxfee` could be dropped to anything above zero.
3. It does not make sense to leave the fee estimates unchanged if lower fee transactions are relayed and mined.

In the long term, mining pools will realize the [revenue](https://studio.glassnode.com/charts/fees.VolumeSum?a=BTC&mScl=log) will drop further with lower fee rates. It has only made new lows since 2017 (fees in BTC per day), price (in USD) is only ~5x since then and reward down from 12.5 BTC to 3.125 BTC.

Still hopeful that [most blocks](https://mempool.space/graphs/mining/block-fee-rates#1w) will continue to have minimum fee rate above 1 sat/vbyte and users will prefer L2 for lower fees.

-------------------------

Sceptic | 2025-08-06 15:44:53 UTC | #7

Why the sudden urgency to push this change into the next release? Policy changes usually aren’t rushed.

-------------------------

glozow | 2025-08-06 16:47:35 UTC | #8

[quote="Sceptic, post:7, topic:1886"]
Why the sudden urgency to push this change into the next release?
[/quote]

See https://github.com/bitcoin/bitcoin/pull/33106#issuecomment-3155627414

-------------------------

davidgumberg | 2025-08-12 15:24:47 UTC | #9

I wonder if it is possible to set minimum relay fees dynamically and preserve DoS protections, without requiring a release cycle for Bitcoin Core policy to catch up with blockspace demand and the exchange rate between BTC and EC2 bandwidth.

If one only had to account for blockspace demand the problem would be more tractable since the only thing that needs to be considered is what transactions get included in blocks, and you could pick some metric like percentile of transactions included in blocks you want to fit above the minimum relay fee. Coupled with @0xB10C’s prefill [proposal](https://delvingbitcoin.org/t/stats-on-compact-block-reconstructions/1052/24), this could mitigate compact block relay issues related to minimum relay feerate.

But, miners don’t pay the total cost to the network of relaying a transaction, they only pay their own small portion of that cost. So, in some circumstances it might be profitable for miners to include transactions at a feerate (sats/vB) below the network’s relay cost (# of nodes \* cost of bandwidth in sats/byte \* 4 bytes/vB^[Multiplied by 4 because in the worst case a vB is four bytes. (BIP-0141#transaction-size-calculations]*).* For an example estimate of network relay cost, that follows along the same lines as @ajtowns [calculations](https://github.com/bitcoin/bitcoin/pull/32959#issuecomment-3095260286): taking EC2’s most expensive bandwidth tier of $0.09 / GB, and 1 BTC = 100,000 USD, the cost of bandwidth is $90 \\frac{\\text{sats}}{\\text{GB}} = 9 \\times 10^{-8} \\frac{\\text{sats}}{\\text{Byte}} = 3.6 \\times 10^{-7} \\frac{\\text{sats}}{\\text{vB}}$. Multiplied by 100,000 ($10^5$) nodes the total network bandwidth cost is $0.036 \\frac{\\text{sats}}{\\text{vB}}$. In this scenario, the proposed minimum relay feerate of 0.1 sats/vB makes DoS’ing the network with transaction relay still more expensive than DoS’ing it with EC2.

But, determining the cost to externally relay to the Bitcoin network seems like it will always require external information (cost of internet bandwidth in sats / 4 bytes) and accurately determining the number of nodes on the network is difficult and will always be subject to sybiling, so will Bitcoin Core always have to hardcode minimum relay feerates, responding to network / market shifts one release cycle after they happen?

-------------------------

glozow | 2025-08-12 13:56:31 UTC | #10

> will Bitcoin Core always have to hardcode minimum relay feerates, responding to network / market shifts one release cycle after they happen?

There are a number of hardcoded things in Bitcoin Core that are based on “real world” values and updated every 6-12 months. The [headerssync parameters](https://github.com/bitcoin/bitcoin/blob/273e600e65c2e31a6e9a0bd72b40672aaa503b08/contrib/devtools/headerssync-params.py#L24) depend on specs of modern processors (the speed a “Ryzen 5950X CPU thread can hash headers“), the length of the chain, and the release sunset date. Fallback [fixed seeds](https://github.com/bitcoin/bitcoin/tree/master/contrib/seeds) hardcode a snapshot of current nodes on the network, which is highly ephemeral.

Values like network size and approximate computation/bandwidth costs change very gradually, certainly slower than the pace at which other parts of the software are modernized/fixed. The current proposal is to just be within an order of magnitude of what these back-of-the-envelope calculations come out to. I don’t think we should tweak it at every release, but it’s sensible and normal to have these values grounded in an economic reality that can change over time.

> mitigate compact block relay issues related to minimum relay feerate

I completely agree with a more comprehensive solution to poor compact block reconstruction rates. In this case, however, we can diagnose the problem (at least the majority of it) as a single policy rule that is stricter on the network than for miners, and one that we should have revisited sooner than once every 10 years.

-------------------------

