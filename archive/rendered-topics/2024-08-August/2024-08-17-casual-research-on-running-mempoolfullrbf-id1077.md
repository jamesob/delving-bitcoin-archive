# Casual research on running `mempoolfullrbf`

valuedmammal | 2024-08-18 19:13:58 UTC | #1


![fullrbf|690x398, 100%](upload://poqKLN0e3EUs03gBKjR3o4MYVxj.png)
_scatterplot of ~2000 data points between heights 827650 and 856176_

-------------------------

0xB10C | 2024-08-19 08:21:15 UTC | #2

Hey @valuedmammal, it's not clear to me what your chart shows.

-------------------------

valuedmammal | 2024-08-19 11:09:32 UTC | #3

Right, the reason is that I tried to write a post and being a new user I couldn't include more than one piece of media, so this is a work in progress for now while the writeup currently lives at https://valuedmammal.github.io/. In short, this "score" is the fraction of block tx data in a newly confirmed block that was expected by the node's recent block template.

-------------------------

valuedmammal | 2024-08-27 01:15:31 UTC | #4

For comparison this is the plot taken from the BIP125 node over a similar block range.

![bip125|690x397, 100%](upload://2XvA8w7pxOJ5TJ0nKOllTCozPFv.jpeg)

-------------------------

valuedmammal | 2024-08-27 01:31:46 UTC | #5

This is a test showing that the full-rbf node (B) saw significantly less variance in _score_. If you see any problem with the calculations or believe a different test would be more appropriate let me know.

![mempool-variance|400x264](upload://jxdrN9OpGgJ5nazn8lB8PS8BAvP.jpeg)

<hr>

## Discussion
An obvious confounding variable was **node uptime** which could affect network visability. The fullrbf node is an always-on server while the bip125 node is a pruned node on an old laptop that is off most of the time. I attempted to smooth out that variation by making sure the nodes were peered with one another giving them the opportunity to share their own mempool contents.

One of the reasons I took on this exploration was to engage in a larger discussion about the health of the p2p network. It's important that node operators have a habit of monitoring statistics to track changes in usage by network participants.

The score is intentionally "dumb". We would like it to have a value close to 1, but deviations from perfect aren't necessarily a cause for concern. Indeed we expect to see variation by virtue of the distributed network - some nodes see some transactions and other nodes see others. It would be unrealistic to expect perfection from every block - certainly that would make it less useful as a metric. Thankfully I was surprised to observe such high p2p scores on a regular basis. In contrast, witnessing large or prolonged deviations could be a sign that either **1)** local policy has fallen adrift of the wider network or **2)** significant volumes of transactions are confirming having never entered the mempool to begin with.

In terms of policy I don't take a stance on whether nodes should conform to miner practices or vice versa. I do think we should try to strike a balance between sane and reliable defaults while recognizing the need to evolve and adapt policy with the aim of making the mempool an efficient place where users will want to transact.

-------------------------

