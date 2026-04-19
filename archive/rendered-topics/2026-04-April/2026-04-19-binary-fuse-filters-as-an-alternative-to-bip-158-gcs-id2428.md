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

