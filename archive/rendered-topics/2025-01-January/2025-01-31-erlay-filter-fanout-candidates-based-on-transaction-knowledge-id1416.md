# Erlay: Filter fanout candidates based on transaction knowledge

sr-gi | 2025-01-31 21:19:46 UTC | #1

This post is part of the Erlay implementation experiments. See https://delvingbitcoin.org/t/erlay-overview-and-current-approach/1415 for context.

# Thesis

When selecting what peers we should be fanning out transactions to, a question arises for whether peers who already know about it should also be considered. In none of the cases the transaction should be announced more than once, but, in theory, the peer selection strategy affects how the fanout rate evolves with respect to the transaction propagation rate.

## Filtering nodes that already know the transaction

Filtering out nodes based on the knowledge of the transaction should have the effect that the fanout rate stays high when the propagation of the transaction is low, and it should decrease gradually when the transaction reaches a bigger chunk of the network. This is because, as long as the number of peers that don’t know about the transaction is smaller than our fanout target rate, the rate would remain constant, and once the number grows too small, the remaining peers will be picked.

## Not filtering based on transaction knowledge

Not filtering peers should have a less smooth effect. It means that, on average, nodes should be able to reach the target fanout rate when not many of their peers know about the transaction, and hit mostly peers that already know about it the further the transaction has propagated. However, some nodes could be “unlucky” for the propagation of certain transactions, hitting mostly peers that already know about it, and relaying at a fanout rate below what would be expected based on how much the transaction has propagated.

# Simulation

