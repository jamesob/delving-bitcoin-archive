# Transaction rate-limiting

ajtowns | 2026-07-27 07:59:57 UTC | #1

Very pleased to have [#34628](https://github.com/bitcoin/bitcoin/pull/34628) merged into Bitcoin Core for the 32.0 release in the coming months.

It's (hopefully!) the conclusion to three years of (on and off) work, beginning (from my perspective) with an alert from the VPS hosting my mainnet debug-build node that it was using excessive CPU, along with log entries indicating it was taking 80 seconds to accept individual txs to the mempool:

```
2023-05-03T12:08:22Z [lock] Enter: lock contention cs_main, net_processing.cpp:1450 started
2023-05-03T12:08:49Z [lock] Enter: lock contention cs_main, net_processing.cpp:5116 started
2023-05-03T12:09:30Z [mempool] Removed 2 txn, rolling minimum fee bumped to 0.00003008 BTC/kvB
2023-05-03T12:09:30Z [mempool] AcceptToMemoryPool: peer=2550: accepted 45d87efceeb53745408114f6ad291c01349388483eaaf861a5c8f2e4cc198a99 (poolsz 65133 txn, 197357 kB) feerate=124.00
2023-05-03T12:09:30Z [lock] Enter: lock contention cs_main, net_processing.cpp:1489 completed (85,862,565us)
```

https://bitcoincore.org/en/2024/10/08/disclose-large-inv-to-send/

https://b10c.me/observations/15-inv-to-send-queue/

That all ended up being the combination of two bugs: [one](https://github.com/bitcoin/bitcoin/issues/27586) was boost implementing iterators with O(N) performance in the size of the container (!), but the other was a result of the desire to limit the negative impacts of large bursts of transactions on the network.

The idea there is that each bitcoin node acts as an amplifier during transaction relay -- you get one copy of the transaction in from a single peer, but then you send (up to) $n-1$ copies of the tx outbound, one to each of your peers that hasn't already received the transaction from someone  else. This applies both to the actual transaction data, and also to the announcement of the wtxid. As a general principle amplifiers in a network are dangerous: an attacker can exploit them to amplify a small amount of effort into a large attack. If there weren't any limits in place here, an attacker could perhaps announce 100MB of transaction data to a single peer, and let that peer relay the data to all its peers (and so on), and use up the entire network's bandwidth for however long it takes to propoagate 100MB to perhaps 80k nodes (so perhaps 8TB total), potentially delaying header and block relay, while also incurring heavy CPU usage on any nodes that are more CPU-limited than bandwidth limited.

This behaviour was introduced ten years ago, with the following description in the commit message:

> Limits the amount of transactions sent in a single INV to 7tx/sec (and twice that for outbound); this limits the harm of low fee transaction floods, gives faster relay service to higher fee transactions. The 7 sounds lower than it really is because received advertisements need not be sent, and because the aggregate rate is multipled by the number of peers.

https://github.com/bitcoin/bitcoin/pull/7840

There are a few reasons it didn't work out exactly as planned:

 * across a link, only about 24.5 txs/s can get propagated (17.5 from the node that created the connection to its outbound peer because the actual multiplier was 2.5 not 2 due to rounding, and 7 back; fewer if the peers' announcements overlap and duplicate each other), and because the transactions are sorted, that doesn't get an effective multiplier by the number of connections
 * a shortcut in the sorting comparator meant that high-feerate descendant transactions (eg, CPFP txs) would relay after low-feerate transactions that only spend confirmed inputs, increasing discrepencies between mining and relay tx choice
 * better support for replace-by-fee means the number of transactions relayed per second can be significantly larger than the number of transactions confirmed per second, especially if feerates are rising
 * between 2016 and 2023 segregated witness doubled the number of potential transactions per block, doubling the potential confirmed transactions per second
 * finally, in May 2023 saw the start of [high volumes of small text-based inscriptions](https://www.coindesk.com/tech/2023/05/03/bitcoin-ordinals-surge-to-3m-inscriptions-but-most-are-just-text) replacing large image-based inscriptions

The immediate fix at the time was twofold: (a) make sure that txs removed from the mempool weren't left around in the per-peer queues, doing nothing but slowing everything down, and (b) when there was a large backlog, increase the rate at which txs are processed so things can be cleared out.

Cluster mempool happily fixed the sorting issue.

A later followup also increased the 7tx/s target to 14tx/s:

https://github.com/bitcoin/bitcoin/pull/28592

These changes aren't entirely satisfactory though; they minimise the problems in practice, but leave the theoretical worst cases unresolved: you're still maintaining per-peer queues that can grow large, and if they do grow large the CPU cost to process them grows as $O(N^2)$ (cumulatively). @0xB10C noticed an indication of that being something of a problem in [February of this year](https://bnoc.xyz/t/increased-b-msghand-thread-utilization-due-to-runestone-transactions-on-2026-02-17/81) -- a large surge of runestone transactions caused excessive CPU usage on some of his nodes, nudging the problem back from theory and towards practice.

Happily, by that time, I'd already been banging my head against the problem for a few months, which was long enough to realise that replacing the per-peer queues with a single global queue was reasonably plausible. And while that still has $O(N)$ CPU usage every few seconds (and hence $O(N^2)$ cumulative time to actually process all $N$ transactions), it's a substantial improvement because it's not repeated for every peer.

https://github.com/bitcoin/bitcoin/pull/34628

It took a few months to refine that PR into the form that eventually got merged, but I believe what we have now should be pretty effective:

 * should keep your CPU available for validating blocks and transactions, not use it all up for resorting the same set of txs for each peer
 * should cope cleanly with large backlogs of transactions
 * should smooth out large transaction bursts, focusing on relaying the txs in the top of the mempool (per cluster mempool rules, of course)
 * introduces a new bandwidth ratelimit that should prevent excessive numbers of large transactions that never confirm from blowing out your VPS's monthly limit, even if you have a couple of hundred peers to relay those txs to

So, touch wood, that should provide significant robustness improvements for nodes dealing with high tx volume, independent of whether that volume comes from increased monetary adoption, random spammer behaviour, or some sort of deliberate attack.

-------------------------

