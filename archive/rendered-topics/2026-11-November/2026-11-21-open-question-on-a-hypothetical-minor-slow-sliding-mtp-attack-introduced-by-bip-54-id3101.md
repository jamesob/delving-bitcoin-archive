# Open Question on a hypothetical minor "slow-sliding MTP" attack introduced by BIP 54

ariard | 2026-11-21 04:07:39 UTC | #1

In the process of reviewing BIP54, and while I was brooding back on the MTP usage by "forward block", it comes to my mind, there might be slight and subtle security changes introduced by BIP54. I don't think it is significant enough to warrant the security benefit brought by BIP54, though it did sound interesting to me to share the insight.

As I was reminding it in some [ML thread](https://groups.google.com/g/bitcoindev/c/tKETFYnEtng), some long-term attack vectors known among bitcoin experts to disrupt a competing miner engaged in the global race of finding valid proof-of-works is to hijack the settings of its local clock. Indeed the local clock can be leveraged to set the [`nTime`](https://github.com/bitcoin/bitcoin/blob/004cfb21a51db563b92cde968f249086ef58df7c/src/primitives/block.h#L33) of an upcoming valid block and falsifying it allow to mislead a target miner in infringing the "*block timestamp too far in the future*" [consensus rule](https://github.com/bitcoin/bitcoin/blob/004cfb21a51db563b92cde968f249086ef58df7c/src/validation.cpp#L4197). Let's frame this attack vector as an adversarial header time hijacking.

I do wonder if the new rule introduced by BIP54 i.e "if `N % 2016` is equal to 0, the timestamp of the block must be set to a value higher than or equal to the value of the timestamp of block at height `N-1 `minus 7200 (T<sub>N</sub> ≥ T<sub>N−1</sub> − 7200) seconds;" is not introducing some "reverse header time hijacking", where the header for a block at height `N % 2016 == 0` is made conditional on the value
of the block at height `N -1`.

Therefore, who is able to set the nTime of the block `N -1` (be it the previous miner herself or himself or an hijack targeting said miner) might be able to affect the validity of the next block in a discretionary fashion. While at first sight, this might appear as a problem, the window margin of validity for the differential between the block `N -1`  and the block `N` have been selected in such a way that it is [superior](https://github.com/bitcoin/bitcoin/pull/35793/changes#diff-74d3d558e2ae1b1c967ea13d01ce15875cc7de50c93c26db73ccb3e4db5cfdacR34) to the median-time-past interval on the assumption of an expected interval block time of 600 seconds (i.e 7200 seconds is superior to 6000 seconds of the expected interval between the first block and the last block of the median range).

This is where another observation should be made, technically there is nothing in consensus preventing the median-time-past to be adversarial pin in the past i.e all the timestamps composing a median for the last 11 blocks being set to minimally increasing value (e.g for a range with a positive diff from the chain tip at the latest 12th block: `[+1, +2, +3, +4, +5, +6, +7, +8, +9, +10, +11]`). If this median timestamp layout and maintain for a sliding window large enough for a 7200 diff accumulating between the latest element of the MTP range and the "real-world" time,
one might adversarial slips in a +7200 diff to provoke the invalidity of the block at height `N % 2016 == 0`.

This attack vector, dumbly dubbed a "*slow-sliding-mtp*" attack is very unlikely as it assuming that the adversarial entity is able to keep coalesce under its control a hashrate superior to 51%. Moreover the impact would be still limited as it's only concerning an advantage for 1 block among a sequence of 2016 ones. Yet in some adversarial scenario (e.g for selfish-mining attack), it could be combined to build an edge on the competition by selectively invalidating some block.

This novel attack vector sounds "novel" as under consensus rule whatever the median-time-past lateness with regards to the "real-world" time it's not coming as a source of invalidity for a new block added to the chain tip. It does sound possible to do a "fast-forward" sliding in the future where the MTP would be pushed in the future to render invalid an upcoming block (while still under the 2 hours window), however it's a bit of a "taste of one's own medicine", situation here (there is no guarantee on how the set of majority of network nodes set their local clocks here).

To conclude, I don't believe it's changing anything for BIP54, though, if I'm correct in my understanding of bitcoin consensus rules, my observations and reasoning, I find it was an interesting insight to share.

*edited: to fix my own english on some pronoun usage*

-------------------------

