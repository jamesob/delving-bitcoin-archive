# Binary Fuse filters as an alternative to BIP 158 GCS

purszki | 2026-04-19 08:09:18 UTC | #1

Hi Everyone,

While at Chaincode BOSS 2026, I started looking into whether we can do better than the Golomb-Coded Sets (GCS) used in BIP158.
Binary Fuse filters didn’t exist when BIP158 was drafted, and I suspected they might offer a better CPU/bandwidth trade-off for light clients.

I built a benchmarking framework using Core’s infrastructure to see if modern set-membership algorithms brings something to the table or it is just theory.

The framework processes \~50,000 mainnet blocks and tests various wallet sizes across x86 and ARM (Raspberry Pi 5).
It validates everything against ground truth to ensure I’m not wrong.

You can find the code and full methodology here: [Light Client Block Filter Research (Binary Fuse Filters)](https://purszki.github.io/bitcoin_research_01/)

Shortly, the 16-bit Binary Fuse variant (Fuse16) is a strong candidate for a drop-in replacement for GCS.
To be honest, the CPU win was more than I expected:

* Client CPU work: \~9×–80× reduction on desktop; \~6×–45× on ARM.
* Bandwidth: A negligible increase (\~0–3%) in the scenarios I tested.

The 16-bit filter is essentially “flat” here. It’s a simple replacement of GCS to Fuse16.

I also have some ideas for hierarchical constructions to shave down the bandwidth further,
but that introduces more complexity and potential privacy leaks that need a closer look.

A few questions before I go deeper:

* Is a massive CPU gain worth a \~2-3% bandwidth hit? Is this tiny bandwidth increase a significant concern for the network, or is the dramatic CPU efficiency more critical?
* Any "gotchas" with Binary Fuse I might be missing?
* I'm not a security expert, so if you see any potential threats or security trade-off I didn't think of, please let me know.
* Is the benchmarking framework itself worth polishing?
  Personally, I’m okay with the current "research-grade" code, as it served my purpose.
  I'd be happy to make it more modular so others can drop in their own algorithms and compare against BIP158 on real mainnet data — if people find that useful.

A personal note: I’ve been working in corporate environment my whole carrier, and this is my first open-source post. Also, this is my first bitcoin-post. Honestly, I’m a bit nervous to put this out there…

Any feedback is appreciated.

Cheers,
Csaba

-------------------------

rustaceanrob | 2026-04-26 07:54:02 UTC | #2

First off, thanks for taking a look at new filter types. Great to see others are interested in this research. A few comments:

The benchmark compares bandwidth but the false positive rates do not match, GCS has a false positive rate of `1/784931` whereas BF with 16-bit fingerprints is `1/65536`. To reach parody with the GCS false positive would be 20-bits. As a follow up, it could be interesting to build the index and measure a wallet rescan using varying fingerprint sizes. I might take this on.

One simple “gotcha” for BF is how values are seeded and how the filter is constructed. To mitigate targeted hash collisions we would likely want to use the block hash as a seed/entropy. There is a possibility the algorithm fails in the case of BF, in the case where there is a cycle in the graph. I wonder how often this occurs in practice.

Some general feedback I’ve gotten on using GCS for mobile is that the battery draw can make CBF unappealing for users. The reported numbers on CPU are promising. I can imagine a class of user that is willing to trade off higher bandwidth for faster queries and lower CPU expenditure, perhaps a mobile user on WiFi or high throughput data plan. For those running Bitcoin Core, I can see potential for speeding up wallet rescans with these as well.

-------------------------

purszki | 2026-04-28 20:08:16 UTC | #3

[quote="rustaceanrob, post:2, topic:2428"]
The benchmark compares bandwidth but the false positive rates do not match, GCS has a false positive rate of `1/784931` whereas BF with 16-bit fingerprints is `1/65536`. To reach parody with the GCS false positive would be 20-bits.

[/quote]

Thank you for the review! I appreciate it a lot.

I think the measured false-positive counts do match the expected rates, let me explain why.

I also believe that the measures on 10 different wallet scenarios supports that Fuse16 is providing better bandwidth and CPU load than Fuse20, even though Fuse20’s FP rate is closer to GCS. It seems strange, but I believe my 50k mainnet dataset with 10 wallet scenarios do support this statement.  

I agree with you, that F20 is closer to GCS on per query FP rate. But the benchmark is measuring something more practical: *filter bandwidth plus block downloads caused by false
positives for multiple scripts*. This is what really matters because this is the real bandwidth we are using. Fuse20 has better FP rate, but also has a bigger filter size, so eventually Fuse16 is providing better practical results on my 50k blocks mainnet dataset.  

I think that the measured false positive numbers do match the excepted rates. Here is why.

Here you can find the actual false positive ratios of all the measured filters:

 https://github.com/purszki/bitcoin_research_01/blob/fuse_filter_research/light_client_research/results/raspberry_pi5_wallet_benchmark_results_50000.txt 

For example, for the simple_user wallet (24 scripts):
GCS: matches=1037, FP=3
F16: matches=1053, FP=19
F18: matches=1041, FP=7
F20: matches=1035, FP=1

The ground truth has 1034 real matching blocks. For Fuse filters, 55 blocks are currently skipped because I have not handled the tiny-filter corner cases yet, so the number of negative blocks tested is: 50000 - 55 - 1034 = 48911

What is the probability of a false positive block in case of a Fuse16 filter? It requires some calculations. The probability of a FP query of a single script: 1/65536. The probability that a query is non-false positive: 1-1/65536 = 65535/65536. We have 24 scripts in the simple_wallet test case, so the probability of a non-false positive block query for the wallet: (65535/65536)^24. The probability of a false positive block query: 1-(65535/65536)^24 = 0.00036614.  We had 48911 investigated blocks, so the expected value of the false positive hits is 48911\*0.00036614=17.9. We measured 19. I think it’s pretty close. 

It’s also an interesting question that 50k block mainnet datasets and the example wallets are conclusive enough to tell that Fuse16 is better than Fuse20… I feel the dataset is huge enough, however I’m not sure about the wallet examples. I am not super-certain that the current wallet examples cover enough realistic wallet shapes.

-------------------------

purszki | 2026-04-28 20:11:49 UTC | #4

[quote="rustaceanrob, post:2, topic:2428"]
One simple “gotcha” for BF is how values are seeded and how the filter is constructed. To mitigate targeted hash collisions we would likely want to use the block hash as a seed/entropy. There is a possibility the algorithm fails in the case of BF, in the case where there is a cycle in the graph. I wonder how often this occurs in practice.

[/quote]

100% I agree, this is one of the open points in my write-up.

My current intuition is that this should be solvable, but I didn’t work on the solution yet.

-------------------------

