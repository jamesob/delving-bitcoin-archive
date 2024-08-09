# Zawy’s Alternating Timestamp Attack

murch | 2024-08-09 16:00:45 UTC | #1

Zawy’s alternating timestamp attack
---

@zawy [described](https://github.com/bitcoin/bitcoin/pull/29775#issuecomment-2276135560) yesterday an attack on the Testnet 4 PR. I’ll paraphrase below my understanding of the attack.

Similar to the Timewarp Attack, it requires the attacker to be able to timestamp the majority of blocks, but in contrast to the Timewarp Attack, it does not rely on the non-overlapping durations of difficulty periods (i.e. the off-by-one error in measuring the duration of difficulty periods). The attack looks theoretically sound, although the necessity to perform 16 weeks of selfish mining (and hopefully also the necessity to control 50% of the hashrate) should make it impractical.

The main idea of the attack is to hold back most blocks’ timestamps, but then to alternate putting timestamps set in the future and past on the difficulty periods’ last blocks to skip between maximally reducing and increasing the difficulty. The overshooting and undershooting the target time of difficulty period allows the attacker to produce significantly more blocks than would be expected to be produced by honest miners in the same time.

In detail, the attack works as follows:

1. The attacker starts by keeping timestamps as low as possible. In the original description the timestamp increases by one second every block, but the timestamp only has to be increased [every 6 seconds](https://bitcoin.stackexchange.com/a/123700/5406). The miner has a slight majority of the hashrate, so we will expect blocks to be found about every 20 minutes. We simplify by assuming that the cadence of blocks is constant across a difficulty period:

| label | height | MTP [s] | timestamp [s] | elapsed time | difficulty |
|:--:|:--:|:--:|:--:|:--:|:--:|
| start | 0 | < 0 | 0 | 0 | D |
| | 1 | < 0 | 0 | 20 min | D |
| | 2 | < 0 | 0 | 40 min | D |
| | 3 | < 0 | 0 | 60 min | D |
| | … | … | …  | … | … |
| | 6 | 0 | 1 | 2 h | D |
| | … | … | …  | … | …|
| | 12 | 1 | 2 | 4 h | D |
| | … | … | …  | … | …|

2. The attacker stamps the last block of the difficulty period 8 weeks after the attack’s start date. This block will not be acceptable to other nodes due to being more than 2 h in the future and the attacker (temporarily) forks itself off the network here. However, the future date allows the attacker to reduce the difficulty for the next period to 1/4. The first block of the second difficulty period is also timestamped 8 weeks into the future to adhere to the restraint proposed per the Great Consensus Cleanup.

| label | height | MTP [s] | timestamp [s] | elapsed time [s] | difficulty |
|:--:|:--:|:--:|:--:|:--:|:--:|
| start | 0 | < 0 | 0 | 0 | D |
| | … | … | …  | … | … |
| | 2014 | 335 | 336 | 4 w – 40 min | D |
| | 2015 | 335 | **8 w** | 4 w − 20 min | D |
| | 2016 | 336 | **8 w** | 4 w | **D/4** |

3. The attacker continues to mine blocks with the minimum timestamp in the second difficulty period, then again sets the last block in the difficulty period an additional 8 weeks into the future to reduce the difficulty by another factor 4. Due to mining with approximately half the hashrate, the first difficulty period takes the attacker four weeks, and the second takes one week.

| label | height | MTP [s] | timestamp [s] | elapsed time [s] | difficulty |
|:--:|:--:|:--:|:--:|:--:|:--:|
| start | 0 | < 0 | 0 | 0 | D |
| | 1 | < 0 | 0 | 1200 | D |
| | … | … | …  | … | … |
| | 2014 | 335 | 336 | 4 w − 40 min | D |
| | 2015 | 335 | **8 w** | 4 w − 20 min | D |
| | 2016 | 336 | **8 w** | 4 w | **D/4** |
| | 2017 | 336 | 337 | 4 w + 5 min | D/4 |
| | 2018 | 336 | 337 | 4 w + 10 min | D/4 |
| | … | … | …  | … | … |
| | 4030 | 671 | 672 | 5 w − 10 min | D/4 |
| | 4031 | 671 | **16 w** | 5 w − 5 min | D/4 |
| | 4032 | 672 | **16 w** | 5 w | **D/16** |
| | 4033 | 672 | 673 | 5 w + 75 s | **D/16** |

4. Zawy now describes that the attacker can alternate between setting the minimal timestamp and the future timestamp for the last block of difficulty periods. This causes the calculated elapsed time of some difficulty periods to be negative. However, the difficulty adjustment treats that the same as half a week and still increases the difficulty only by a factor of four. Even while adhering to the restriction proposed by the Great Consensus Cleanup against shifting the first block to the past against the preceding last block, this allows the attacker to alternate the difficulty between a quarter and a sixteenth of the starting difficulty. The attacker produces two difficulty periods of blocks every 5/4 weeks.

| label | height | MTP [s] | timestamp [s] | elapsed time [s] | difficulty |
|:--:|:--:|:--:|:--:|:--:|:--:|
| start | 0 | < 0 | 0 | 0 | D |
| | … | … | …  | … | … |
| | 2015 | 335 | **8 w** | 4 w − 20 min | D |
| | 2016 | 336 | **8 w** | 4 w | **D/4** |
| | 2017 | 336 | 337 | 4 w + 5 min | D/4 |
| | … | … | …  | … | … |
| | 4031 | 671 | **16 w** | 5 w − 5 min | D/4 |
| | 4032 | 672 | **16 w** | 5 w | **D/16** |
| | 4033 | 672 | 673 | 5 w + 75 s | D/16 |
| | … | … | …  | … | … |
| | 6047 | 1007 | **1008** | 5 w 42 h − 75 s | D/16 |
| | 6048 | 1008 | **1009** | 5 w 42 h | **D/4** |
| | 6049 | 1008 | 1009 | 5 w 42 h + 5 min | D/4 |
| | … | … | …  | … | … |
| | 8063 | 1343 | **1344** | 6 w 42 h − 5 min | D/4 |
| | 8064 | 1344 | **16 w** | 6 w 42 h | **D/16** |
| | 8065 | 1344 | 16 w | 6 w 42 h + 75 s | D/16 |
| | … | … | …  | … | … |
| | 10079 | 1679 | **1680** | 6 w 84 h − 75 s | D/16 |
| | 10080 | 1680 | **1681** | 6 w 84 h | **D/4** |
| | 10081 | 1680 | 1681 | 6 w 84 h + 5 min | D/4 |
| | … | … | …  | … | … |
| | 12095 | 2015 | **2016** | 7 w 84 h − 5 min | D/4 |
| | 12096 | 2016 | **16 w** | 7 w 84 h | **D/16** |
| | 12097 | 2016 | 16 w | 7 w 84 h + 75 s | D/16 |
| | … | … | …  | … | … |

5. Once the elapsed time reaches 16 weeks, the attacker can publish their withheld chain, and assuming slightly more total work, invalidate 16 weeks of transaction activity on the public network, reorganize about 8,064 blocks, collect block rewards for about 39,816 blocks, and maybe even collect a larger amount of transaction fees than the public network by including all non-conflicting transactions that were broadcast during the period of slow blocks.

Variant on Zawy’s Attack
---

After reading about this attack yesterday, I had an idea how to tweak it:

Instead of alternating the timestamp of the last block between minimum and 16 weeks, the attacker could decrease to the minimum in one period and ratchet up to the future in two steps:

| label | height | MTP [s] | timestamp [s] | elapsed time [s] | difficulty |
|:--:|:--:|:--:|:--:|:--:|:--:|
| start | 0 | < 0 | 0 | 0 | D |
| | 1 | < 0 | 0 | 1200 | D |
| | 2 | < 0 | 0 | 2400 | D |
| | 3 | < 0 | 0 | 3600 | D |
| | … | … | …  | … | … |
| | 6 | 0 | 1 | 7200 | D |
| | … | … | …  | … | … |
| | 12 | 1 | 2 | 14400 | D |
| | … | … | …  | … | … |
| | 2014 | 335 | 336 | 4 w − 40 min | D |
| | 2015 | 335 | **8 w** | 4 w − 20 min | D |
| 2nd diff period | 2016 | 336 | **8 w** | 4 w | **D/4** |
| | 2017 | 336 | 337 | 4 w + 5 min | D/4 |
| | 2018 | 336 | 337 | 4 w + 10 min | D/4 |
| | … | … | …  | … | … |
| | 4030 | 671 | 672 | 5 w − 10 min | D/4 |
| | 4031 | 671 | **16 w** | 5 − 5 min | D/4 |
|3rd diff period | 4032 | 672 | **16 w** | 5 w | **D/16** |
| | 4033 | 672 | 673 | 5 w + 75 s | D/16 |
| | … | … | …  | … | … |
| | 6047 | 1007 | 1008 | 5 w 42 h − 75 s | D/16 |
| | 6048 | 1008 | **1009** | 5 w 42 h | **D/4** |
| | 6049 | 1008 | 1009 | 5 w 42 h + 5 min | D/4 |
| | … | … | …  | … | … |
| | 8063 | 1343 | 1344 | 6 w 42 h − 5 min | D/4 |
| | 8064 | 1344 | **8 w** | 6 w 42 h | **D/16** |
| | 8065 | 1344 | 8 w | 6 w 42 h + 75 s | D/16 |
| | … | … | …  | … | … |
| | 10079 | 1679 | 1680 | 6 w 84 h − 75 s | D/16 |
| | 10080 | 1680 | **16 w** | 6 w 84 h | **D/64** |
| | 10081 | 1680 | 16 w | 6 w 84 h + 19 s | D/64 |
| | … | … | …  | … | … |
| | 12095 | 2015 | 2016 | … | D/64 |
| | 12096 | 2016 | **2017** | … | **D/16** |
| | 12097 | 2016 | 2017 | … | D/16 |
| | … | … | …  | … | … |
| | 14111 | 2351 | 2352 | … | D/16 |
| | 14112 | 2352 | **8 w** | … | **D/64** |
| | 14113 | 2352 | 8 w | … | D/64 |
| | … | … | …  | … | … |
| | 16127 | 2687 | 2688 | … | D/64 |
| | 16128 | 2688 | **16 w** | … | **D/256** |
| | 16129 | 2688 | 16 w | … | D/256 |
| | … | … | …  | … | … |
| | 18143 | 3023 | 3024 | … | D/256 |
| | 18144 | 3024 | **3025** | … | **D/64** |
| | 18145 | 3024 | 3025 | … | D/64 |
| | … | … | …  | … | … |

This would not only allow the attacker to produce more blocks with their attack, but also reduce the difficulty exponentially.

Conclusion
---

It may be useful to softfork an additional requirement on timestamps, that requires that the last block in a difficulty period N has a higher timestamp than the first block in the same difficulty period N, i.e. require that

$n ∈ ℕ; timestamp_{2016×n} < timestamp_{2016×n+2015}$

I don’t anticipate that honest mining would ever run afoul of a requirement to increase timestamps across a difficulty period by at least one second as that is indirectly already caused by the MTP rule, unless there is some extreme manipulation of timestamps as described above. However, it should mitigate this attack vector as far as I can tell.

-------------------------

