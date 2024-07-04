# Empirical Data Analysis of Fee Rate Forecasters for ASAP (next-block) Fee Estimation

ismaelsadeeq | 2024-07-04 10:47:56 UTC | #1

This analysis examines the fee estimation data from Block `848920` to `850213`, covering a total of `1293` blocks, for several fee rate forecasters. The primary objective is to assess the effectiveness of these forecasters in comparison with the [Bitcoind Policy estimator](https://johnnewbery.com/an-intro-to-bitcoin-core-fee-estimation/) and test the effectiveness of https://github.com/bitcoin/bitcoin/pull/30157.

The forecasters are:

- **Mempool**: Uses all mempool transactions to generate a block template and returns the 50th and 25th percentile mining scores as high and low fee rate estimates.
- **Mempool Last 10 Minutes**: Uses mempool transactions to generate a block template. Packages received in the last 10 minutes are accounted for twice,  the forecaster returns the 50th and 25th percentile fee rates.
  - **Rationale**: The next block is expected in 10 minutes, and we assume that the flow of the previous 10 minutes will be similar to the flow of the next 10 minutes.
- **Last Block**: Uses the 25th and 50th percentile fee rate of the transactions in the last block seen in our mempool.
  - **Rationale**:This is a worst-case scenario indicator. This method does not consider anything else, so forecasters should be at least as good as this.
- **Last 6 Blocks**: Uses the 25th and 50th average percentiles of the transactions in the last 6 blocks seen in our mempool.
  - **Rationale**: It is difficult to reorg out the last six blocks; hence, miners are less likely to manipulate them. The primary objective is to determine whether this forecaster can serve as a fallback for mempool-based forecasters to prevent miner manipulation. See [Finney attack](https://delvingbitcoin.org/t/mempool-based-fee-estimation-on-bitcoin-core/703/6?u=ismaelsadeeq).

### Methodology

- Implemented all forecasters to log fee estimates every minute.
- Adjusted the branch to log all connected blocks' percentile fee rates ([Branch implementation](https://github.com/ismaelsadeeq/bitcoin/tree/new-fee-estimator-data)).
- Collected and cleaned data from both the traced and debug logs ([Data](https://drive.google.com/file/d/13ic91Z9rUH-anz9LVL_rfoD4FKijsuSO/view?usp=sharing)).
- Analyzed the data. For each forecaster, the low priority fee estimates, high priority fee estimates, Bitcoind conservative fee estimates, and Bitcoind economic fee estimates were plotted against the confirmed block target (50th - 5th percentile fee rate range).

### Definitions

- **Overpaid Estimates**: An estimate above the 50th percentile fee rate of the target block.
- **Underpaid Estimates**: An estimate below the 5th percentile fee rate of the target block.
- **Estimates Within Range**: An estimate that falls between the 5th and 50th percentile fee rate of the target block.

### Analysis for Individual Fee Rate Forecasters

#### Mempool Forecaster

Log scale:

![mempool_log](upload://fl3Ci9OcWXyzJZZo5mGFGcVZHMi.png)

Linear scale:

![mempool_forecast_abs](upload://bTwQoFy3n736RxqioYytjBLLFnl.png)

#### Data Summary for Mempool Forecaster

| Estimator               | Overpaid Estimates | Underpaid Estimates  | Estimates Within Range |
|-------------------------|--------------------|----------------------|------------------------|
| Mempool Low Priority    | 0 (0.00%)          | 6116 (46.16%)        | 7135 (53.84%)          |
| Mempool High Priority   | 38 (0.29%)         | 4274 (32.25%)        | 8939 (67.46%)          |

#### Mempool Forecaster with Bitcoind Threshold (Economic Estimate)

Log scale:

![mempool_log_economic](upload://1o03qA7Cen6ZRBwaBIGIzyHdUrf.png)


Linear scale:

![mempool_forecast_abs_economic](upload://bZcs4vgkweRwpdR1NqWHVGwkP8t.png)

#### Data Summary for Mempool Forecaster with Bitcoind Threshold (Economic Estimate)

| Priority Level           | Overpaid Estimates | Underpaid Estimates  | Estimates Within Range |
|--------------------------|--------------------|----------------------|------------------------|
| Low Priority             | 0 (0.00%)          | 7069 (53.35%)        | 6182 (46.65%)          |
| High Priority            | 21 (0.16%)         | 5823 (43.94%)        | 7407 (55.90%)          |


#### Mempool Forecaster with Bitcoind Threshold (Conservative Estimate)

Log scale:

![mempool_log_conservative](upload://t0WXR2o9yOYoqGrSPCkAFrCyRPZ.png)

Linear scale:

![mempool_forecast_abs_conserv](upload://3CtBhcmoAtlALVSG45QzKEIsnx8.png)

#### Data Summary for Mempool Forecaster with Bitcoind Threshold (Conservative Estimate)

| Priority Level           | Overpaid Estimates | Underpaid Estimates  | Estimates Within Range |
|--------------------------|--------------------|----------------------|------------------------|
| Low Priority             | 0 (0.00%)          | 6483 (48.92%)        | 6768 (51.08%)          |
| High Priority            | 31 (0.23%)         | 4841 (36.53%)        | 8379 (63.23%)          |

#### Mempool Last 10 Minutes Forecaster

Log scale:

![mempool_last_10_log](upload://4JgRRnqB4De5Vt6fyEj9ylaVX3j.png)

Linear scale:

![mempool_last_10_min_abs](upload://9i9Gl99viEUExbDfjq6GqZYrRlY.png)


#### Data Summary for Mempool Last 10 Minutes Forecaster

| Priority Level           | Overpaid Estimates | Underpaid Estimates  | Estimates Within Range |
|--------------------------|--------------------|----------------------|------------------------|
| Low Priority             | 429 (3.24%)        | 3957 (29.86%)        | 8867 (66.91%)          |
| High Priority            | 3372 (25.44%)      | 2656 (20.04%)        | 7225 (54.52%)          |

#### Last Block Forecaster

Log scale:

![last_block_forecast_log](upload://eA03uiOnqOf8P4FT1b1ZHysaYtz.png)

Linear scale:

![last_block_forcast_abs](upload://tQGAiKTsB1W6c4hPKmhrHcTjQFO.png)

#### Data Summary for Last Block Forecaster

| Priority Level           | Overpaid Estimates | Underpaid Estimates  | Estimates Within Range |
|--------------------------|--------------------|----------------------|------------------------|
| Low Priority             | 1682 (12.98%)      | 6225 (48.02%)        | 5056 (39.00%)          |
| High Priority            | 3039 (23.44%)      | 5015 (38.69%)        | 4909 (37.87%)          |

### Average of Last 6 Blocks Forecaster

Log scale:

![last_6_block](upload://eoPQGoWHmQv9hwuFwFZ8XKq8Xyi.png)

Linear scale:

![block_abs](upload://47WQ57le2ogMH0EPIdFZWf8pyR4.png)

#### Data Summary for Last 6 Blocks Forecaster

| Priority Level           | Overpaid Estimates | Underpaid Estimates  | Estimates Within Range |
|--------------------------|--------------------|----------------------|------------------------|
| Low Priority             | 1827 (14.09%)      | 6778 (52.29%)        | 4358 (33.62%)          |
| High Priority            | 2946 (22.73%)      | 5528 (42.64%)        | 4489 (34.63%)          |


All these graphs' data points were collected from `2024-06-21 15:58:58` to `2024-07-01 09:28:42` (Blocks `848919` to `849120`) by logging estimates at 1-minute intervals consecutively. This spans a `200` block interval. You can recreate these graphs, summaries, plot for data points above `849120`, but it must not exceed `200` blocks at a time using this code: [Fee Estimates Analysis](https://github.com/ismaelsadeeq/fee-estimates-analysis/tree/forecast-analysis).

### Key Findings and Ideas for Improvement

- The last six blocks perform worse than the naive last block forecaster because mempool conditions change significantly within this period.
- Using Bitcoind conservative mode as a fallsafe is better than last 6 blocks forecaster.
  
- The mempool-based forecaster provides better predictions than other forecasters but sometimes risks underestimation.
 
  
- The mempool last 10 minutes high priority sometimes overestimates.

- The mempool last 10 minutes forecaster for low priority has almost the same accuracy as the mempool forecaster for high priority.

Log scale graph plot of the two forecasters:

![mempool_vs_mempool_last_10](upload://6bZ6vZy1jZSUQctgYhf57M1Ba9G.png)


| Estimator               | Overpaid Estimates | Underpaid Estimates  | Estimates Within Range |
|-------------------------|--------------------|----------------------|------------------------|
| Mempool High Priority   | 38 (0.29%)         | 4274 (32.25%)        | 8939 (67.46%)          |
Mempool last 10 min Low Priority             | 429 (3.24%)        | 3957 (29.86%)        | 8867 (66.91%)          |

Instead of duplicating the transaction received in the last 10 minutes we should just aim for a higher percentile in mempool forecaster when there is a high inflow of high mining score transactions see ideas below for **Smart Mempool-Based Forecaster**.

**Smart Mempool-Based Forecaster**

- Aim for higher percentile (50th and 75th) when a significant percentage of the transactions in the generated block template were received within the last 10 minutes and the previous block was mined a short while ago.
This indicates high block space demand previously, hence there is a high likelihood of transactions getting bumped out by the inflow of transactions, so pay a bit more.

- Aim for higher percentile (50th and 75th), if the time from the previous block is very short, like 1 minute. This is based on the assumption that it will take approximately 10 minutes for the next block to be mined. Targeting a higher percentile reduces the likelihood of transactions getting bumped out.

- Aim for lower percentile (15th and 25th) of the generated block template whenever a significant percentage of transactions in the generated block template were received more than 10 minutes ago and the last block was mined less than 10 minutes ago. This indicates low block space demand; fee rate estimate will likely reduce.


Offline feedback by @ClaraShk on smart mempool-based forecaster we can not be sure of this kind of heuristics without checking them with data. I think some things would really depend on how full the mempool is, for example.

As of now just taking the 25th and 50th percentiles of the mempool seems to be the most accurate approach.

---

[Mempool-based fee estimation on Bitcoin Core](https://delvingbitcoin.org/t/mempool-based-fee-estimation-on-bitcoin-core/703/8)  
[PR #30157](https://github.com/bitcoin/bitcoin/pull/30157)

-------------------------

