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

