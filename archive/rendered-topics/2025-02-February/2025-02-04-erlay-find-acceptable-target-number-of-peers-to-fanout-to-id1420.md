# Erlay: Find acceptable target number of peers to fanout to

sr-gi | 2025-02-04 19:21:59 UTC | #1

This post is part of the Erlay implementation experiments. See [Erlay: Overview and current approach](https://delvingbitcoin.org/t/erlay-overview-and-current-approach/1415) for context.

# Thesis

For Erlay to work efficiently, some degree of transaction fanout is necessary. This ensures that a sufficient number of peers receive the transaction quickly. The key reason for this is that fanout is more bandwidth-efficient **if the receiving node has not already seen the transaction**. However, if the node already knows the transaction, fanout becomes wasteful. Additionally, Erlay introduces some latency to the transaction exchange process.

The ideal approach would be to fanout until a transaction has reached a sufficiently large portion of the network, then switch to reconciliation. However, this is impractical because a node does not know how widely a transaction has propagated at the time of reception.

A more feasible strategy is to set a minimum fanout rate and adhere to it, allowing reconciliation to handle the rest. This ensures that a targeted percentage of the network receives the transaction through fanout, while reconciliation takes care of the rest.

A viable approach is to target a minimum fanout rate and stick to it while leaving the rest to reconciliation. This way a target percentage of the network will receive the transaction via fanout, and the rest will use reconciliation for it [^1]. 

- A **higher fanout rate** means the transaction reaches a larger subset of the network more quickly, but with lower bandwidth savings.
- A **lower fanout rate** saves more bandwidth but increases latency.

# Simulation

To determine reasonable Erlay configurations, we can begin by simulating how a transaction propagates across the network using the current protocol. This will establish a baseline in terms of both data usage and propagation time. Once we have this baseline, we can run the same simulation with different Erlay configurations, each targeting different fanout levels. By analyzing the relationship between bandwidth saving and latency we can fine-tune our parameters to achieve an acceptable configuration.

We will start by defining our baseline. For this, as in previous simulations, we are using a random network of **110000 nodes**, **10000 reachable, 100000 unreachable.** Given Erlay has the additional goal of allowing nodes to increase their peer count without incurring significant bandwidth costs, we will be simulating this for **8, 10, and 12 outbound connections** per node (to do this, we can simply specify the `--outbound` option in [the simulator](https://delvingbitcoin.org/t/hyperion-a-discrete-time-network-event-simulator-for-bitcoin-core/1042)).

```cmd
cargo run --release -- -n 100 --erlay --seed 8482337119851113330 --outbound 8
cargo run --release -- -n 100 --erlay --seed 8482337119851113330 --outbound 10
cargo run --release -- -n 100 --erlay --seed 8482337119851113330 --outbound 12
```

We will compare the baseline with a wide variety of Erlay configurations for `OUTBOUND_FANOUT_DESTINATIONS` and `INBOUND_FANOUT_DESTINATIONS_FRACTION`. For outbounds, we will select values on the [0-8] range, whereas for inbounds, we will increase the count percentage-wise (in 10% increments) starting from 0. 

Given the scope of this experiment, running all the necessary simulations would take considerable time. To streamline the process, we have added a `multi-run` script to the simulator. This script automates a series of simulations, testing different ranges for `OUTBOUND_FANOUT_DESTINATIONS` and `INBOUND_FANOUT_DESTINATIONS_FRACTION`. The results are then stored in a CSV file for further analysis.

We will be running instances of the script simultaneously (one for each `--outbound` configuration) and interleave the results afterward:

```cmd
hyperion/multi-run.sh --outbounds=8 --seed=8482337119851113330 --out=result_8.csv
hyperion/multi-run.sh --outbounds=10 --seed=8482337119851113330 --out=result_10.csv
hyperion/multi-run.sh --outbounds=12 --seed=8482337119851113330 --out=result_12.csv
```

# Results

Since this is a wide-range experiment, the results are extensive and only meaningful when compared. Documenting every configuration in detail would make this report tedious to read. Instead, we will focus on three specific results to analyze how the charts change based on the selected number of peers. An interested reader can refer to https://docs.google.com/spreadsheets/d/1YGG3Lm6__xmKsd9HlmNuQOaw_Lg7rLwsBNcWYJsUsIw/edit?usp=sharing to view the complete set of results. The interleaved data is located in columns **A-Z**, while the corresponding charts can be found further to the right in the document.

## Transaction propagation time

The three following charts show the transaction propagation times to **90%** of nodes in the simulated network, for configurations of **1 outbound/[0-100%] inbounds**, **4 outbounds/[0-100%] inbounds** and **7 outbounds/[0-100%] inbounds** respectively.

![Propagation time (90%)(fanout_out_1)|684x424](upload://yKQrw2MLaBeDae5Vdim4gzRFJPt.png)
![Propagation time (90%)(fanout_out_4)|684x424](upload://mZvOpYGZExURoWvYK1ziolE5xTu.png)
![Propagation time (90%) (fanout_out_7)|684x424](upload://w7oioyCLYy6WgTkYzDac79lCGaN.png)

As it is expected, the transaction propagation times are reduced the more nodes are picked for fanout, given it is a faster propagation technique than set reconciliation. We can also observe that, independently of the configuration, the latency is reduced linearly when the node connectivity is increased (i.e. the number of outbound connections is increased).

## Data volume per transaction

For the volume of data used per transaction per node (i.e., the total amount of data required to send and receive each transaction, including announcements, reconciliation, and other overhead) we observe that a lower fanout rate results in lower data usage. This outcome is expected, as relay under Erlay is designed to optimize bandwidth utilization more efficiently than fanout.

Additionally, it is worth noting that when the number of inbound connections is lower, the bandwidth slope is less steep when the network connectivity is increased.

###  `OUTBOUND_FANOUT_DESTINATIONS=1, INBOUND_FANOUT_DESTINATIONS_FRACTION=[0.0, 1.0]`

When keeping the fanout rate low, we observe substantial bandwidth savings. The configuration suggested in the current Erlay approach (1/10%) achieves approximately **35% savings** compared to the baseline when using **8 outbound connections**. The efficiency improves further as connectivity increases: at **12 outbound connections**, the savings reach around **45%**.

![Data volume per tx (reachable nodes) (fanout_out_1)|684x424](upload://rW9osK8Q2z6e5KaKmvucqFZJaZt.png)
![Data volume per tx (unreachable nodes) (fanout_out_1)|684x424](upload://zI1GTOd0Bf9vQ9aDgPtK6Jk55vy.png)

### `OUTBOUND_FANOUT_DESTINATIONS=4, INBOUND_FANOUT_DESTINATIONS_FRACTION=[0.0, 1.0]`

Increasing the outbound fanout rate show to reduce the potential savings, as this affects all types of nodes in the network (both reachable and unreachable). The savings are still significant, as long as we select a small enough number of inbounds to fanout to. For instance, the **4/10%** configuration achieves approximately **21% savings** compared to the baseline when using **8 outbounds**, and about **28%** when increasing the node's connectivity to 12.

![Data volume per tx (reachable nodes) (fanout_out_4)|684x424](upload://v6RLTht0YXvyzJbMBKrx2w4WGSX.png)
![Data volume per tx (unreachable nodes)(fanout_out_4)|684x424](upload://qAnci9U8iVX2qXpCrO6r7Po1xB2.png)

### `OUTBOUND_FANOUT_DESTINATIONS=7, INBOUND_FANOUT_DESTINATIONS_FRACTION=[0.0, 1.0]`

Finally, we can see how heavily increasing the fanout rate leads to the saving becoming negligible with the current 8 outbound peers configuration, and really small even if increasing the connectivity: **less than 10% for the 12 outbound peers configuration.**

![Data volume per tx (reachable nodes) (fanout_out_7)|684x424](upload://lEhcmqeOVxJUg1KQE8QsPqpfU2W.png)
![Data volume per tx (unreachable nodes) (fanout_out_7)|684x424](upload://2MsKQ7pnHoogNdMypjbu7IjM3LK.png)

## Bandwidth vs Latency

Examining bandwidth and latency improvements in isolation wouldn’t be meaningful, as there is a clear tradeoff between the two, as outlined in our thesis.

Focusing on the **1/10%** configuration (the approach implemented in the current Erlay design) we observe that while bandwidth savings are substantial, the propagation time increases significantly, by approximately **2.4×**. The **4/10%** configuration reduces this propagation delay to **2×**, but at the cost of around **40% less bandwidth savings**.

# Conclusion

Our experiments show how there is a clear tradeoff between bandwidth and latency when implementing Erlay. When using a constant fanout rate, the potential bandwidth saving will be dependent on what propagation latency is acceptable, so the configuration parameters may be dependent on that. 

Alternative approaches, such as a variable fanout rate based on transaction propagation knowledge or heuristic-based optimizations, may yield better results. However, our findings provide a useful reference for understanding the relationship between latency and bandwidth across a wide range of configurations. Additionally, these results give us a baseline to compare to when designing alternative approaches.



[^1]: In a partially deployed Erlay network, the fanout rate will be higher since regular nodes will only relay using fanout

-------------------------

