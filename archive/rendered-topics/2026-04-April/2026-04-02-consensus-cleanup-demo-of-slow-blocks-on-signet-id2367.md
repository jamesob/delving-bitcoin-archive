# Consensus Cleanup: demo of slow blocks on Signet

AntoineP | 2026-04-03 14:12:36 UTC | #1

We are going to do a demo of long-to-validate blocks on Signet on Wednesday. Here are instructions on how you can join.

The goal of this demo is to let end users see for themselves the impact of hard to validate blocks. We have crafted a series of 6 blocks that are hard to validate, but [not too much](https://groups.google.com/g/bitcoindev/c/wOVjJoLDWfA/m/zWcDG7_7CQAJ). Each should take between a handful of seconds and one minute to validate, depending on the hardware. @ajtowns is going to mine the six blocks in a row and then reorg them out, so that anyone running a Signet node at the time would see them, but they would not impose a cost to IBD forever.

We are going to do three runs of the demo so everyone has a chance to join live:
1. [date=2026-04-08 time=14:00:00 timezone=Etc/UTC]
2. [date=2026-04-08 time=22:00:00 timezone=Etc/UTC]
3. [date=2026-04-09 time=09:00:00 timezone=Etc/UTC]

## How to join the demo live

All you need to join the demo is run a Bitcoin Core node on Signet. You would be able to observe how long it took your own node to validate each block, and compare arrival time with other participants. Feel free to share your results in this thread!

Your Signet node needs to be fully-synced. The historical block chain is still pretty small, you would need about 30GiB of disk space. If necessary, you can also run in `-prune` mode. It will not affect your ability to participate in the demo.

If you are not able to run a Signet node yourself, you should still be able to observe the blocks as they arrive on [mempool.space](https://mempool.space/signet) and the fork dynamics at the [temporary peer.observer instance](https://signet.peer.observer/forks/) for Signet.

*Later i may add detailed instructions here on how to run a Signet node and inspect the logs for various end-user platforms.*

-------------------------

shinobi | 2026-04-02 20:58:49 UTC | #2

Just wanted to suggest vibe coding a basic GUI/display to pull from the logs and display to a user validation times as the blocks come in. I think without an easy visual component there isn’t much to hook many users to directly spin up a signet node and participate with their own node.

-------------------------

0xB10C | 2026-04-02 22:52:22 UTC | #3

[quote="AntoineP, post:1, topic:2367"]
* Wednesday April 8th at **2pm UTC** (7am SF / 10am NY / 4pm Paris / 1am Sydney)
* Wednesday April 8th at **10pm UTC** (3pm SF / 6pm NY / midnight Paris / 9am Sydney)
* Thursday April 9th at **9am UTC** (2am SF / 5am NY / 11am Paris / 8pm Sydney)
[/quote]

In your timezone, this is:
1. [date=2026-04-08 time=14:00:00 timezone=Etc/UTC]
2. [date=2026-04-08 time=22:00:00 timezone=Etc/UTC]
3. [date=2026-04-09 time=09:00:00 timezone=Etc/UTC]

Maybe adding this to the OP makes sense too.

-------------------------

AntoineP | 2026-04-03 16:47:41 UTC | #4

I guess this is also a good occasion for testing Bitcoin Core's latest release candidate. If you are joining the demo and comfortable with running Bitcoin Core already, consider using [the binaries from the latest "rc" for 31.0](https://bitcoincore.org/bin/bitcoin-core-31.0/).

-------------------------

ajtowns | 2026-04-03 18:08:34 UTC | #5

[quote="shinobi, post:2, topic:2367"]
a basic GUI/display to pull from the logs and display to a user validation times as the blocks come in.
[/quote]

Here's a patch for bitcoin-tui that people might find interesting to try:

https://github.com/janb84/bitcoin-tui/pull/25

This is what a 5-block reorg looks like in it (without any of the blocks being slow):

![tui-slow-blocks|690x459](upload://pMfcvaF4GvFukmM5Qk0CGq4gwxJ.png)

Enabling `-debug=cmpctblock` and `-debug=bench` logging are good and not very expensive and are required to fill out some of the columns; enabling `-debug=net` logging as well might also gather some interesting info.

-------------------------

