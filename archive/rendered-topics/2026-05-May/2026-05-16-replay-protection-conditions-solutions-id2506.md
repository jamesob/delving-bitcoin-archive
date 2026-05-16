# Replay protection conditions/solutions

Kruw | 2026-05-16 13:11:08 UTC | #1

Some nodes are signaling support for a [soft fork,](https://github.com/bitcoin/bips/blob/6deafd07ff5d38c7d69b27530ba55a4ee038cde7/bip-0110.mediawiki) which will activate via BIP8 at the end of summer. This creates an opportunity for anyone holding Bitcoin to sell their coins that exist on one side of the fork (and double down on their decision by using the proceeds to purchase more coins on their preferred chain).\* For the sake of simplicity, I will refer to the post-split chains as “Core” (for the original rule set) and “Knots” (for the restricted rule set), even though this terminology is not completely inclusive or exclusive.

However, a chain split introduces a footgun: Transactions you sign that spend coins on one network are valid on [u]both[/u] chains. This would allow the recipient of a payment you made on the Core chain to rebroadcast it on the Knots network, draining the coins from your wallet there as well. Similarly, if you spend your coins on the Knots chain, then the recipient can rebroadcast that transaction on the Core network and take your coins on the main chain. This is called a *replay attack.*

*Replay protection* is the term for the various strategies to make your signatures [u]only[/u] valid on the chain you intend to spend on. For the Knots chain, the only way to do this is to wait for miners to create new blocks and obtain a descendant of the newly generated coinbase output.\*\* Because the same block does not exist on both chains, you can finally safely spend your existing UTXOs by combining the miner-descended coin as an input for that tx.

On the Core chain, it is much simpler to gain replay protection. Anyone can just create a transaction that is invalid under BIP110’s new rules, and all descendant outputs from that tx (and all their future descendants, etc) can’t be replayed on the Knots chain. Participating in a coinjoin transaction makes it easy to gain replay protection without acquiring new coins, since all rounds will eventually contain at least one non-replayable input as an ancestor.

*\*Assuming the Core chain has >51% of the total hash rate. If the Knots chain gains >51% of the hash rate, the Core chain will be reorged out of existence and there won’t be a split.*
*\*\*Miners are forced to wait 100 blocks before spending their new coins. If the Knots chain does not include a change to the mining difficulty mechanism, chain progression may stall entirely.
\________________________________________________________________________\_*

Is this summary correct?

-------------------------

