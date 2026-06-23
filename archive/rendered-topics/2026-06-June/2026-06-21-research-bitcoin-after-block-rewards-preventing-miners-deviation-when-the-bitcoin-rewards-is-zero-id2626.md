# [Research] Bitcoin After Block Rewards : preventing miner's deviation when the Bitcoin rewards is zero

xodn348 | 2026-06-21 05:02:06 UTC | #1

Howdy.

I have written paper about Bitcoin;s security when the Bitcoin block rewards approach zero near 2140. 

**The original motivation of this paper is worrying about reducing block rewards and miner's profitability because of halving, while the price could not doubling in every four years.** 

**Problem :** 

How the Bitcoin's security changes when the Bitcoin rewards is 0 and how could we prevent the miner's deviation.

**Findings :** 

1\. Found G_t threshold which could determine whether miner's deviation is happening or not

2\. The massive deviation is not happening only because of block rewards

3\. In a pure fee-only regime, deviation can become rational even at 0.17% of transaction fees

4\. In a pure fee-only regime, base fee, fee floor and adaptive block size could help to alleviate massive deviation of miners.

My suggestion would not be a only solution fot the problem but I would be appreciate if this incur the other brilliant scientists' inspiration.

The link is in below.

 https://arxiv.org/abs/2606.05503 

Thank you.

Sincerely,

Junhyuk Lee

-------------------------

show1225 | 2026-06-23 15:19:14 UTC | #2

Dear Junhyuk Lee,

Provoked by your post, I summarized my thoughts on a separate [post](https://delvingbitcoin.org/t/addressing-the-diminishing-block-subsidy/2640).
While floor fee seems to be worth trying (as it is soft-forkable), an ultimate solution could be tail emission + EIP-1559 style fee burn and priority tipping in my opinion.

SHO

-------------------------

