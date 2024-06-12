# Bitcoind Policy Estimator modes Analysis

ismaelsadeeq | 2024-06-12 15:06:55 UTC | #1

This analysis examines the fee estimation data from Block `846887` to `847322`, covering a total of `435` blocks, to analyze the difference between estimates from [Bitcoind Policy Estimator](https://johnnewbery.com/an-intro-to-bitcoin-core-fee-estimation/) `conservative` and `economical` modes.

### Methodology

- Logged and traced `conservative` and `economical` fee estimates every minute.
- Logged and traced all connected blocks' percentile fee rates ([Branch implementation](https://github.com/ismaelsadeeq/bitcoin/tree/new-fee-estimator-data)).
- Collected and cleaned data from both the traced and debug logs ([Data](https://gist.github.com/ismaelsadeeq/6a6531e9b96bfeed20178e353b187332)).
- Analyzed the data, plotting Bitcoind `conservative` and `economical` estimates against the confirmed block target (50th - 5th percentile fee rate range).

### Definitions

- **`conservative` mode**: This is the `estimatesmartfee` RPC mode which considers a longer history of blocks. It potentially returns a higher fee rate and is more likely to be sufficient for the desired target, but it is not as responsive to short-term drops in the prevailing fee market.
- **`economical` mode**: This is the `estimatesmartfee` RPC mode where estimates are potentially lower and more responsive to short-term drops in the prevailing fee market.
- **Overpaid Estimates**: An estimate above the 50th percentile fee rate of the target block.
- **Underpaid Estimates**: An estimate below the 5th percentile fee rate of the target block.
- **Estimates Within Range**: An estimate that falls between the 5th and 50th percentile fee rates of the target block.

### Data Summary for Bitcoind `estimatesmartfee`

A total of 13,718 estimates were made from `2024-06-07 12:18:25` to `2024-06-10 07:48:05` from Block `846886` to Block `847321`, with confirmation targets of 1 and 2.

| Estimator                 | Overpaid Estimates | Underpaid Estimates | Estimates Within Range |
|---------------------------|--------------------|---------------------|------------------------|
| Bitcoind Conservative     | 3760 (96.14%)      | 66 (1.69%)          | 85 (2.17%)             |
| Bitcoind Economic         | 3033 (77.55%)      | 433 (11.07%)        | 445 (11.38%)           |

#### Logscale Graph

Estimates from block `846887` to block `847087` plotted on a log scale:

![bitcoind economic v conservative 1|690x417](https://hackmd.io/_uploads/SyKxxVDHC.png)


Estimates from block `847087` to block `847287` plotted on a log scale:

![bitcoind economic v conservative 2|690x417](upload://3AmbdfWEWi661cvC2VLqrMsWfU8.png)


#### Absolute Values Scale

Estimates from block `846887` to block `847087` plotted using absolute values:

![bitcoind economic v conservative 1 normal|690x417](upload://tUYk60hBSzbdIsjQwvBoMgskP7F.png)


Estimates from block `847087` to block `847287` plotted using absolute values:

![Bitcoind economic v conservative 2 normal|690x417](upload://oEfgBdp7P3AvLuACUAjAtQvn7jc.png)


You can recreate these graphs, summaries, and more from the repository using the branch 'analyse-bitcoind-estimates': [Fee Estimates Analysis](https://github.com/ismaelsadeeq/fee-estimates-analysis/tree/analyse-bitcoind-estimates).

### Key Findings

- This empirical data shows that the economic mode has a much lower overestimation compared to the conservative mode. [The previous analysis](https://delvingbitcoin.org/t/mempool-based-fee-estimation-on-bitcoin-core/703) used the economic mode. There was an issue reporting this claim in [Bitcoin issue #30009](https://github.com/bitcoin/bitcoin/issues/30009). These graphs show that the economic mode responds much quicker to recent fee market adjustments than the conservative mode.

How significant is the overestimation by `estimatesmartfee` conservative mode?

Here is a worst case scenario estimate for block `847088` or `847089`:

Block `847088` was confirmed with a 50th percentile fee rate of `38.7` sat/vB. 

- `estimatesmartfee` `conservative` mode was an excessive `505.7` sat/vB. 
- `estimatesmartfee` `economical` mode was much lower at `48.3` sat/vB.

![Worst case|690x417](upload://fTqWmGqI5P0eGA4bF6E0RyhS5re.png)


Many transactions paid this high fee to get included, as seen [here](https://mempool.space/block/0000000000000000000074877b16a3ca2a512114f731fc76e226ec77dcfc38db).
 
For an average `P2WPKH` 2-input 2-output transaction [weighing](https://bitcoinops.org/en/tools/calc-size/) approximately `208.5` vbytes, the costs are:

- **Bitcoind Economic estimate: `208.5 * 48.3 = 10071 satoshis (≃ $6.73)`**
- **Bitcoind Conservative estimate: `208.5 * 505.7 = 105438 satoshis (≃ $70.52)`**

Using the conservative estimate results in paying 15 times more than necessary to get included in the block. This excessive overestimation continues for quite some time, causing many transactions to overpay.

![worst case continuesly|690x441](upload://2mqCJOBBg7f1pesxWFd2EKUlUPj.png)

This analysis reveals that the overestimation by the `estimatesmartfee` `conservative` mode is not an isolated incident but a recurring trend. The significant discrepancy in fee estimates, as seen in the case of Block `847088` and more, where the `conservative` estimate was more than ten times higher than necessary, warrants modifying the default estimation mode to `economical` given the current fee market behavior.

Rapid adjustments are frequent, it is imperative to adopt a more responsive fee estimation mode to reduce paying more than necessary.

-------------------------

