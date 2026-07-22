# RFC: Block-Range Filters (a.k.a. Hierarchical filters)

optout | 2026-07-22 09:07:55 UTC | #1

I'd like to gather some feedback on the block-range filter idea: is it worth persuing, is/were there similar attempts, is the approach promising, what are possible caveats.

## Summary

In order to optimize for the total download size when using compact block filters (BIP-158), introduce filters for a *range* of blocks, and download these initially. Download the individual block filters only if the block-range filter matches. With the savings due to block-range filters the total download size is reduced. Preliminary results indicate that download size can be reduced to under 25%.

## Description

The basic idea is that if it's possible to create a set of filters for ranges of blocks (e.g. every 256 blocks) such that the total filter size is less than the total size of block filters, then the download size can be decreased. Only for those ranges where the block-range filter indicates a match for a given scripts, are the individual block filters downloaded. From here the process is the same as with regular block filters. The total size can be lower, despite the fact that for matching ranges both the block-range and block filters are downloaded.

The block-range filters are constructed similarly to block filters (with the input being the union of the script sets from the blocks from the range).

Preliminary simulation results (with several caveats) indicate that total download size can be decreased to as low as less than 20%, for 256 block ranges, and for typical scripts with low number of transactions.

Hierarchical filters were mentioned in a discussion on Binary fuse filters (link: https://delvingbitcoin.org/t/binary-fuse-filters-as-an-alternative-to-bip-158-gcs/2428 ), but without details.

## Preliminary simulation results

Results from a simulation, with simulated data (30k blocks), with different range sizes.
The simulation scenario is downloading all the filters and blocks strictly needed to find all the transactions of a given script.
Simulation was performed with two set of scripts, one with very low transaction count (4--6, avg 5.7) and one with low transaction count (20--30, avg 26.7). Total download sizes are given in bytes ("DL").

Results:

* Total block-range filter size is decreasing with increasing range size, being at 14.8% for 256-blocks, and 3.4% for 2016-blocks (relative to the total size of block filters).
* There are download sizes with significant decrease, the lowest ones **under 20%**. For the lower transaction count scripts (5.7) the results are 18.2% (256-blocks), for the higher one it is 28.1% (256-block)
* The range size analysis shows that there is an optimal range, over which savings are cancelled. The optimal seems to be around 256 blocks range, with the size 2016 being clearly sub-optimal.

| range size | DL (txc=5.7) | DL (txc=26.7) | Total filter size |
|----|----|----|----|
| 1 | 784019591 | 798145316 | 779618251 |
| 4 | 576031753 | 592016694 | 571100158 |
| 16 | 378268000 | 398304397 | 372302356 |
| 64 | 240684804 | 275303978 | 230555023 |
| 256 | 142775403 | 224413749 | 115442497 |
| 1024 | 141354737 | 361915243 | 45448912 |
| 2016 | 208937478 | 507760131 | 26481126 |

## Optimal range size

The optimal range size is not clear, it's a tradeoff.
Benefits of larger range sizes:

* The net total block-range filter size is lower with larger range sets

Benefits of smaller range sizes:

* Lower range sizes mean less block filter to be downloaded
* More manageable block-filter sizes, very large sizes introduce practical concerns.
  The optimal range size can only be determined with measurements, with controlled assumptions.

## Further work

* Explore different filter parameters for the block-range filters (different parameters may be optimal)
* Explore the idea of optimized block filters where instead of the original script hashes the script indexes within the block-range filters are used. May be smaller (e.g. using 13 bits instead of 19).
* Perform numerical analysis on the real bitcoin blockchain

-------------------------

nothingmuch | 2026-07-22 16:01:11 UTC | #2

Bloom filters have the nice property that the filter of the union of two sets is the bitwise OR of the filters of the two sets. This property has been the basis of bloom filter trees, RAMBO etc, and later work that switched from bloom filters to more appropriate ones (since the load at the leaves vs. the root will not be balanced for the same bloom filter parameters).

The most interesting of this line of work that I'm aware of is the paper [Building Fast and Compact Sketches for Approximately Multi-Set Multi-Membership Querying](https://dl.acm.org/doi/10.1145/3448016.3452829?__cf_chl_rt_tk=5dN6ut4YhbJsrTY6McFmkNEAMeH_XnbVhB1DHviT0mY-1784731645-1.0.1.1-f41CsLJ4xrSiel6DWb8L5II6MLB9U1KiCAeLvAO4Lso). In this work, a clever partitioning scheme ensures disjointness of entries pertaining to the same key at different positions. This work is agnostic of the underlying approximate set membership query structure, I believe quotient filters to be the most promising candidate, for a number of reasons but really the choice is arbitrary and the wire encoding vs. the in memory representation of course may have very different considerations.

The main challenge with this approach is that txids can be selected maliciously to create collisions with short hashes, the aggregation/range querying does not work if each block / leaf in the tree uses a different(ly salted) hash. However, this is also where the information theoretic advantage can come from: compact block filters require $O(bn)$ queries for $n$ addresses over a range of $b$ blocks, so the false positive rate has to be designed to be lower to avoid downloading too many blocks. By designing for a different query structure the efficiency limit implied by the specific way in which Golomb coded sets are optimal for CBF can be bypassed. Addressing this does not seem insurmountable.

## Trusted Server

The simplest idea I've had for a while in relation to this bears some resemblance to LDK's rapid gossip sync: a server (or github repo) with pre-computed AMSMMQs for a balanced tree structure. In this mode, clients can begin by downloading just the root node (which would encode a coarse grained AMSMMQ structure), or any strata of intermediate nodes below it. Assuming the server is honest, true negatives allow excluding a range of blocks. A possible positive can be responded to by downloading lower levels of the tree to narrow the range further, or downloading existing block filters for specific leaves, suggesting that the optimal structure for the tree as a whole is to minimize total information downloaded by quickly discovering the minimal set of leaf/single block filters to download. Note that because the hashing will be different than CBFs, the error is independent, which provides an advantage (FP in both the AMSMMQ and CBF structure is only as likely as the product of the two independent FPRs).

In this setting a maliciously constructed txid collision would take very little work, and is a griefing attack that in the worst case forces clients to download all compact block filters *and* this auxiliary structure which would be rendered useless. This is not costless, and the server can rotate the salt as more collisions are constructed over time, so it doesn't seem insurmountable. For narrow hashes a 2nd preimage attack would also be feasible, which would allow targeting a specific user, requiring more computational work but the same on chain cost.

## P2P via probablistic verification node class

Here the txid problem becomes is harder because the network would need to agree on a salt on collisions. I don't know of a solution to this that doesn't make it possible for an attacker to force peers that serve these data to reindex the blockchain on collisions, and since it's not just a single server with its own threshold for collision tolerance, this seems like a serious resource exhaustion vector. For the purpose of discussion, let's assume a reasonable solution exists.

In this setting, several adjustments make sense:

- compute not just one tree but a forest sharing the same leaves, each with a randomized partitioning scheme per the AMSMMQ paper (so each root represents one of the random partitions, not the whole lookup structure).
- index not just scriptPubKey or prevout scriptPubkey, but also txid merkle tree for outpoint resolution, including full utreexo commitments

Suppose an uninitialized client syncs the block headers, and some of the top layers of this forest. It can then choose a block at random, and prepare a block-with-prevouts structure that is the input to such an index, and also is a necessary but insufficient condition for block validity. With utreexo data, it is further possible to validate that no double spending has occurred, assuming that data is valid.

Note that this also permits the calculation of a leaf filter structure for that block. Leaves can be randomly merged bottom up into the forest (and see the open question in the paper about group testing, it relates to how to design the arity/height, and the partitioning scheme of this structure).

Hopefully this clarifies why I thought quotient filters are the most promising - they are easily merged, resized etc.

In zkSTARKs, soundness is often amplified additively by requiring the prover to grind. Another notion is the relaxed soundness of [FlyClient](https://eprint.iacr.org/2019/226.pdf). By including a work requirement in the leaves, this index structure unforgeably costly. Weak arguments for validity construction of the filters per the FlyClient model by having the leaves commit to a Merkle trace of their constructions, and be accompanied by Merkle inclusion proofs that can be checked against the block header mMrkle roots, and are evidence of no omissions.

Since this structure is itself a form of proof of work, as new blocks are found, it that can serve as a randomness beacon that is the basis of a continual distribute recomputation of the index structure. Each worker in this computation must contribute asymptotically less than the cost of validating the whole chain, but the results can be aggregated together. Different roots in the forest would then be "heavy" index structures, selected out of the space of random subsets (corresponding to the AMSMMQ partitioning schema) which were built upon by other clients that successfully used them to spot-check the chain. This continual updating approach seems plausible as a the basis for a mitigation of the txid collision concern, but again I don't know how to do that without potentially unbounded recomputation work at the time of the adversary's choosing, though forcing the attacker to target multiple "flavors" at once (i.e. not just a forest of random partitions, but a set of forests, one forest per hash) seems like a promising direction that should impose minimal overhead.

The reason I find this interesting is that this class of clients, unlike regular light clients, could hypothetitcally be non-parasitic on the network without imposing that utreexo commitments or these data be part of the consensus rules or even served by existing full nodes. Even though it's a relaxed form of soundness, weaker than a full node in a number of ways, it is still significantly better than just heaviest chain, and lets clients progressively improve their surety about the validity of the chain tip that they accept.

The reason I no longer find this as interesting as a few years ago is that SwiftSync largely obsoletes this, even weak FlyClient soundness would introduce a significant overhead making this likely less efficient than CBF today (but less trustful), and the whole thing is likely to be obsolete in a few years by SNARGs enabled UTreeXO, i.e. O(1) CoinWitness, so the tremendous engineering effort required for such a node does not seem justified.

-------------------------

