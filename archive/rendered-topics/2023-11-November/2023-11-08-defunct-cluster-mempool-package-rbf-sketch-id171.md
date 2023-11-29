# [DEFUNCT] Cluster Mempool Package RBF sketch

instagibbs | 2023-11-29 18:21:07 UTC | #1

"I really should write this on delving but" - Pieter, literally

Copied for posterity:

So I think this is roughly the idea we discussed today for post-clustermempool package-rbf processing (no subpackage validation!):

1. Compute the set OLD of all affected clusters in the mempool (any parent/conflict of the package's transactions, plus all other transactions in the same cluster(s)). This may contain multiple existing clusters.
2. Compute the set NEW that this OLD would be replaced with: equal to OLD, with the new package added to it, and all conflicts removed. Note that this may contain multiple clusters if existing stuff splits. NEW contains actually new transactions, but also stuff already in OLD that just may get rechunked.
3. Linearize all clusters in NEW.
4. "Pre-eviction": remove from NEW any actually new transaction (so in NEW but not in OLD) whose chunk feerate is below mempoolminfee.
5. Incremental feerate check: fee(NEW) - fee(OLD) >= incfeerate * size(NEW \ OLD). If not, reject the whole thing.
6. Fee-size diagram check: compare the fee-side diagram of NEW with that of OLD. If it's worse anywhere, reject the whole thing. Also do the tail feerate check here, if not subsumed by incfeerate check.
7. Verify new transactions.
8. Add to NEW all transactions from OLD that don't actually conflict anymore (due to things removed by the pre-eviction step).
9. Apply to mempool.

Writing this down now, it seems to me that doing (8) between (4) and (5) is strictly better; it's work you need to do anyway, and it can only help the (5) and (6) checks.
5:02 PM

2

Pieter Wuille
A re-linearization after (4) and/or (8) is possible too, but it's not clear how much that helps, and doing that consistently would probably imply redoing (4) again too - possibly leading to a long set of iterations.

-------------------------

sipa | 2023-11-29 18:33:53 UTC | #2

Replaced by https://delvingbitcoin.org/t/post-clustermempool-package-rbf-per-chunk-processing/190

-------------------------

