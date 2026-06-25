# Roast our real-time Bitcoin explorer built for everyday Bitcoin users

DaniB | 2026-06-25 17:35:45 UTC | #1

Hi everyone,

We are building **BlockSight.Live**, a free real-time Bitcoin blockchain explorer.

[https://blocksight.live](https://blocksight.live)

The idea behind BlockSight is different from most traditional Bitcoin explorers.

Many explorers are powerful, but they often feel like raw database interfaces: dense pages, technical fields everywhere and information that is useful mainly if you already know exactly what you are looking at.

We wanted to build a Bitcoin explorer that is still useful technically, but much easier to understand for everyday Bitcoin users.

BlockSight is focused on practical daily use: checking whether a transaction was confirmed, following confirmations, looking at the current block height, understanding mempool congestion, estimating fees in sats/vB and seeing what is happening on the Bitcoin network in real time.

The explorer includes live Bitcoin blockchain data, transaction search, address search, recent blocks, mempool visibility, fee estimation, congestion monitoring and real-time network visualization.

It is Bitcoin-only, free to use, requires no registration and is currently available in 31 languages.

The multilingual part is important to us. Bitcoin is global, but many Bitcoin tools still feel built mainly for English-speaking technical users. We want a normal Bitcoin user in Spanish, Hindi, Hebrew, Portuguese, German or any other supported language to be able to understand what is happening on-chain without feeling like the tool was not made for them.

The goal is not to hide technical Bitcoin data. The goal is to present it in a way that is more useful for daily use.

This is still being improved, and I would appreciate honest technical feedback from this community.

Please roast the implementation, UX and data presentation. Practical criticism is welcome.

-------------------------

sipa | 2026-06-25 18:08:11 UTC | #2

Showing the time to the next block as 10 minutes after the previous block is not correct. The time between blocks follows (roughly) an exponential distribution, which is memoryless. This means that the expected time to the next block is 10 minutes **from the current time**. It does not decrease depending on how long ago the previous block was.

-------------------------

DaniB | 2026-06-25 18:37:06 UTC | #3

you're right about the math, no argument there. expected time to the next block is \~10 min from now, doesn't matter how long it's been. memoryless.

but that's not what the timer is showing. it's not a forecast of the next block. it's where we are in the cadence.

10 min is the protocol's target. difficulty retargets to hold that mean. so we start the clock at 10:00, the network's heartbeat, and show how far the current beat has run. once it passes 10:00 it just counts the delay.

the question i'm answering for the user is "are we on time or running late," not "when is the next block." that second one i won't predict, because you're right, you can't.

**simplest honest read of the current status. that's the whole idea.**

-------------------------

