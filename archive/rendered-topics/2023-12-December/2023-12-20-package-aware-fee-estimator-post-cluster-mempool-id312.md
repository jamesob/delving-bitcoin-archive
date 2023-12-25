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

glozow | 2023-12-21 11:11:07 UTC | #2

> is it just better to track chunks?

My mental model for package aware fee estimation is that the fee estimator essentially tracks "bids" consisting of block height + feerate, and block processing sees which bids succeeded in how many blocks. A CPFP is a new bid that overrides the original one, though the original tx could theoretically be mined by itself, so I guess the hard part is checking which bids were "accepted."

Theoretically, a chunk can represent a bid, i.e. count as 1 data point. In the case of, say, an ephemeral anchors commitment transaction + sponsor, the idea is the transactions are inseparable and the package can be treated as a single transaction. All CPFPs, including batched ones, are like new bid for 1 big transaction at the chunk feerate.

And in practice, most clusters are probably 1 transaction, and most chunks will probably be mined in tact (?).

So yes, maybe just track chunks? Or I guess you could track chunks + the individual transactions, and only use exact matches after you linearize the block?

-------------------------

sipa | 2023-12-22 19:37:48 UTC | #3

Interesting!

When suggesting the "fee estimation should use chunk feerates", I had imagined it would just use our mempool's clusters/chunks to determine those values.

But you're right to point out that clustering/linearizing/chunking the block itself may yield different values. When cluster inclusion in a block doesn't match "all chunks up to x" for some chunk x, the chunk feerates can differ. I'm not sure what to do about that.
* The miner may simply not have had the same mempool as us, and it might be incorrect to use our own prediction for their behavior.
* On the other hand, we also shouldn't introduce a means for miners to influence our estimates unfairly - if they can cause our estimates to be off (significantly) by excluding some transactions, that may be an issue.
* As you point out, we may not be able to discern the clusters/chunks miners used. Our (re)linearization of the block's clusters may differ from their (it could be worse, but it could be better too). And at least initially post-cluster-mempool we may make mistakes by treating blocks as being cluster-mempool-based while miners use older or custom logic.

I think overall, it's probably best to initially use our own (mempool) chunk feerate estimates (at the time the block is found). Our mempool is our best prediction of what the next block will be, so we should assume miners have access to it even though that's not always exactly true.

-------------------------

ismaelsadeeq | 2023-12-23 11:10:35 UTC | #4

> When suggesting the “fee estimation should use chunk feerates”, I had imagined it would just use our mempool’s clusters/chunks to determine those values.

Yes

> I guess you could track chunks + the individual transactions, and only use exact matches after you linearize the block?

I agree and think that's what's better to do and the safest.

So we should track chunks, individual transactions are also chunks with single transactions.

Whenever we receive a new block we linearize the block to get a series of chunks.

If there is an exact match for a chunk of transactions and its fee rate, remove it as a success.

Otherwise, remove the chunk transactions from the fee estimator mapMempoolTxs and blockIndex.

The mempool should notify the fee estimator about new chunks, updated chunks, and chunk eviction.


Reasons why chunk fee rate will not match in some cases might be due to 

* Mempool differences
* Differences in linearization algorithm
* When a subset of the package is mined

I think it's best to ignore if there is a mismatch in all the cases above.

This way we are on safe side by relying on our mempool just as sipa mentioned.

-------------------------

sipa | 2023-12-23 14:25:17 UTC | #5

I don't think we need to cluster/linearize/chunk the received block;  the transactions in the block can be looked up one by one in the mempool to find their chunk feerate?

-------------------------

ajtowns | 2023-12-24 11:14:22 UTC | #6

[quote="ismaelsadeeq, post:4, topic:312"]
So we should track chunks, individual transactions are also chunks with single transactions.
[/quote]

My understanding is that we model "how many blocks does it take for a tx at this fee rate to be confirmed", and we count from when a tx entered the mempool until it was confirmed in a block. But if we have a CPFP where tx C pays for tx P, we might initially have a chunk [P] in the mempool at 50sat/vb, then later a chunk [P,C] in the mempool at 90sat/vb. If [P,C] then confirms, presumably we count how many blocks since we say P. But what if C gets RBF'ed by some unrelated tx D, and then P eventually confirms on its own at 50sat/vb? Do we bump the count back up to however long P has been sitting in the mempool?

I think maybe you could do a rule like:
 * for every chunk in our mempool that had some txs included in the block
 * if some tx in the chunk was not in the block, nevermind; ignore this for fee estimation
 * otherwise, use the feerate of the chunk, and the time-in-mempool of the youngest tx in chunk, and add that info to the fee estimation

-------------------------

ismaelsadeeq | 2023-12-24 13:22:00 UTC | #7

>I don’t think we need to cluster/linearize/chunk the received block; the transactions in the block can be looked up one by one in the mempool to find their chunk feerate?

Yes, not even CTxMempool but the fee estimator Internal MapMempoolTxs.

>Do we bump the count back up to however long P has been sitting in the mempool?

I am assuming For each chunk, the time-in mempool to be the time of entry of the sponsor of the chunk (which should be the youngest transaction in the chunk) - the block height the chunk was mined.

I the case you mentioned the block height  P entered the mempool - block height P was mined.

-------------------------

ajtowns | 2023-12-25 10:42:32 UTC | #8

I think you could have an arrangement:

 * T=0600 chunk: A, feerate 20 sat/vb
 * T=0800 chunk: A B, feerate 50 sat/vb
 * T=1000 chunk: A B C, feerate 80 sat/vb
 * T=1500 chunks: [A D] feerate 100 sat/vb, [B] feerate 90 sat/vb, [C] feerate 85 sat/vb

That is, B C and D all CPFP A, but D does so at a much higher feerate than B or C.

In that case B and C have been in the mempool since 0800 and 1000 respectively, but should perhaps only be considered to have been in the mempool at 90sat/vb and 85sat/vb since T=1500.

-------------------------

ismaelsadeeq | 2023-12-25 12:25:27 UTC | #9

Interesting, this will happen when the cluster is linearized after the entry of D, which will lead to the creating of chunks [B] and [C].

So instead tracking the entry time of the sponsor of the chunk, I could just use the time the chunk fee rate was last updated, 
* Which will represent 1500 for both chunk [B] and [C]. Since 1500 is the time the chunk was created/it represent to the time it was last updated.
* For [A, D] also 1500 is the time the chunk fee rate was last updated.

-------------------------

