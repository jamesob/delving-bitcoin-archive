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

harding | 2024-05-04 21:22:17 UTC | #3

I don't understand the point of this.  Your concern is that _most_ miners will choose centralized transaction selection, so you want to force _all_ miners in a decentralized pool to use the transactions selected by other miners?  How does that not make the problem worse?

Here's an example: assume today that 99% of all transaction selection is performed by X.  That means there will still be about one block a day that includes indepedently selected transactions---and anyone paying a high enough feerate can almost guarantee that their transaction will be included in that block (provided the tx is valid and policy acceptable).  Fees are a good solution to weak censorship: it allows the people being censored to take direct action and buy their way out of their problem.

But if 100% of miners use your described pool, they'll have to mine any transaction that was previously included in a share before they can mine any transactions of their own choosing.  So whoever can include a tx in a share first gets to decide the transaction selection for every other miner in a pool.  That creates a strong incentive to be the person who can first include a tx in a share: they can collect out-of-band fees that nobody else can collect and they can perfectly censor unwanted transactions by including junk transactions instead.

-------------------------

Fi3 | 2024-05-06 10:00:29 UTC | #4

I don't see how this can solve the issue, if you don't enforce it a bitcoin consensus level (not sure if even possible). A government can always say every miner in my country must use a pool that do transaction selection as I want. (this is true also with sv2)

-------------------------

mcelrath | 2024-05-09 15:16:43 UTC | #5

> I don’t see how this can solve the issue, if you don’t enforce it a bitcoin consensus level (not sure if even possible). A government can always say every miner in my country must use a pool that do transaction selection as I want. (this is true also with sv2)

I don't think there is any technical solution to this criticism. But miners in different jurisdictions will have different legal responsibilities, and hopefully someone will mine every transaction.

Cheers,
-- Bob

-------------------------

mcelrath | 2024-05-09 15:43:20 UTC | #6

> I don’t understand the point of this. Your concern is that *most* miners will choose centralized transaction selection, so you want to force *all* miners in a decentralized pool to use the transactions selected by other miners? How does that not make the problem worse?

By forcing miners to do tx selection, we increase the number of parties that would need to be attacked by the state in order to control what is mined. Whether or not we have deterministic tx selection, both SV2 and Braidpool are attempting to move tx selection to the individual miners instead of (a smaller number of) pools. If we succeed in getting miners to do tx selection, then this deterministic tx selection idea probably isn't necessary.

> Here’s an example: assume today that 99% of all transaction selection is performed by X. That means there will still be about one block a day that includes indepedently selected transactions

A miner with < 1% hashrate performing independent tx selection has to take a huge risk in variance - about 18% monthly on his revenue, and it gets worse for even smaller miners. In a decentralized pool that same miner would win around 1000 shares per day, with a monthly revenue variance of about 0.58%. We want independent miners to not have to take this financial risk, it's the whole point of pools in the first place.

> But if 100% of miners use your described pool, they’ll have to mine any transaction that was previously included in a share before they can mine any transactions of their own choosing.

