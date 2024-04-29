# Deterministic tx selection for censorship resistance

mcelrath | 2024-04-29 12:26:28 UTC | #1

One of my fears is that the risk and liability of transaction selection is so bad that no one will want to do it, leading to full centralization around the block template providers. We may already be there. Matt made a lot of hay about this in his recent TFTC interview and he's right. The thesis that we can achieve decentralization by pool hopping may be false if there's only one tx set being used by pools, or if there are too few pools to hop to, and having many pools has a negative consequence in terms of variance reduction. Tx selection is an all downside and no upside problem. Every jurisdiction has someone they don't like, whether it be political rivals, business rivals, criminals or freedom fighters. If Bitcoin is to be censorship resistant, we're failing.

An alternative is to have a deterministic algorithm select transactions. The reason we can't do this today is that we do not all have the same set of txs in our mempool, we have no consensus on the contents of the mempool, and miners can and do introduce txs that have never been seen by anyone in their blocks. 

But if we did have consensus on the mempool, it's straightforward to use a deterministic algorithm to select a subset of those txs.

Step one is to have a committed mempool with consensus. This can be accomplished fairly straightforwardly in a decentralized mining pool like P2Pool or Braidpool by adding a metadata block to the weak blocks ("shares") used by the pool containing transactions. These metadata "mempool" txs must not conflict with other txs committed to the mempool in ancestor shares, and you can regard this "mempool" as a single large self-consistent block with no block size restriction.

Once we have this committed tx set, every weak block in the share chain defines its own mempool in its ancestors by following the highest work-weighted path to the previous Bitcoin block. (This definition is valid both for blockchain-based P2Pool and DAG-based Braidpool).

One advantage of this approach is that by defining the deterministic block template algorithm, it's no longer necessary to actually communicate the block template to sharechain nodes since it can be independently computed, vastly speeding up share/block validation and reducing communications bandwidth to be basically equivalent to today's mempool bandwidth plus the weak block shares, which would add a few hundred bytes about once a second. 

Now this doesn't fully solve the problem of transaction censorship, but rather moves it to the share chain. If you want to prevent a tx from being mined in the main chain, you now have to prevent it from being mined in the share chain. However being mined in the share chain is not finalization. The UTXO set of the parent chain has not updated and in a double spend can still occur. In a DAG this happens via diamond-type graphs with conflicting txs on opposite sides - the algorithm must follow the highest work-weighted path and will pick up only one side of the diamond, but work can be added to the opposite side of the diamond to effect a change in the highest work-weighted path. In a P2Pool-like chain, the same can occur by orphaning a share chain block.

Cheers 
- Bob

-------------------------

evoskuil | 2024-04-29 13:45:50 UTC | #2

I remember around the time sv2 was announced, you and I discussing the failure of pool hopping in the face of state threats. As a result of overselling this idea, sv2 likely delayed investment in potential solutions or at least actual improvements.

I don’t know if tx pool consistency can solve the problem, but small scale miners operating independently and competitively is certainly the ideal. I’ve been hopeful ever since you introduced braids at scaling HK. It’s good to see that it is finally getting some amount of investment.

-------------------------

