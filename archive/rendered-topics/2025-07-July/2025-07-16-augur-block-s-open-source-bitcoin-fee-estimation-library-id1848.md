# Augur: Block’s Open Source Bitcoin Fee Estimation Library

lauren | 2025-07-24 20:29:33 UTC | #1

# Augur: Block's Open Source Bitcoin Fee Estimation Library

At Block, we've just open-sourced [Augur](https://github.com/block/bitcoin-augur), the Bitcoin fee estimator that powers all of Block's on-chain transactions. While Block's engineering blog [post](https://engineering.block.xyz/blog/augur-an-open-source-bitcoin-fee-estimation-library) covers the broader motivations behind the project, we wanted to share some technical details here - what we built, how it works, how it performs, and what's next.

## What We Built

**Augur** is a mempool-based Bitcoin fee estimator designed to generate real-time fee recommendations across a range of confirmation targets and confidence levels.

Highlights:

* **Mempool-driven:** Estimation relies on live mempool data, not historical block data.  
* **Reference implementation:** A starter Kotlin [package](https://github.com/block/bitcoin-augur-reference) is available for developers.  
* **Public API endpoint:** You can query live fee estimates, updated every 30 seconds, at  
    [`https://pricing.bitcoin.block.xyz/fees`](https://pricing.bitcoin.block.xyz/fees)

Here's a sample response from the API:

```json
{
  "estimates": {
    "6": {
      "probabilities": {
        "0.50": { "fee_rate": 2.1135 },
        "0.80": { "fee_rate": 4.0277 },
        "0.95": { "fee_rate": 6.008 }
      }
    },
    "144": {
      "probabilities": {
        "0.50": { "fee_rate": 1.0101 },
        "0.80": { "fee_rate": 1.0202 },
        "0.95": { "fee_rate": 1.1052 }
      }
    }
  },
  "mempool_update_time": "2025-05-27T16:09:18.461Z"
}
```

Interpretation Example: *If I pay 1.1052 sat/vB in fees, then I can be 95% confident that my transaction will be confirmed within 144 blocks.*

## How It Works

Augur builds on the mempool-first approach popularized by @FelixWeis in [WhatTheFee](https://whatthefee.io). Unlike Bitcoin Core that models fee dynamics based on past blocks, Augur considers the current mempool state, transaction arrival rates, and the inherent randomness of Bitcoin mining.

* Current mempool weight: Transactions grouped by fee rate into buckets, tracking total weight at each fee level.  
* Transaction inflow rates: Measured over both short (30 min) and long (24 hr) windows to capture recent market dynamics.  
* Block production probability: Statistical modeling of how many blocks will likely be found in your target timeframe using a Poisson distribution.

We took the original WhatTheFee [source code](https://github.com/FelixWeis/WhatTheFee--legacy) and re-engineered it into a **production-ready Kotlin library**—a somewhat unconventional but practical choice, since much of our internal infrastructure runs on Kotlin. In the process, we changed to updating mempool snapshots and estimates continuously as opposed to just after a new block is mined. We also implemented JetBrains' [viktor](https://github.com/JetBrains-Research/viktor) for faster numerical array operations.

## How We Measured It

We developed a **custom benchmarking tool** to evaluate Augur's accuracy and cost-effectiveness against other major fee estimators. Tests were conducted under both high and moderate mempool volatility.

To benchmark, we compared estimated fees to actual transaction fees confirmed in each block. We chose to use the following thresholds based on @ismaelsadeeq’s informative [post](https://delvingbitcoin.org/t/mempool-based-fee-estimation-on-bitcoin-core/703/7):

* **5th percentile (p5):** The confirmation threshold. Transactions below this fee rate likely wouldn't have made it into the block.  
* **75th percentile (p75):** The overestimation threshold. Fees above this level were considered unnecessarily high.

Using these thresholds, we calculated these metrics:

* **Miss Rate:** Percentage of estimates that were too low (i.e., didn't confirm, below p5)  
* **Average Overestimate:** How much users overpaid when estimates were high (above p75)  
* **Total Difference:** Composite score capturing both under and overestimation

More benchmarking results can be found in Block's engineering blog [post](https://engineering.block.xyz/blog/augur-an-open-source-bitcoin-fee-estimation-library), but below is one example for a 1-3 block target confirmation during a two-week period of normal-to-moderate volatility (May 2-15, 2025). 

| Provider | Miss Rate | Avg Overestimate | Total Difference |
| :---- | :---- | :---- | :---- |
| **Augur** | 14.1% | 15.9% | **13.6%** |
| WhatTheFee | 14.0% | 18.7% | 16.1% |
| Mempool.space | 24.4% | 21.7% | 16.4% |
| Bitcoiner.Live | **3.6%** | **65.5%** | 63.2% |
| Blockstream | 18.7% | 44.2% | 35.9% |


Bitcoiner.live achieved its low miss rate by **massive overpayment**—users paid over 65% more than necessary. Augur, by contrast, struck a better balance between reliability and cost, delivering acceptable success rates at roughly **1/4 the cost**.


## What's Next

We're building an **open-source version** of our benchmarking tool to enable the community to:

* Reproduce our performance metrics  
* Backtest other fee estimators using historical fee provider data  
* Contribute improvements to Augur or build their own estimators

We'll share the benchmarking tool here once it's ready. In the meantime, feel free to check out [Augur](https://github.com/block/bitcoin-augur) and its [reference implementation](https://github.com/block/bitcoin-augur-reference). @zpv and I welcome any questions or feedback in our posts and repos.

## Addendum
In our original post we discussed our benchmarking results against various providers and we incorrectly used the label "Blockstream (Bitcoin Core)" instead of "Blockstream". To clarify, we used the publicly available Blockstream API when retrieving fee estimates and we can't speak to the underlying algorithms or versions that provide these estimates.

We did not explicitly benchmark Augur against Bitcoin Core but we would like to do that, time permitting, in the future, and add it to our [open-source benchmarking tool](https://github.com/block/bitcoin-augur-benchmarking) that is now available. Thanks to @ismaelsadeeq for pointing out that if we do, we should use v28.0+ for its default economical mode.

-------------------------

ismaelsadeeq | 2025-07-17 09:26:05 UTC | #2

This is great work!

> Blockstream (Bitcoin Core): 18.7% / 44.2% / 35.9%

Which version of Bitcoin Core did you benchmark?

One important metric we can also measure is the average overestimation factor i.e, 2x, 3x, etc. This would help indicate the severity of overpayment, because a little overpayment is negligible for conservative users.

> We’re building an open-source version of our benchmarking tool to enable the community to:

Really looking forward to it.

I did experiment and play around with the library for a while, I've dropped some thoughts on tracking issue https://github.com/block/bitcoin-augur/issues/3

-------------------------

lauren | 2025-07-17 20:51:59 UTC | #3

Not 100% sure which version of Bitcoin Core is used to return results from `curl https://blockstream.info/api/fee-estimates`, but based on [this](https://github.com/Blockstream/esplora/blob/52de3ccf39c56ff839e26829ccd6d18f169832f7/contrib/Dockerfile.base#L37) file, I'm guessing 27.2.

v1 of the benchmarking tool should be ready very soon (@zpv has been busy this week!), at which point we're happy to iterate with incorporating additional measures (average overestimation factor, etc.)

-------------------------

ismaelsadeeq | 2025-07-17 21:30:12 UTC | #4

[quote="lauren, post:3, topic:1848"]
Not 100% sure which version of Bitcoin Core is used to return results from `curl https://blockstream.info/api/fee-estimates`, but based on [this](https://github.com/Blockstream/esplora/blob/52de3ccf39c56ff839e26829ccd6d18f169832f7/contrib/Dockerfile.base#L37) file, I’m guessing 27.2
[/quote]

Nice. I expect lower overestimation and a lower average overestimation factor on versions >= v28.0 because of improvement made to it (switching default mode to economical).

-------------------------

zpv | 2025-07-23 00:36:52 UTC | #5

We've now released the dataset and benchmarking tool used to generate the metrics in our blog post. The dataset includes Bitcoin block fee spans and fee estimations from multiple providers covering January 2025 through June 2025.

Check it out here: https://github.com/block/bitcoin-augur-benchmarking

Feel free to explore, reproduce our results, and share any feedback!

-------------------------

