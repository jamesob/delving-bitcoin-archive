# Erlay: Define fanout rate based on the transaction reception method

sr-gi | 2025-02-05 20:48:35 UTC | #1

This post is part of the Erlay implementation experiments. See [Erlay: Overview and current approach](https://delvingbitcoin.org/t/erlay-overview-and-current-approach/1415) for context.

# Thesis

For optimal network bandwidth utilization when relaying transactions, the fanout rate should be higher the earlier in the transaction propagation state, and lower the later. This is due to **fanout being more efficient (and faster) than transaction reconciliation provided the receiving node doesn’t know about the transaction.** This means that, in a perfect world, we could start with a high fanout rate to reach a high portion of the network (with minimal redundancy), and slowly scale the fanout down to relay more on set reconciliation when the transaction has spread enough.

Unfortunately, the receiver node does not possess precise knowledge of how far a transaction has propagated along the network. We have previously tried to use the number of announcements a node has received about a given transaction as a proxy for this. Still, it has turned out to work mostly for **reachable** nodes, but not as much for **unreachable** (see https://delvingbitcoin.org/t/erlay-filter-fanout-candidates-based-on-transaction-knowledge/1416/3).

An alternative approach is checking under what method the transaction was received as part of the decision-making to propagate the transaction further. That is, have a fanout rate if the transaction was received via fanout, and a different one if the transaction was received via set reconciliation. Being fanout significantly faster than set reconciliation, it would be expected that the first nodes to learn about a given transaction would do so through fanout, **so having a higher propagation rate if the transaction was received via fanout, and a smaller one if it was received via set reconciliation** may be a good enough approximation to our ideal solution.

# Simulation

To simulate this, we’ll be using a patched version of [the simulator](https://delvingbitcoin.org/t/hyperion-a-discrete-time-network-event-simulator-for-bitcoin-core/1042) that will scale the fanout rate based on how a node received the target transaction. The code can be found at https://github.com/sr-gi/hyperion/commit/a0f11424dac30a2bebc22cd534f98a2d9de36cf8.

The simulation will be built on the following two assumptions:

- The fanout rate when receiving a transaction via fanout needs to be higher, but not too high. This is to prevent too much redundancy, given some nodes will still be receiving the transaction via fanout even by the end of the transaction propagation.
- We can scale up the number of **outbound** connections to fanout to, but the number of **inbounds must remain constant.** This is to prevent malicious peers from knowing what fanout rate we used by simply establishing multiple connections to our node.

To run the experiment we simply need to define the corresponding env variables representing the low and high fanout targets: `OUTBOUND_FANOUT_DESTINATIONS` and `OUTBOUND_FANOUT_DESTINATIONS_HIGH`. An example would be:

```bash
OUTBOUND_FANOUT_DESTINATIONS=1 OUTBOUND_FANOUT_DESTINATIONS_HIGH=4 cargo run --release --erlay -n 100
```

As with all our previous simulations, we will create a network of **110000 nodes**, **10000 reachable, 100000 unreachable.** In this case, we will run 3 different simulations, corresponding to **8, 10, and 12** outbound peers respectively, as we previously did for https://delvingbitcoin.org/t/erlay-find-acceptable-target-number-of-peers-to-fanout-to/1420. Every simulation will be run **100** times, with the same random network, picking a different source every time, and the results will be averaged.

The results will be compared with our baseline (same network, no Erlay), as well as with the setting for the currently implemented approach: **1 outbound and 10% of inbounds**.

## Result

We run simulations for the three following cases (on top of our reference points)[^1]:

- 1/10% + 4/10%
- 1/10% + 6/10%
- 1/20% + 4/20%

The leftmost side of the expression represents the fanout rate when receiving a transaction **via reconciliation**, whereas the rightmost accounts for receiving it **via fanout**. For instance, `a/b% + c/d%` means `a` outbound and `b%` of inbounds if received via reconciliation, `c` outbound and `d%` of inbounds if received via fanout.

The most noticeable result is the reduction in transaction latency compared to **1/10%**. This is expected, as even in the most conservative approach (**green line: 1/10% + 4/10%**), the lowest rate already matches the reference, preventing any increase in propagation time. The improvements range from **~18% speedup** in the most conservative case to **~24%** in the other two cases.

![Propagation time (90%)|684x424](upload://kPq6YfhqBDDxpeDXKqNoHLXDNhg.png)

However, this doesn’t provide much context on its own. As we previously saw when running our wide-range search experiment, Erlay presents a tradeoff between latency and bandwidth. So to evaluate these results, we should also check how bandwidth has been affected. 

![Data volume per tx (reachable nodes)|684x424](upload://8I6s1HD04w4uRLMcxXXLgAMn91J.png)

For reachable nodes, the data volume per transaction has increased between ~6.5% (green line, 1/10% + 4/10%) to ~13% (purple line 1/20% + 4/20%).

![Data volume per tx (unreachable nodes)|684x424](upload://f9qvZOsfZ5O7twgwlgVZrATfasA.png)

For unreachable, the data volume has increased between ~4% to ~11%.

Both for reachable and unreachable cases, the lower end of this range is the most compelling. It not only results in a smaller bandwidth increase but also achieves a higher relative speedup. Specifically, **1/10% + 4/10%** delivers an **18% speedup** with only a **6.5% increase in bandwidth utilization**. In contrast, **1/20% + 4/20%** reduces latency further (**~25% faster**) but comes at a higher cost—**double the bandwidth increase for reachable nodes** and **nearly 3× for unreachable nodes** compared to **1/10% + 4/10%**.

You may have noticed an additional yellow line in the plots, representing **4/10%**. This serves as a reference to compare the impact of the variable approach against the fixed approach. The propagation times for **4/10%** are akin to **1/10% + 4/10%** (they are barely 3% faster), however, the bandwidth utilization is 13%-15% worse than for **1/10% + 4/10%** 

# Conclusion

Given the slower nature of the transaction reconciliation protocol, the method by which a node receives a transaction (fanout or reconciliation) serves as a strong indicator of how far the transaction has propagated. This insight can be used in the propagation decision-making to adjust the fanout rate: **increasing it when the transaction is closer to the source** and **reducing it as it spreads further**. By making the transaction quickly available to a big subset of nodes in the network, meaningful reconciliation becomes available sooner, at which point the fanout rate will be compensated down so set reconciliation can take care of fixing the remaining differences.

A rather simple approach to this has shown to be useful, substantially reducing the propagation latency in one of the most bandwidth-conservative settings, while maintaining most of the bandwidth savings. Given these results, my opinion is that **1/10% + 4/10% could be used as a more performant drop-in replacement** of the current 1/10% setup.

[^1]: The data for the experiment, as well as all charts shown in this document, can be found at https://docs.google.com/spreadsheets/d/1t7bVL88FTx3vIE_gLeda0udfGps1ypTSyRhokh2yCIA/edit?usp=sharing

-------------------------