With deterministic tx selection they would be unable to mine transactions of their own choosing at all. They must first be mined into a share in order to have consensus that they're known to all miners. (This is true with or without deterministic tx selection) Also note that Braidpool will use "full proportional" payouts, so any fees are shared with the pool. A miner can't claim high fee transactions for himself. (Note that this isn't fully decided - if you have good arguments that the miner should earn fees for himself or get an extra reward for mining a block I'd be interested in hearing it - there's a wide design space and game theory here)

>  So whoever can include a tx in a share first gets to decide the transaction selection for every other miner in a pool. That creates a strong incentive to be the person who can first include a tx in a share: they can collect out-of-band fees that nobody else can collect and they can perfectly censor unwanted transactions by including junk transactions instead.

Yes, miners can mine never-before-seen transactions into shares instead of broadcasting them to the mempool, and collect OOB fees for them if they wish. I don't really see a problem with that. As far as censoring by including junk transactions, a large enough miner can do that today anyway and it's a 51% attack. It's still a 51% attack on the share-chain. This might be an argument to limit the size of the sharechain-mempool though, it doesn't make sense to allow this sharechain-mempool to grow without bound, which would enable the attack you describe.

Cheers,
-- Bob

-------------------------

ajtowns | 2025-01-04 07:56:50 UTC | #7

[quote="harding, post:3, topic:842"]
But if 100% of miners use your described pool, they’ll have to mine any transaction that was previously included in a share before they can mine any transactions of their own choosing.
[/quote]

I think there's two ways in which this isn't true -- if your deterministic algorithm is "highest feerate txs", then by including some very high feerate txs you could be able to mine them into your block immediately just by following the deterministic algorithm. But in addition you could also allow for 5% of the block space to be free-use (similar to the old space for mining txs by priority) after having filled up 95% of the block space deterministically.

Maintaining a deterministic mempool at pool-consensus level is likely fairly complicated in practice; particularly in rolling out upgrades/changes to relay policy (eg TRUC relay, different RBF policies, new soft fork features, changes to min fee rates or limits). 

It's also likely a bit more complex if you have to be able to efficiently calculate the top of the mempool from a previous state, for dealing with beads when the braid has width greater than 1.

At ~1/1000th the difficulty of a block, with a pool at 10% hashrate, you're receiving a new bead every 6 seconds, so probably don't want to receive 2MB of new tx data every bead. Probably it would work to take `sha256(sharehash)` as a random seed and then allow 10% of beads to include 150kvB of tx data that will be incorporated into descendants, while the remaining 90% of beads can only include 5kvB of tx data -- that's about ~2MvB tx data relayed over 100 beads (which is 600s for 10% hashrate at 1/100th difficulty).

-------------------------

mcelrath | 2025-01-06 18:36:22 UTC | #8

> But in addition you could also allow for 5% of the block space to be free-use (similar to the old space for mining txs by priority) after having filled up 95% of the block space deterministically.

What is the motivation for doing this? Every tx included this way needs to be transmitted and validated with every share-chain block, so possibly as fast as every 250ms.

The entire purpose of the deterministic block template idea is to avoid having to transmit or validate any txs at all (because it's already been done in the DAG parents).

As regards censorship resistance, graffiti blocks, parasitic assets and the like, Braidpool will need to have its own tx acceptance rules (yet to be determined), which will fall somewhere between Bitcoin's P2P rules and Bitcoin's block rules, avoiding the well-known pitfalls. Personally I don't see any point in being very opinionated about acceptable txs and I don't plan to be as restrictive as OCEAN and Knots, nor do I plan to implement tools in Braidpool that make censorship "easy", whether the user wants to omit graffiti/runes/BRC-20 or prevent Russians from donating to Ukraine. We will be liberal in what we accept. Those who want to omit certain txs will have to fork the code and run a modified client (like Knots). Those who want to be even more liberal than what we choose can either fork the code to make a new instance of Braidpool or solo mine.

-------------------------

ajtowns | 2025-01-07 04:06:21 UTC | #9

[quote="mcelrath, post:8, topic:842"]
What is the motivation for doing this? Every tx included this way needs to be transmitted and validated with every share-chain block, so possibly as fast as every 250ms.
[/quote]

Providing an escape hatch for txs to be mined outside of policy, eg RSK's [recovery txs](https://github.com/bitcoin/bitcoin/pull/26348), and to provide a way for pool members to support [off-chain tx acceleration](https://mempool.space/accelerator) without requiring a central coordinator. (Perhaps it would be possible to do off-chain acceleration by revoking unredeemed pool shares though?)

[quote="mcelrath, post:8, topic:842"]
Braidpool will need to have its own tx acceptance rules (yet to be determined), which will fall somewhere between Bitcoin’s P2P rules and Bitcoin’s block rules,
[/quote]

If Braidpool were to be successful (>30% hashrate?), I'd expect bitcoin core's relay policy to match braidpool's (well, probably with a larger backlog) to minimise divergence between actual blocks and expected blocks ([ref](https://github.com/bitcoin/bitcoin/pull/30493#issuecomment-2260918779)).

-------------------------

mcelrath | 2025-01-07 06:20:49 UTC | #10

If RSK's recovery txs should be mined then they should be isstandard(). Why aren't they already?

Off-chain tx acceleration is worth consideration. Braidpool could mine such txs into its deterministic mempool and prioritize them if we can figure out how to make an out of band fee go to the pool. (A competing pool could just steal the extra fee if an apparently-unrelated tx pays high fees) But frankly this should be handled by fee-bumping mechanisms in Bitcoin itself. Paying out-of-band for mining is a bug, not a feature. You should be able to bump fees.

I agree on the divergence from P2P relay policy generally...

-------------------------

ajtowns | 2025-01-07 10:34:33 UTC | #11

RSK's txs shouldn't have been structured that way, and no longer are. But their testing failing to catch the problem before going live, and being able to contact miners directly to bypass the problem was helpful. It's been helpful in other cases too. OTOH, so long as there's someone with significant hashpower willing/able to do special cases, doesn't matter that much if one pool or even the majority of hashpower doesn't.

-------------------------

mcelrath | 2025-01-07 13:40:25 UTC | #12

I think for Braidpool, making decisions about acceptable transactions is just as complicated as doing it for Bitcoin, because it will be a decentralized network and blockchain (DAG) similar in most respects to Bitcoin. Changing policy requires upgrading all nodes and possibly a hard fork in the DAG. We'll try to make reasonable choices here but it seems unlikely that we will be able to support extremely odd transactions.

I'll give some more thought to allowing miners to include a small number of non-committed bitcoin txs in their share-chain blocks. It increases share validation and propagation time but if it's only one or two txs maybe that's acceptable.

-------------------------

