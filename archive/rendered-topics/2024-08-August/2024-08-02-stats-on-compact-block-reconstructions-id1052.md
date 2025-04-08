# Stats on compact block reconstructions

0xB10C | 2024-08-02 12:08:52 UTC | #1

I've recently looked at compact block reconstruction statistics from the debug.log's of my [monitoring nodes](https://public.peer.observer). Particularly, at how many blocks require an extra `getblocktxn` -> `blocktxn` round-trip as the node does not know about transactions during block reconstruction. I thought I'd share my data and a few observations here.

My nodes have `debug=cmpctblock` logging enabled. This allows me to grep for compact block reconstructions. The log output contains information on the number of transactions pre-filled in the compact block (here 1, the coinbase transaction), how many transactions the node could use from its mempool (3986), how many transactions were used from the extra pool (9) and how many were requested (6).

```log
[cmpctblock] Successfully reconstructed block 000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f with 1 txn prefilled, 3986 txn from mempool (incl at least 9 from extra pool) and 6 txn requested
```

While the monitoring nodes have slight differences in their node configuration, they mostly use Bitcoin Core defaults. All monitoring nodes accept inbound connections and the default inbound slots are usually full. I set `maxconnections=1000` on node `dave` - it usually has about 400+ inbound connections. On 2024-05-08, I switched node `frank` to `blocksonly=1`, which means it won't do block reconstructions anymore. I switched `erin` to `mempoolfullrbf=1` on 2024-07-26.

The following shows the share of block reconstructions that didn't need to request transactions - higher is better.

![image](upload://AwxNcgcknvS5mhs0cOLv8Zg4iT2.jpeg)


 I observed three periods with a 50% or less "compact block reconstruction without transaction request" rate across all nodes:

1. End of February till early March
2. Mid April till early May - especially bad around the halving.
3. End of May till mid June - especially bad around end of May

This seems to correlate with increasing mempool activity.
![image|690x257](upload://agOcDQLoH8PEZQVIN5OTnj3K1QN.png)

Node `dave` under performing between 2024-07-10 and 2024-07-27 correlates with it rising from about 320 to nearly 500 inbound connections. It was restarted on 2024-07-27 and inbound connections were reset. 

During other periods were `dave` had similar numbers of inbound connections, it didn't seem to have had an impact on block reconstruction.

As noted in my review on [policy: enable full-rbf by default](https://github.com/bitcoin/bitcoin/pull/30493#issuecomment-2263336312), enabling `mempoolfullrbf=1` on node `erin` significantly reduced the compact block reconstructions where the node had to request transactions. This seems to indicate that most pools now have policy different from the default Bitcoin Core policy. Turning on `mempoolfullrbf` by default could improve block reconstruction and thus reduce block propagation time across the network.    

### block reconstruction duration

I was curious how long it takes to reconstruct a block when we have to request transactions and when we don't have to. Note that these nodes are located in well connected data centers with reasonably good hardware, YMMV. 

I looked at the timings of block reconstructions in high-bandwidth mode. In low-bandwidth mode, Bitcoin Core [does not log](https://github.com/bitcoin/bitcoin/blob/8e1bd17252e660ea3f515673a6d330d41c8a96d7/src/net_processing.cpp#L4763-L4766) `new cmpctblock header`. I took the `new cmpctblock header` log timestamp (with `logtimemicros=1`) as start time and the `Successfully reconstructed block` timestamp as end time.

When no transactions need to be requested, median block reconstruction time on my hardware is usually around 15ms and about 130ms when transactions need to be requested.

![image|690x398](upload://aLXb4Wk1EnYbppGIOoohgyVXqKP.png)
<sub>note the different y-axis!<sub>

### further questions

I had the impression that low-bandwidth block reconstructions more often needed to request extra transactions compared to high-bandwidth reconstructions. I'm not sure if that would be expected. I think I have the data to look into it. Additionally, the share of low- vs high-bandwidth reconstructions over time would be interesting.

How do different extra pool sizes affect block reconstruction? How was the block reconstruction performance before miners started to enable full-rbf? Does anyone have historic `debug=cmpctblock` debug.logs for analysis? Also, all my (current) nodes have inbound connections. Looking at an outbound-only node in comparison could also be interesting.

-------------------------

0xB10C | 2024-08-04 15:45:21 UTC | #2

[quote="0xB10C, post:1, topic:1052"]
I had the impression that low-bandwidth block reconstructions more often needed to request extra transactions compared to high-bandwidth reconstructions. I’m not sure if that would be expected. I think I have the data to look into it. Additionally, the share of low- vs high-bandwidth reconstructions over time would be interesting.
[/quote]

About 75% of compact blocks are delivered in high-bandwidth mode (peer sends us a `cmpctblock` message before they have validated the block). The remaining ~25% are delivered in low-bandwidth mode (peer sends us a `inv`/`headers` and we request with a `getdata(compactblock)`).

![image|690x438](upload://u5Hgl8wvzSTOrQ4xNb3Lv0PhFE9.png)

Compact blocks received via high-bandwidth mode request transactions less often than (which is better) than compact blocks received in low-bandwidth mode.

I've noticed that nearly all compact blocks received have only a single transaction (the coinbase) pre-filled. As far as I understand, compact blocks delivered in low-bandwidth mode are fully validated before being announced (via `inv`/`headers`) and sender could pre-fill the transactions it didn't know about itself. This might reduce the number of low-bandwidth compact blocks that require a transaction request. I've yet to check the Bitcoin Core implementation and see if there's a reason why isn't currently being done. 

![image|690x438](upload://v6pNHEVIrM8dWrjRdyp50G01DvK.jpeg)

Edit: 

It's still a TODO. https://github.com/bitcoin/bitcoin/blob/2aff9a36c352640a263e8b5de469710f7e80eb54/src/blockencodings.cpp#L24-L25

It might be good to recheck these numbers once Bitcoin Core v28.0 (with full-RBF by default, if merged) is being adopted by the network. If by then, the low-bandwidth mode still has similarly performance, it might make sense to spend some time on implementing this.

-------------------------

Crypt-iQ | 2024-08-06 13:41:52 UTC | #3

[quote="0xB10C, post:1, topic:1052"]
How do different extra pool sizes affect block reconstruction?
[/quote]

By this do you mean the extra transactions we keep around for compact block reconstruction or do you mean differing mempool sizes? If it's the latter, it might be worth it to track your peers' `feefilter` as that can give some indication of how large their pool size is. Note that it's not precise because of the way the filter is calculated with a half-life, and because certain mempool maps might differ in memory usage. There is also some rounding in the result as well for privacy reasons.

-------------------------

0xB10C | 2024-08-08 10:17:20 UTC | #4

[quote="Crypt-iQ, post:3, topic:1052"]
By this do you mean the extra transactions we keep around for compact block reconstruction or do you mean differing mempool sizes?
[/quote]

I specifically mean the extra pool Bitcoin Core keeps for compact block reconstruction. Different node mempool sizes and different peer mempool sizes might have an effect too, but I feel this is better simulated with e.g. Warnet than measured on mainnet.

```console
$ man bitcoind | grep blockreconstructionextratxn -A 2
-blockreconstructionextratxn=<n>
  Extra transactions to keep in memory for compact block reconstructions (default: 100)
```

-------------------------

0xB10C | 2025-01-19 11:03:42 UTC | #5

Here is an updated graph of share of block reconstructions that didn’t need to request transactions - higher is better.

![image|141x500, 100%](upload://zAST1Hc8viBRGS4k3DhGq3mwfnn.jpeg)

When the mempool grows (below shows everything above 3 sat/vByte), we usually need to request transactions for more block reconstructions. A few examples are:

- between 2024-11-08 and 2024-11-15
- between 2024-12-04 and 2024-12-17

![image|690x198](upload://szTUIVWV1vZ9TrQYg35kKy2k7DH.png)

-------------------------

sipa | 2025-01-19 13:26:01 UTC | #6

Interesting, do you have any insight in what causes this? Inconsistent eviction from nodes, or more out-of-band relayed/prioritized transactions?

-------------------------

ismaelsadeeq | 2025-01-19 15:21:50 UTC | #7

One possible way to get this insight is to target blocks with low numbers to check how many of the requested transactions are non-standard. If they are much in all blocks with low numbers, it is most likely due to out-of-band prioritized transactions causing the issue; otherwise, it maybe inconsistent eviction.

-------------------------

0xB10C | 2025-01-20 10:51:08 UTC | #8

I have been running some of my nodes with this patch https://github.com/0xB10C/bitcoin/commit/90181a45651fdebc7cc5dc5262416f0f83fa4c48 and have the logs for them. Here is an example: https://gist.github.com/0xB10C/bc24f1406443faf92fe30efd16555327

Don't really have an idea yet how to automatically determine why my node didn't know about these transactions yet/anymore, but it probably makes sense to look at:

- feerate (low-feerate evicted?)
- age (exired?)
- non-standard?
- does the txid appear anywhere else in the logs?

---

I grepped for a few randomly picked txids:

```log
2024-12-07T00:21:12.376791Z [msghand] [txorphanage.cpp:75] [EraseTx] [txpackages]    removed orphan tx dd869c31dfafba5daec91a772f0b74b505245a31a49d9ad15e61fc0f689581e2 (wtxid=ee1b55a6a2ec26997f1a52c3156a2fa20cf686ee9f887d909d9d017b57a58ef1) after 1240s
2024-12-07T00:31:58.796780Z [msghand] [blockencodings.cpp:222] [FillBlock] [cmpctblock] Reconstructed block 00000000000000000000ce6d3b8a652dcea3b7d83cbaeb5831bc599aa7356618 required tx dd869c31dfafba5daec91a772f0b74b505245a31a49d9ad15e61fc0f689581e2
```


```log
2024-12-07T00:20:59.602944Z [msghand] [net_processing.cpp:2977] [ProcessInvalidTx] [mempoolrej] 0b77bbf7b1ed040b96f59d4db84cbacf6a65bcc04e8f6952e39059960b1155ea (wtxid=d9fbb98ab4d6497704b43aa6b9a86962745c36a0e74d74c19c075a830d2d4d46) from peer=964 was not accepted: bad-txns-inputs-missingorspent
2024-12-07T00:20:59.603119Z [msghand] [txorphanage.cpp:43] [AddTx] [txpackages] stored orphan tx 0b77bbf7b1ed040b96f59d4db84cbacf6a65bcc04e8f6952e39059960b1155ea (wtxid=d9fbb98ab4d6497704b43aa6b9a86962745c36a0e74d74c19c075a830d2d4d46), weight: 798 (mapsz 78 outsz 164)
2024-12-07T00:21:03.668277Z [msghand] [net_processing.cpp:4276] [ProcessMessage] [txpackages] package evaluation for parent 59cca4937290bf650b84b4ca015d07d57eed8114054ff962870b1efcabfa8c8f (wtxid=afa16ed76c1f1f871aa2f8af6fc38cf8da0a0cf1ded0903037b73b82e41341d3, sender=964) + child 0b77bbf7b1ed040b96f59d4db84cbacf6a65bcc04e8f6952e39059960b1155ea (wtxid=d9fbb98ab4d6497704b43aa6b9a86962745c36a0e74d74c19c075a830d2d4d46, sender=964): package rejected
2024-12-07T00:25:31.420207Z [msghand] [txorphanage.cpp:75] [EraseTx] [txpackages]    removed orphan tx 0b77bbf7b1ed040b96f59d4db84cbacf6a65bcc04e8f6952e39059960b1155ea (wtxid=d9fbb98ab4d6497704b43aa6b9a86962745c36a0e74d74c19c075a830d2d4d46) after 272s
2024-12-07T00:32:58.242773Z [msghand] [blockencodings.cpp:222] [FillBlock] [cmpctblock] Reconstructed block 0000000000000000000182f6bf1083700dfba1a3d5644649ed004a3d0e4bacec required tx 0b77bbf7b1ed040b96f59d4db84cbacf6a65bcc04e8f6952e39059960b1155ea
```


```
2024-12-07T01:07:17.535310Z [msghand] [node/txdownloadman_impl.cpp:218] [GetRequestsToSend] [net] Requesting tx b5d8f7a4b4988e3bfe3aa0608d9ff2eae681b9ddff4b9dd107d2103f8e7b9e55 peer=12433
2024-12-07T01:07:17.622901Z [msghand] [net_processing.cpp:2977] [ProcessInvalidTx] [mempoolrej] b5d8f7a4b4988e3bfe3aa0608d9ff2eae681b9ddff4b9dd107d2103f8e7b9e55 (wtxid=08d230a6818d5264dc0fd525cc5f7cfb01311b7870367553b96e0c30d8ac6de1) from peer=12433 was not accepted: bad-txns-inputs-missingorspent
2024-12-07T01:07:17.623050Z [msghand] [txorphanage.cpp:43] [AddTx] [txpackages] stored orphan tx b5d8f7a4b4988e3bfe3aa0608d9ff2eae681b9ddff4b9dd107d2103f8e7b9e55 (wtxid=08d230a6818d5264dc0fd525cc5f7cfb01311b7870367553b96e0c30d8ac6de1), weight: 840 (mapsz 86 outsz 227)
2024-12-07T01:17:35.855476Z [msghand] [blockencodings.cpp:222] [FillBlock] [cmpctblock] Reconstructed block 00000000000000000001a7e2ed2922731cf8b7cee9dac340818b4b8432e5d5d6 required tx b5d8f7a4b4988e3bfe3aa0608d9ff2eae681b9ddff4b9dd107d2103f8e7b9e55
2024-12-07T01:17:37.815985Z [scheduler] [txorphanage.cpp:75] [EraseTx] [txpackages]    removed orphan tx b5d8f7a4b4988e3bfe3aa0608d9ff2eae681b9ddff4b9dd107d2103f8e7b9e55 (wtxid=08d230a6818d5264dc0fd525cc5f7cfb01311b7870367553b96e0c30d8ac6de1) after 620s
```

```
2024-12-07T01:25:34.454162Z [msghand] [net_processing.cpp:2977] [ProcessInvalidTx] [mempoolrej] 1d20e245f0f18e5f0978c2cd53bc38e36068017b6dcfc5d7bbfb1490fdf2d903 (wtxid=243602b97786ee30a35a407e32f24dc1b6e286a8ac105d5258cf48f8328e2e07) from peer=37 was not accepted: bad-txns-inputs-missingorspent
2024-12-07T01:25:34.454350Z [msghand] [txorphanage.cpp:43] [AddTx] [txpackages] stored orphan tx 1d20e245f0f18e5f0978c2cd53bc38e36068017b6dcfc5d7bbfb1490fdf2d903 (wtxid=243602b97786ee30a35a407e32f24dc1b6e286a8ac105d5258cf48f8328e2e07), weight: 798 (mapsz 72 outsz 180)
2024-12-07T01:25:36.477590Z [msghand] [net_processing.cpp:4276] [ProcessMessage] [txpackages] package evaluation for parent f8d9aa002166e96cebb52b112ae15fb40a02b26247b05307a8ee9dd209e98e09 (wtxid=f20440a4b8a9a3907aeb74c6dd02a0e41761044564939999845730a74c8e732a, sender=37) + child 1d20e245f0f18e5f0978c2cd53bc38e36068017b6dcfc5d7bbfb1490fdf2d903 (wtxid=243602b97786ee30a35a407e32f24dc1b6e286a8ac105d5258cf48f8328e2e07, sender=37): package rejected
2024-12-07T01:47:46.164651Z [msghand] [blockencodings.cpp:222] [FillBlock] [cmpctblock] Reconstructed block 000000000000000000025423e700b6520db875948468bc4c1d9e8c0ec12e7b69 required tx 1d20e245f0f18e5f0978c2cd53bc38e36068017b6dcfc5d7bbfb1490fdf2d903
2024-12-07T01:47:47.707790Z [scheduler] [txorphanage.cpp:75] [EraseTx] [txpackages]    removed orphan tx 1d20e245f0f18e5f0978c2cd53bc38e36068017b6dcfc5d7bbfb1490fdf2d903 (wtxid=243602b97786ee30a35a407e32f24dc1b6e286a8ac105d5258cf48f8328e2e07) after 1333s
```

These all seem to be orphans - It seems to me like they weren't in [`vExtraTxnForCompact`](https://github.com/bitcoin/bitcoin/blob/89720b7a1b37af885f304350fa25f2479179ae3e/src/net_processing.cpp#L952C34-L952C53) (anymore). Two ideas:

- run a node with a larger `-blockreconstructionextratxn` than the default of 100 transactions to check if these perform significantly better
- look for the transactions in the orphan pool too

-------------------------

ajtowns | 2025-01-20 14:43:27 UTC | #9

Doing a better job of requesting parents of orphan txs from other peers that offered us the orphan tx might help here too? See https://github.com/bitcoin/bitcoin/pull/31397 merged in master a couple of days ago.

-------------------------

jungly | 2025-01-23 08:49:12 UTC | #10

I am curious if we considered _not_ deleting transactions from the pool of received transactions? In other words, back the mempool with a db store.

As transactions get pushed out from the mempool by better fees transactions, we eject them from the mempool and store them to disk/db instead. Later we can fetch them from the db when required instead of reaching out to a peer.

This might also help with the problems that we are trying to solve using package relay of transactions. Not sure about this, but it could have an impact.

I think libbitcoin does not delete any received transactions, ever. Which avoids problems like trying to fetch them over the network if they are later required. There is no mempool there and instead the same transactions table holds all received transactions, spent or unspent. As long as they are unspent they can be used to construct next block.

In Core, the problem will be to delete txs from the mempool backup store when txs are spent. This will have a performance penalty. Maybe the libbitcoin solution of a single txs table is worth looking into again? One store, for all rcvd txs, spent or unspent and never delete them.

Regarding spam txs, we need the same defences we use against spam when protecting mempool.

-------------------------

instagibbs | 2025-01-23 14:46:08 UTC | #11

[quote="jungly, post:10, topic:1052"]
I think libbitcoin does not delete any received transactions, ever.
[/quote]

This opens up some barn-door sized DoS vectors that graduate bandwidth waste to disk filling. If you're ever out of step with miner's policies, or someone partitions miner's mempools with pins, you'll fill your disk for free. And you'd still be taking round-trips for those blocks.

It also makes the amount of disk required per time period only bounded by gossip rate limiting.

-------------------------

jungly | 2025-01-23 17:14:31 UTC | #12

[quote="instagibbs, post:11, topic:1052"]
This opens up some barn-door sized DoS vectors that graduate bandwidth waste to disk filling.
[/quote]

Yes. The assumption in that decision is that disk is cheap. I suppose you can occasionally clean up older transactions in a manner similar to how we prune right now. 

We can use a pruning policy that is similar policies for tx eviction from mempool, but much more forgiving as we have a much larger store available.

I was just curious if such a line of thinking has been debated before.

-------------------------

ajtowns | 2025-01-23 21:00:49 UTC | #13

[quote="instagibbs, post:11, topic:1052"]
This opens up some barn-door sized DoS vectors that graduate bandwidth waste to disk filling.
[/quote]

For the purposes of avoiding round trips on block relay, I think you could probably restrict that pretty easily -- only keep txs that were in the mempool within the last 20 minutes, only keep the most recent 20MB of txs. If you also limited it to txs that were near the top of the mempool (either accurately via cluster mempool, or approximately by estimating a top of mempool feerate and comparing that to ancestor feerate and keeping their ancestors) I think that could work pretty okay, and probably just be done in memory, without being persisted to disk.

-------------------------

sipa | 2025-01-23 22:37:29 UTC | #14

Isn't that pretty much what the compact block reconstruction "extra pool" is?

-------------------------

ajtowns | 2025-01-24 00:42:29 UTC | #15

Yeah, differences are:
 * limit on extra txns is 100 txns at any one time (by default), which is a fair bit smaller than 20MB
 * all evicted txns go through the extra txns pool, not just those at the top of the mempool (so it doesn't prioritise top of mempool txs)
 * the extra txns pool isn't indexed, so what's evicted isn't prioritised

(All of which only matter if there's enough replacements going on that top of mempool stuff isn't already being kept around long enough)

-------------------------

glozow | 2025-01-31 13:31:01 UTC | #16

[quote="0xB10C, post:8, topic:1052"]
These all seem to be orphans - It seems to me like they weren’t in [`vExtraTxnForCompact`](https://github.com/bitcoin/bitcoin/blob/89720b7a1b37af885f304350fa25f2479179ae3e/src/net_processing.cpp#L952C34-L952C53) (anymore). Two ideas:

* run a node with a larger `-blockreconstructionextratxn` than the default of 100 transactions to check if these perform significantly better
* look for the transactions in the orphan pool too
[/quote]

Wow, that's new for me.

(Using vextra as short for `vExtraTxnForCompact`) It seems very reasonable for vextra to be larger, or for it to only contain non-orphan transactions, and then `InitData` can look through the orphanage as well (the orphans in vextra are kind of "free" memory-wise compared to the others).

In addition to running with a larger `-blockreconstructionextratxn`, I wonder if we can try to analyze how well we use vextra. Broadly we have (1) replaced transactions (2) orphans (3) transactions that fail validation, you could have a patch that turns the buffer into pairs of `<CTransactionRef, OriginEnum>`. I wonder if one or two of these types of vextra transactions is usually more useful than the others. (More granular, are there certain `TxValidationResult`s that are particularly helpful/unhelpful? I think `TX_CONSENSUS` is skippable. I expect this is inconsequential though.)

If most of the reconstructions are orphans, then perhaps we should consider iterating through the whole orphanage in addition to the vextra buffer. fwiw I'm also currently working on a PR to that would make the number of transactions in orphanage potentially much larger, though (in the tens of thousands). So that might be a consideration if we're thinking about doing this.

-------------------------

instagibbs | 2025-01-31 20:45:26 UTC | #17

[quote="glozow, post:16, topic:1052"]
consider iterating through the whole orphanage
[/quote]

Didn't immediately realize why we need to iterate through whole thing so saying it out loud in case others weren't aware: it's because the short IDs aren't known until one gets the compact block itself, so it cannot be precomputed.

-------------------------

ajtowns | 2025-02-02 03:30:20 UTC | #18

[quote="glozow, post:16, topic:1052"]
If most of the reconstructions are orphans, then perhaps we should consider iterating through the whole orphanage in addition to the vextra buffer.
[/quote]

I'm not sure if adding orphans would do much to help avoid round trips -- you're missing the parent and the parent will need to be in the block, after all.

One thing that could be worth looking into is doing a better job of populating prefilledtxns -- ie, "I'm pretty sure my peers don't know about these txs in the block, so I'll send them straight away". Though maybe it would be more useful to just spend that time on bringing the FIBRE patch set up to date, and having some public servers running again.

-------------------------

instagibbs | 2025-02-03 14:16:02 UTC | #19

[quote="ajtowns, post:18, topic:1052"]
Though maybe it would be more useful to just spend that time on bringing the FIBRE patch set up to date, and having some public servers running again.
[/quote]

This was the end conclusion of my offline discussion with a few folks back when I was thinking harder about this. I just saw so many different reasons for misses that I felt anything that isn't a relatively simple improvement wouldn't be worth the lift. Second look at a p2p version of FIBRE?

-------------------------

ajtowns | 2025-02-04 07:44:49 UTC | #20

The FIBRE code is [AGPL licensed](https://github.com/bitcoinfibre/bitcoinfibre/blob/65e12faf7c5516757476b9b9f227b9bd497ab8cb/src/fec.cpp) so would presumably need a rewrite/relicense to be included in bitcoin p2p.

Not sure what it would look like in a decentralised model; maybe just the ability to select some peer(s) as ultra-high-bandwidth compact block relay, and have them stream an FEC-encoded version of the block contents at you immediately after the compact block announcement? Not sure if it would make sense to have that open to random peers, or just restricted to whitelisted peers.

-------------------------

0xB10C | 2025-03-24 18:01:57 UTC | #21

[quote="ajtowns, post:18, topic:1052"]
One thing that could be worth looking into is doing a better job of populating prefilledtxns – ie, “I’m pretty sure my peers don’t know about these txs in the block, so I’ll send them straight away”.
[/quote]

I've started recording the contents of inbound and outbound `getblocktxn` messages a week ago. This should allow for some insights into "are peers often missing the same transactions?" and "can we pre-fill the transactions we had to request our self?". I haven't taken a closer look at the data yet.

Also, I've changed one of my nodes to run with `blockreconstructionextratxn=10000` and updated two nodes to a `master` that includes [p2p: track and use all potential peers for orphan resolution #31397](https://github.com/bitcoin/bitcoin/pull/31397). Probably need to wait until the mempool fills up again to see the effects of this.

-------------------------

ajtowns | 2025-02-05 04:51:21 UTC | #22

[quote="ajtowns, post:20, topic:1052"]
Not sure what it would look like in a decentralised model;
[/quote]

One other thing is that FIBRE is designed for UDP transmission to avoid delays due to retransmits; so redoing it over TCP via the existing p2p network would be a pretty big loss...

-------------------------

0xB10C | 2025-03-24 19:26:58 UTC | #24

[quote="0xB10C, post:21, topic:1052"]
Also, I’ve changed one of my nodes to run with `blockreconstructionextratxn=10000` and updated two nodes to a `master` that includes [p2p: track and use all potential peers for orphan resolution #31397](https://github.com/bitcoin/bitcoin/pull/31397). Probably need to wait until the mempool fills up again to see the effects of this.
[/quote]

- I've changed node `alice` to run with `blockreconstructionextratxn=10000` early February. This had a noticeable effect the following days with slightly higher scores starting 2025-02-06. During the increased mempool activity between 2025-02-21 and 2025-03-06 it performed significantly better than my other nodes.
- Node `charlie` and node `erin` were switched to a branch that includes [p2p: track and use all potential peers for orphan resolution #31397](https://github.com/bitcoin/bitcoin/pull/31397) at the same time in early February. I don't see any immediate improvement for these two nodes.
- Node `ian` was running Bitcoin Core `v26.1` until I switched all nodes to run `v29.0rc1` release candidate. `ian` clearly performed worse than the other nodes before the update, which is expected as e.g. `mempoolfullrbf` wasn't default in `v26.1`.
- Node `mike` doesn't allow inbound connections (while the other nodes do and usually have full inbound slots). This is noticeable in the reconstruction performance. Only having eight peers that inform `mike` about transactions is probably likely worse than having close to 100 peers that inform you about new transactions.

The stats from `alice`, `charlie`, and `erin` could indicate that orphans aren't the problem, but conflicts, replacements, and policy invalid transactions (i.e. extra pool txns) cause low performance during high mempool activity. Although, I'm not sure if these three moths of data are enough to be certain yet.

![image](upload://8CSj419oYeckv0IIiXCsKbPbnIy.png)

[quote="0xB10C, post:21, topic:1052"]
[quote="ajtowns, post:18, topic:1052"]
One thing that could be worth looking into is doing a better job of populating prefilledtxns – ie, “I’m pretty sure my peers don’t know about these txs in the block, so I’ll send them straight away”.
[/quote]

I’ve started recording the contents of inbound and outbound `getblocktxn` messages a week ago. This should allow for some insights into “are peers often missing the same transactions?” and “can we pre-fill the transactions we had to request our self?”. I haven’t taken a closer look at the data yet.
[/quote]

I've started to look at the data I've been recording. It seems that many of my peers I announced a compact block to end up requesting very different sets of transactions (and usually larger sets) than I request from my peers. I assume many of them might be non-listening nodes like `mike` or run with a non-default configuration. This needs more work, but I hope to post more stats on requested transactions here at some point.

I've also noticed that my listening nodes running with the default configuration often independently request similar sets of transactions. This seems promising in regards to predictably prefilling transactions in our compact block announcements. My assumption would be that if we prefill:
- transactions we had to request
- transactions we took from our extra pool
- and prefilled transactions we didn't have in our mempool (i.e. prefilled txns that were announced to use and ended up being useful)

we can improve the propagation time among nodes that accept inbound connections and use a "Bitcoin Core" default policy. This in turn should improve block propagation time of the complete network as now more nodes know about the block earlier. Additionally, useful prefilled transactions don't end up wasting bandwidth, only transactions that a peer already knew about would waste bandwidth. These improvements would probably be most noticeable during high mempool activity: the (main) goal wouldn't be to bring the days with 93% (of reconstructions not needing to request a transaction) to 98% but rather the days with 45% to something like 90% for well-connected nodes. 

Since Bitcoin Core only high-bandwidth/fast announces compact blocks to peers that specifically requested it from us (because we quickly gave them new blocks in the past), non-listening nodes that are badly connected won't start sending wasteful announcements with many prefilled, well-known transactions to their peers.        

I've started implementing this in [2025-03-prefill-compactblocks](https://github.com/0xB10C/bitcoin/commits/2025-03-prefill-compactblocks/) but its still work-in-progress:
- limit the prefill amount to something like 10kB worth of transactions as per [BIP152 implementation note #5](https://github.com/bitcoin/bips/blob/02ad0e01c2a9189124e05a52afe97ef90a3b7f1f/bip-0152.mediawiki#implementation-notes). I think this is useful to avoid wasting too much bandwidth if a node does a high-bandwidth announcement but, for some reason, prefills a lot of well-known transactions in the announcement
- `cmpctblock` debug logging on wasted bandwidth: log the number of bytes of transactions we already knew about when receiving a prefilled compact block. This can be tracked/monitored to determine if were wasting too much bandwidth by prefilling
- since the positive effect on the network is only measurable with a wide(r) deployment of the prefilling patch, it's probably worthwhile to do some Warnet simulations on this and test the improvement under different scenarios.

-------------------------

andrewtoth | 2025-04-08 13:50:16 UTC | #25

[quote="0xB10C, post:24, topic:1052"]
My assumption would be that if we prefill:

* transactions we had to request
* transactions we took from our extra pool
[/quote]
 
I'm not sure I understand why we would want to also prefill txs from our extra pool. The logic for extra pool inclusion would be the same for all nodes. So if we consider that our peers would have the same txs in their mempool then logically we would consider that our peers would have the same txs in their extra pool, no?

-------------------------

0xB10C | 2025-04-08 16:06:52 UTC | #26

Yeah, good question. I don't have data on this yet, but I think it makes sense to look at the extra_pools of nodes and see if they are similar or different. My assumption is that they aren't too similar. 

> So if we consider that our peers would have the same txs in their mempool then logically we would consider that our peers would have the same txs in their extra pool, no?

A few arguments against extra pool similarity are:
- the extra pool is quite small with only 100 transactions in it by default
- mempool transactions are relayed with the hope that mempools converge, extra pool transactions are stopped at their first hop and aren't relayed
- you might have peer that is sending a lot of transactions you'll reject and put into your extra pool, but I might not have a connection to this peer - our extra pools will be quite different

-------------------------

andrewtoth | 2025-04-08 16:43:45 UTC | #27


[quote="0xB10C, post:26, topic:1052"]
the extra pool is quite small with only 100 transactions in it by default
[/quote]

But our peers will also have the same default.

[quote="0xB10C, post:26, topic:1052"]
mempool transactions are relayed with the hope that mempools converge, extra pool transactions are stopped at their first hop and aren’t relayed
[/quote]
RBF replaced txs are put into the extra pool, and the replacing tx is still relayed. So they should converge. If we are going to search the orphanage anyways, we can stop putting orphans into the extra pool.

[quote="0xB10C, post:26, topic:1052"]
you might have peer that is sending a lot of transactions you’ll reject and put into your extra pool, but I might not have a connection to this peer - our extra pools will be quite different
[/quote]
Would it be likely a miner will mine these rejected txs though? Not sure.

-------------------------

