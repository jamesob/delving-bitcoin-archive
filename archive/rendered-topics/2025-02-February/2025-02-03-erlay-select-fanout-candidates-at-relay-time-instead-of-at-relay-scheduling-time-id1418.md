# Erlay: Select fanout candidates at relay time instead of at relay scheduling time

sr-gi | 2025-02-03 15:57:59 UTC | #1

This post is part of the Erlay implementation experiments. See [Erlay: Overview and current approach](https://delvingbitcoin.org/t/erlay-overview-and-current-approach/1415) for context.

# Thesis

Fanout targets are selected on a transaction basis. That means, for each transaction to be relayed, a subset of our peers is selected for fanout, while the rest would be used for transaction reconciliation. The fanout candidate selection can be done at two stages of the transaction relay process:

- At relay schedule, when the transaction is received/created and it is queued for relay to transaction relaying peers
- At transaction relay, after the Poisson timer for a peer has gone off and the corresponding `INV` message is to be created

Our current implementation follows the former, since it is easier to reason about transaction dependencies during transaction scheduling, however, at relay time it should be easier to adapt to fanout candidates already knowing about the target transaction, and potentially swap candidates so the desired fanout rate can be achieved. For a more extensive explanation of both approaches, check: [Choosing fanout peers at relay scheduling time](https://delvingbitcoin.org/t/erlay-overview-and-current-approach/1415#p-4127-choosing-fanout-peers-at-relay-scheduling-time-4) and [Choosing fanout peers at transaction relay time](https://delvingbitcoin.org/t/erlay-overview-and-current-approach/1415#p-4127-choosing-fanout-peers-at-transaction-relay-time-5).

# Simulation

To simulate the difference between both approaches, a new branch has been created in the [simulator](https://delvingbitcoin.org/t/hyperion-a-discrete-time-network-event-simulator-for-bitcoin-core/1042) where the decision-making is done at transaction relay time instead of at relay scheduling time. The new branch gets rid of the `delayed_set` for Erlay peers in their `TxReconciliationState`, and all transactions are kept in `to_be_announced` until relay time, where they would be picked for fanout or reconciliation based on `get_fanout_targets`. Since this process happens on a peer basis, fanout targets are now cached, so the selection is consistent for all peers of a given node for a given transaction.

To simulate this we will use a randomly generated network containing 110000 nodes, 10000 reachable, 100000 unreachable with 8 outbounds per node, **all of them implementing Erlay.** The simulation can be run using the code at https://github.com/sr-gi/hyperion/commit/32f5af9b39a9387d7f745df7da16645e69457d9e.

## Results

Here are the results of running our simulation in our reference model, that is, a network of the aforementioned characteristics selecting fanout candidates **at scheduling time**:

```cmd
2024-12-20T22:12:59.431Z INFO  [hyper_lib::simulator] Using user provided rng seed: 5831067508548421759
2024-12-20T22:12:59.438Z INFO  [hyper_lib::network] Creating nodes (110000: 10000 reachable, 100000 unreachable)
2024-12-20T22:12:59.568Z INFO  [hyperion] The simulation will be run 100 times and results will be averaged
2024-12-20T22:17:42.069Z INFO  [hyper_lib] Transaction reached 90% of nodes in the network in 20.764025s
2024-12-20T22:17:42.069Z INFO  [hyper_lib] Transaction reached all nodes in 28.108896s
2024-12-20T22:17:42.069Z INFO  [hyper_lib] Reachable nodes sent/received 222.2216/369.73523 messages (715.59644/1069.915 bytes) (avg)
2024-12-20T22:17:42.069Z INFO  [hyper_lib] Unreachable nodes sent/received 32.36946/17.618101 messages (94.462814/59.03094 bytes) (avg)
```

And the same simulation in our updated model, selecting fanout peers **at relay time:**

```cmd
2024-12-20T22:34:03.686Z INFO  [hyper_lib::simulator] Using user provided rng seed: 5831067508548421759
2024-12-20T22:34:03.694Z INFO  [hyper_lib::network] Creating nodes (110000: 10000 reachable, 100000 unreachable)
2024-12-20T22:34:03.826Z INFO  [hyperion] The simulation will be run 100 times and results will be averaged
2024-12-20T22:39:01.915Z INFO  [hyper_lib] Transaction reached 90% of nodes in the network in 20.655256s
2024-12-20T22:39:01.915Z INFO  [hyper_lib] Transaction reached all nodes in 28.008577s
2024-12-20T22:39:01.915Z INFO  [hyper_lib] Reachable nodes sent/received 221.15518/367.86584 messages (715.5273/1068.999 bytes) (avg)
2024-12-20T22:39:01.915Z INFO  [hyper_lib] Unreachable nodes sent/received 32.20926/17.538193 messages (94.39123/59.044044 bytes) (avg)
```

Finally, here are both simulations as a `csv` for easier comparison:

```csv
timestamp,percentile-target,percentile-time,full-propagation-time,r-sent-msgs,r-received-msgs,r-sent-bytes,r-received-bytes,u-sent-msgs,u-received-msgs,u-sent-bytes,u-received-bytes,r,u,n,erlay,seed
1734733062,90,20764024832,28108896256,222.2216,369.73523,715.59644,1069.915,32.36946,17.618101,94.462814,59.03094,10000,100000,100,true,5831067508548421759
1734734341,90,20655255552,28008577024,221.15518,367.86584,715.5273,1068.999,32.20926,17.538193,94.39123,59.044044,10000,100000,100,true,583106750854842175
```

We can see that there are no meaningful differences between the two simulations, which may sound surprising. To understand why this is the case, we can follow a similar approach as per our [filter fanout candidates based on transaction knowledge simulation](https://delvingbitcoin.org/t/erlay-filter-fanout-candidates-based-on-transaction-knowledge/1416). This time we will modify the simulation slightly. Instead of just recording how many peers have announced a transaction to us by the time we need to select fanout candidates, we will be taking two samples: **the first one will be taken at transaction relay scheduling**, point at where our reference model selects fanout peers, and **the second one will be taken at transaction relay time**, when our current model selects them.

This creates an output containing a tuple `(first_sample, second_sample, is_reachable)` for all nodes where first_sample ≠ second_sample. To run this test, we can use the code at https://github.com/sr-gi/hyperion/commit/333b22f4b929449b86304da8bb9010fccafaf651, a simulation without repetition will suffice here (`-n=1`). The resulting output looks as follows:

```cmd
(1, 2, false)
(1, 2, true)
(1, 2, true)
(1, 2, true)
(1, 2, true)
(1, 2, true)
(1, 2, true)
...
(2, 3, true)
(2, 3, true)
(2, 4, true)
(3, 5, true)
(1, 2, true)
```

The most noticeable aspect of the simulation results is the number of entries in the output: only **4824** out of **110000** nodes showed a difference. In other words, **approximately 4.38% of the nodes in our simulated network** [^1]. From the sampled nodes, about 43% of them are unreachable, and almost 98% of those show an increase of a single peer between the two samples (going from a single peer, the one that shared the transaction with them, to two).

# Conclusion

This simulation shows that **there is no significant difference between selecting peers at scheduling time or relay time.** It is worth noting that this is likely derived from using a full Erlay network, and **results may vary if simulated in networks with partial support**.

We are currently simulating scenarios where **the whole network is composed of Erlay peers**, plus the amount of peers selected for fanout is rather small (1 outbound and 10% of inbounds). This means the expected number of received `INV` messages should be rather small.

[^1]: The exact value may vary between simulations, given the random nature of some of the involved processes, however, several runs with different seeds have yielded very similar results, obtaining samples of only ~4% of nodes in the network.

-------------------------