To simulate the difference between these two approaches, a new filtering functionality has been added to [the simulator](https://delvingbitcoin.org/t/hyperion-a-discrete-time-network-event-simulator-for-bitcoin-core/1042) that would filter fanout candidates based on their knowledge of the transaction (local knowledge, as seen by the node that makes the decision, not the global simulation knowledge). The code for this can be found at https://github.com/sr-gi/hyperion/pull/31

## Results

The main result of this simulation is that **there is no significant difference between filtering or not**, which, surprisingly, didn’t fit our thesis.

Here are the results of a network of **110000 nodes**, **10000 reachable, 100000 unreachable** with **8 outbounds per node** without filtering:

```cmd
2024-12-17T18:45:04.569Z INFO  [hyper_lib::simulator] Using user provided rng seed: 14569482637851975056
2024-12-17T18:45:04.578Z INFO  [hyper_lib::network] Creating nodes (110000: 10000 reachable, 100000 unreachable)
2024-12-17T18:45:04.721Z INFO  [hyperion] The simulation will be run 100 times and results will be averaged
2024-12-17T18:49:49.995Z INFO  [hyper_lib] Transaction reached 90% of nodes in the network in 20.593739s
2024-12-17T18:49:49.995Z INFO  [hyper_lib] Reachable nodes sent/received 219.7265/365.32852 messages (715.5123/1067.8059 bytes) (avg)
2024-12-17T18:49:49.995Z INFO  [hyper_lib] Unreachable nodes sent/received 31.987862/17.427662 messages (94.282135/59.05277 bytes) (avg)
```

The same simulation **with filtering** yields:

```cmd
2024-12-17T18:56:26.415Z INFO  [hyper_lib::simulator] Using user provided rng seed: 14569482637851975056
2024-12-17T18:56:26.422Z INFO  [hyper_lib::network] Creating nodes (110000: 10000 reachable, 100000 unreachable)
2024-12-17T18:56:26.568Z INFO  [hyperion] The simulation will be run 100 times and results will be averaged
2024-12-17T19:01:15.641Z INFO  [hyper_lib] Transaction reached 90% of nodes in the network in 20.795815s
2024-12-17T19:01:15.641Z INFO  [hyper_lib] Reachable nodes sent/received 222.6441/371.3514 messages (720.47595/1109.9666 bytes) (avg)
2024-12-17T19:01:15.641Z INFO  [hyper_lib] Unreachable nodes sent/received 32.525223/17.654491 messages (98.37036/59.421295 bytes) (avg)
```

The results are within the expected variance for simulations of the same setup. For comparison purposes, here is a re-run of the same simulation **without filtering**:

```cmd
2024-12-17T19:02:32.958Z INFO  [hyper_lib::simulator] Using user provided rng seed: 14569482637851975056
2024-12-17T19:02:32.965Z INFO  [hyper_lib::network] Creating nodes (110000: 10000 reachable, 100000 unreachable)
2024-12-17T19:02:33.115Z INFO  [hyperion] The simulation will be run 100 times and results will be averaged
2024-12-17T19:07:24.090Z INFO  [hyper_lib] Transaction reached 90% of nodes in the network in 20.801981s
2024-12-17T19:07:24.090Z INFO  [hyper_lib] Reachable nodes sent/received 222.98781/371.10538 messages (715.7427/1070.6921 bytes) (avg)
2024-12-17T19:07:24.090Z INFO  [hyper_lib] Unreachable nodes sent/received 32.490536/17.67878 messages (94.54821/59.053257 bytes) (avg)
```

And all three simulations as a `csv` for easier comparison:

```csv
timestamp,percentile-target,percentile-time,r-sent-msgs,r-received-msgs,r-sent-bytes,r-received-bytes,u-sent-msgs,u-received-msgs,u-sent-bytes,u-received-bytes,r,u,n,erlay,seed
1734461389,90,20593737728,219.7265,365.32852,715.5123,1067.8059,31.987862,17.427662,94.282135,59.05277,10000,100000,100,true,14569482637851975056
1734462075,90,20795813888,222.6441,371.3514,720.47595,1109.9666,32.525223,17.654491,98.37036,59.421295,10000,100000,100,true,14569482637851975056
1734462444,90,20801980416,222.98781,371.10538,715.7427,1070.6921,32.490536,17.67878,94.54821,59.053257,10000,100000,100,true,14569482637851975056
```

These results showed that for the given network [^1], filtering didn’t make a difference, which, based on our thesis, should make us wonder **why.** 

To answer this question, we need to check how many peers are being filtered per node on the network, and how that number ties to the propagation of the transaction. To do this, I modified the simulator on the fly [^2] to record how many peers know about a transaction when a node is deciding who to fanout to, that is, how many `INV` messages a given node has received when they are scheduling their transaction announcement. This yielded the following output:

```cmd
0,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,
...
1,1,1,3,1,1,1,1,1,3,1,1,1,1,1,1,2,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,2,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,4,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,2,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,
...
1,1,1,1,1,1,1,1,1,1,1,7,1,1,10,1,1,1,2,1,1,1,5,5,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,2,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,5,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,7,1,1,1,1,1,1,1,1,7,1,1,1,1,1,5,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,6,1,1,1,1,1,1,1,1,
...
1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,
```

Even though these are mostly a few random samples from the **110000** produced, we can see how **ones** are mostly predominant. That is, most nodes have only heard about a given transaction from the peer that sent it to them at the moment of scheduling the transaction relay. This explains why filtering was highly ineffective, but it is still a surprising finding on its own. Something we can notice if we play a bit with the simulation parameters is that the percentage of **ones** seems to be reduced if we reduce the number of unreachable nodes in the network. **But how is the number of unreachable nodes related to this phenomenon, and why?**

To understand this, we need to go deeper into how the network is structured, and how transactions are propagated. When a transaction is scheduled for announcement by a given peer, its announcement is delayed based on the type of peer it is to be announced to. [We covered this on the Erlay overview post](https://delvingbitcoin.org/t/erlay-overview-and-current-approach/1415#footnote-4127-5). This means that outbound peers are likely to receive the transaction before inbounds. Given how the network is structured, unreachable peers are **only connected** to reachable ones through outbound connections (which are inbound on the receiving end), reachable nodes, on the other hand, are connected to other reachable through outbound connections, and to both reachable and unreachable through inbound connections only. 

This means that once a transaction has reached the reachable network it is likely to propagate through the reachable network at a higher rate than it does to the unreachable one [^3]. Furthermore, for every hop in the unreachable network, the transaction **must** go to a reachable node, if any. This should explain why reachable nodes have a higher number of peers having heard of a given transaction when they are to schedule its announcement, but it does not explain why for unreachable nodes the count is almost always `1`. 

To understand why, there is another detail we should be considering: when a transaction is being announced by a peer, **if the peer is inbound**, the transaction request is **delayed by** `NONPREF_PEER_TX_DELAY{2s}`, otherwise, the request is sent straight away. This prioritizes requesting transaction from outbound peers, which should be harder to control by malicious parties. Reachable nodes are mostly connected to peers by inbound connections (they are the gateways to the network), while unreachable nodes are solely connected to peers via outbound connections. This means that reachable nodes are likely to wait for **two seconds** between receiving a transaction announcement and requesting a transaction, whereas unreachable peers will receive the transaction almost instantly (modulo link latencies).

This does explain our finding: reachable nodes, on average, have a longer delay window between a transaction is announced to them and receiving the actual transaction, time at which they will schedule their announcement. Unreachable nodes, on the other hand, always announce transactions via an outbound link, meaning their data requests will be fired straight away, reducing the delay windows to mainly the link latencies. We can verify this hypothesis by increasing/decreasing the `GETDATA` request delay and re-run the experiment:

`NONPREF_PEER_TX_DELAY{0}` 

```cmd
0,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,
...
1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,2,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,
...
1,2,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,2,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,
...
1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,
```

Reducing `NONPREF_PEER_TX_DELAY` to **zero** yields even more `1`s, almost a totality of them, which is coherent with our hypothesis: the shorter the delay windows, the least time `INV` messages have to pile between a node first hears about a transaction and the transaction is received.

`NONPREF_PEER_TX_DELAY{5}` 

```cmd
0,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,
...
1,5,1,1,1,2,1,1,7,10,6,5,1,1,5,1,1,1,1,1,7,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,6,1,1,1,1,1,1,3,1,1,1,5,8,1,1,1,1,1,1,1,1,1,6,1,1,1,2,1,1,1,1,1,1,4,1,1,1,1,1,1,1,1,1,1,1,4,1,1,1,1,1,1,1,1,1,4,1,1,1,1,1,1,1,3,1,1,1,5,1,1,3,1,1,1,1,1,1,7,1,1,1,1,8,1,1,1,1, 
...
10,1,1,1,1,1,7,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,7,1,8,1,1,1,1,3,1,4,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,5,1,1,1,1,1,1,9,1,1,1,1,6,7,1,1,1,1,6,3,1,1,1,1,1,1,1,1,1,1,1,1,1,7,1,1,7,1,1,1,1,1,10,1,1,8,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,8,1,1,1,1,1,1,1,1,1,1,
...
1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,
```

Increasing `NONPREF_PEER_TX_DELAY` to **five** yields higher values for the reachable nodes, which is also consistent with our hypothesis.

A way to see this even more clearly is to make both inbound and outbound equally subjected to delays. This needs minimal code changes (there is no commit for this) and, for `NONPREF_PEER_TX_DELAY{5}` yields:

```cmd
0,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,
...
82,93,1,1,1,1,2,1,1,1,103,1,1,1,1,1,90,1,1,1,93,1,1,1,1,1,1,1,1,1,2,1,1,1,86,1,1,1,1,1,1,107,1,1,1,1,109,87,1,1,1,1,84,1,1,2,82,1,1,2,1,1,1,1,2,1,1,87,94,1,1,2,1,1,1,1,1,1,1,1,1,1,1,1,107,1,1,1,2,83,1,1,1,1,1,1,81,1,1,1,1,1,2,1,1,2,1,90,1,1,104,86,1,1,2,1,1,1,91,100,1,1,1,81,1,1,1,1,
...
4,3,4,3,2,4,2,1,3,4,3,1,2,2,4,3,5,4,2,1,2,4,4,3,4,4,1,5,1,5,3,2,4,4,2,3,3,4,2,2,5,3,3,3,2,2,2,6,3,5,3,2,2,2,3,1,3,3,2,4,4,5,2,2,4,2,3,3,4,5,4,4,3,4,1,3,3,3,2,4,2,5,2,3,4,5,4,4,2,3,2,3,2,3,2,2,6,3,5,5,5,5,2,2,1,5,2,3,3,3,3,3,2,4,3,3,2,3,5,1,2,2,3,3,3,2,2,1,3,3,2,2,3,3,3,3,3,4,3,2,1,2,
...
5,6,5,5,5,5,6,5,6,5,5,5,6,5,6,5,5,5,6,5,6,6,5,4,5,4,6,5,5,5,6,5,5,6,5,6,5,5,5,5,6,6,5,5,5,4,4,5,5,6,6,6,6,5,6,6,5,4,6,5,5,6,6,5,6,3,5,4,5,7,6,3,5,5,7,6,5,5,6,6,4,4,6,6,6,5,4,5,5,6,5,5,6,6,4,4,5,6,5,6,6,5,4,5,5,4,5,4,5,5,6,7,3,5,7,5,6,4,4,6,5,3,6,6,6,3,5,4,5,5,5,6,6,5,5,5,6,6,5,5,4,6,
```

# Conclusion

Given the nature of the Bitcoin network topology, and how transaction announcements from inbound connections are de-prioritized, most nodes in the network would hear from a small subset of their peers about a transaction before they schedule their own announcement. Hence, **filtering fanout selection based on the peer knowledge of the transaction is mostly irrelevant.**

[^1]: Network topologies for our simulator are created at random, making sure that all nodes initiate 8 outgoing connections to reachable nodes, and that connections are not duplicates. The Bitcoin network should follow a similar, but it is “impossible” to tell
[^2]: The code can be run from https://github.com/sr-gi/hyperion/commit/500da78463fa8bbf22b8ecee978caff21c82a35c, a non-erlay simulation running on default settings is enough for this
[^3]:  Notice the transaction would take at most one hop to reach the reachable network

-------------------------

