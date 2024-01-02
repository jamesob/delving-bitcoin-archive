# V3 transaction policy for anti-pinning

AntoineP | 2024-01-02 09:36:47 UTC | #1

"V3" is an opt-in constrained transaction relay policy proposed by Gloria Zhao to mitigate pinning. It is often discussed in conjunction with Greg Sanders' "[ephemeral anchors](https://github.com/instagibbs/bips/blob/7d79c5692bb745bf158f2d8f8e4979d80ad07e58/bip-ephemeralanchors.mediawiki)". With the upcoming phasing out of the mailing list, i'm opening this thread to discuss v3 at a high level.

Ressources:
- Original mailing list thread: https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-September/020937.html
- Original gist with discussions which led to the above mailing list post: https://gist.github.com/glozow/25d9662c52453bd08b4b4b1d3783b9ff
- Pull request to Bitcoin Core implementing the "version 3" policy: https://github.com/bitcoin/bitcoin/pull/28948. This PR contains the v3 [design specifications](https://github.com/glozow/bitcoin/blob/1dd62c3df4856c36bfc610f700684852772dd9f7/doc/policy/version3_transactions.md).
- Where it fits in the bigger package relay picture: https://github.com/bitcoin/bitcoin/issues/27463

-------------------------

AntoineP | 2024-01-02 09:42:08 UTC | #2

For context Peter Todd recently wrote a [blog post](https://petertodd.org/2023/v3-transactions-review) with the following conclusion:
> V3 transactions should not be be shipped at the moment. It has no compelling use-cases right now without ephemeral outputs, which should be discouraged, and in its current form is still vulnerable to transaction pinning.

Discussion about it [is happening on the Bitcoin Core PR](https://github.com/bitcoin/bitcoin/pull/28948#issuecomment-1873490509), and i think this thread is a more appropriate forum to discuss the merits of the v3 proposal at a high level.

-------------------------

glozow | 2024-01-02 11:44:31 UTC | #3

Thanks, I've linked this in the PR description

-------------------------

instagibbs | 2024-01-02 19:20:02 UTC | #4

moving discussion here...

> Again, I am not denying that there is a 100x reduction in the hypothetical situation of transaction pinning. What I am pointing out is that Lightning already fixed that transaction pinning problem with the design of anchor channels, making your 100x reduction irrelevant.

If an adversary splits the view of which commitment transaction is broadcasted, the remote copy can become the pin since the defender is unable to propagate their spend of the remote commit tx anchor(as their peers do not have the remote commit tx)

> But *even that* is actually incorrect: the highest feerate you could ever possibly need is the one where all of your balance in the channel goes to fees.

I don't think that's correct. As a simplified motivating example, if we set reserve amounts to 0, if you want to send "all" your funds, your top feerate will be zero, even if the total funds recoverable in timeout would be all your remaining liquidity.

I'm sure there are ways to mitigate this, like capping total HTLC exposure as a % of liquidity, but it's pretty clearly a reduction in functionality?

-------------------------

