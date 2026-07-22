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

