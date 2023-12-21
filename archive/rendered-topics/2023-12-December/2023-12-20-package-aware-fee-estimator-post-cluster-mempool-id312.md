# Package aware Fee estimator post cluster mempool

ismaelsadeeq | 2023-12-20 21:38:24 UTC | #1

CBlockPolicyEstimator assumes a transaction’s confirmation status is always due to its actual fee rate. Due to the possibility that a transaction may be fee-bumping its parent, CBlockPolicyEstimator currently ignores all transactions with parents in the mempool upon addition.

This is to avoid mistakenly concluding that a transaction was mined due to its high fee rate when it has unconfirmed ancestors and was confirmed as a package relative to its ancestor score.

CBlockPolicyEstimator ignoring all transactions with a parent in the mempool is better than making wrong assumptions.

Some transactions tracked by the CBlockPolicyEstimator may have descendants and get confirmed through CPPF by a descendant’s ancestor score, not the transaction’s low fee rate. In such cases, CBlockPolicyEstimator incorrectly assumes that the transaction confirmation is due to the actual low fee rate, which is incorrect.

The CBlockPolicyEstimator has two issues:

* Ignoring some data points (all transactions that have unconfirmed parents).
* Having some incorrect data points (assuming transactions are always confirmed because of their fee rate when sometimes they are not).

This issue can potentially be solved post-cluster mempool.

The proposed solution involves tracking transactions in CBlockPolicyEstimator by their chunk mining score.

CBlockPolicyEstimator should also additionally track the block height transaction mining score is improved, and also the updated mining score.

When a transaction is added to the mempool.

* CBlockPolicyEstimator should add a new entry with its mining score

When a chunk mining score is updated CBlockPolicyEstimator should update the improved mining score of all the chunk transactions, and update all chunk transactions’ last blockHeight mining score was improved with the current block height.

If a transaction is removed from the mempool for reasons other than BLOCK.

CBlockPolicyEstimator should remove it as a failure.

Upon receiving a new block

* Categorize block transactions into clusters, linearize the clusters and get a series of chunks.
* For all chunks in the new block.
* If the fee estimator did not track any transactions in the chunk, remove the chunk transactions from the CBlockPolicyEstimator stats as failures.
  * Rationale: The fee estimator should only track what it has seen in its mempool which was broadcasted to the network to avoid gameability by miners.
  * If all the transactions in the chunk are in the CBlockPolicyEstimator mempool, and the block’s chunk mining score matches the tracked CBlockPolicyEstimator’s transactions improved mining score, remove all the chunk transactions from CBlockPolicyEstimator as successful. the blocks it takes to mine that chunk is the current block height - the block height the chunk mining score was last improved.
* If all the chunk transactions are present in the CBlockPolicyEstimator mempool but the mining score differs,
* ( I think this is only possible if a subset of the package is mined, for example, if an ancestor’s fee rate has incentivized miners enough.)
* Hence the chunk mining score should match the CBlockPolicyEstimator chunk sponsor transaction entry mining score.
  * Remove the chunks transactions from CBlockPolicyEstimator tracking stats as success, the fee rate bucket that will be marked as success is the chunk sponsor entry mining score, and the number of blocks it takes takes to mine that chunk is the current block height - the block height the chunk sponsor was added to the mempool.

Question:

1. Linearization of the new block, what if it’s different from how the miner built the block template?
2. Why will the CBlockPolicyEstimator chunk mining score differ from the blocks chunk mining score? I can think of a scenario:  Only some subset of the chunk was mined, e.g an ancestor is mined alone, or some subset of a package
3. Is it just better to track chunks?

-------------------------

