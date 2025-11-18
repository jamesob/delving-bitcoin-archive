# Generalizing RBF under Cluster Mempool

josh | 2025-11-18 20:41:14 UTC | #1

Hi all,

I've been learning about RBF under the cluster mempool proposal, and I'm trying to better understand what developers mean by "incentive compatibility."

My understanding, from reading [An overview of the cluster mempool proposal](https://delvingbitcoin.org/t/an-overview-of-the-cluster-mempool-proposal/393), is that cluster mempool is designed to maximize the total fees of the entire mempool, at all mempool sizes, while maintaining a total ordering of clusters. I find this compelling, but one comment still bothers me:

> **Note**: To guard against free relay (an anti-DoS concern), we still will require that the total fee of the new transaction exceed the total fee of the conflicting transactions by at least as much as the min-relay-feerate * size of the new transaction. However this is a small effect, since the incentive compatibility rule described above already requires that the total fee cannot go down – this just bumps that value up slightly.

What I'm struggling with is the last line, implying that it is always incentive compatible to prevent the total fees of the existing mempool from declining. Is this truly the case?

### RBF, RBFr, and Free Relay

Over the weekend, @niftynei was telling me about Peter Todd's [RBFr proposal](https://petertodd.org/2024/one-shot-replace-by-fee-rate) (shoutout to *bitcoin++ NC co-hosted with @vnprc!). We debated the free relay / DOS concerns, and I've been thinking about how cluster mempool could resolve them.

At a high-level, I have two questions:

> **Question 1:** What is the principal concern about free relay from RBFr (replace-by-fee-rate)?  That the submitting user will physically "spend" fewer sats? That the total fees of the mempool will decline? Or that miners will insufficiently profit to warrant relay over the P2P network?

> **Question 2:** If mining is competitive and miners are greedy, would it ever be incentivize compatible to intentionally mine on a less profitable block?

Presumably, the answer to (2) is no, subject to the technical constraints of broadcasting over the P2P network.

Regarding (1), I imagine that what matters most to the network is that miners receive a minimum additional profit from a replacement transaction broadcast over the P2P network, *in all future states of the mempool*. This raises the question:

> **Question 3:** Under cluster mempool, can we prove the existence of a minimum marginal profit $P_{min}$ over any block that contains a replacement transaction $T_r$ instead of $T$, where $T_r$ pays a higher fee rate but has fewer fees (let $P_{min}$ = `min-relay-feerate` * size of $T_r$).

I'm curious to hear how developers think about this question. My inclination is to think that with the total ordering of cluster mempool, we may be able to safely relay the replacement transaction $T_r$, by considering clusters that are guaranteed to never be in the same block as $T$ and which fit within the blockspace opened up by the replacement.

### Formal Model

Formally, let $\Delta B = B - B_r$ and $\Delta F = F - F_r$, where $B$ and $F$ are the virtual size and fee of $T$, and $B_r$ and $F_r$ are the virtual size and fee of $T_r$.

Furthermore, Let $C_i$ by the *ith* cluster, as ordered by the cluster mempool, following $T$, and let $C_k$ be the first cluster that is guaranteed to not be in the same block as $T$, because $T$ plus clusters $i < k$ reaches the 4M WU block size limit.

Finally, let's assume that there exists a set of filling clusters $C_j$, $C_{j+1}$, etc. such that $j \geq k$, the virtual size of the clusters is $B_C \leq \Delta B$, and the total fee of the clusters is $F_C \geq \Delta F + P_{min}$, where $P_{min}$ is `min-relay-feerate` * $B_r$.

Under these definitions, and under the assumption that mining is competitive, we can safely say that replacing $T$ with $T_r$ always clears the profitability required for relay. In any block containing $T$, we can mine a more profitable block containing $T_r$, because we can provably fill the extra blockspace with transactions that increase the total fee by at least $P_{min}$.

Alternatively, if we cannot find a set of filling clusters $C_j$, $C_{j+1}$, etc. that satisfy these constraints, we cannot replace $T$ with $T_r$.

### Potential Issues

One potential issue is that the search for the filling clusters $C_j$, $C_{j+1}$, etc. can be computationally costly, which is a potential attack vector if the search is performed by nodes on the network. This is a linear search starting with cluster $C_k$, and in the worst case, we search the entire mempool and fail to find a filling set of clusters, making the search $O(n)$ w.r.t. mempool size.

One solution is to push the cost of search and selection onto the user, by requiring that the filling clusters be packaged with the replacement transaction $T_r$ during relay. Alternatively, users could include the txids of those clusters, and the receiving node could look them up. If the node lacks those transactions (or the transactions in clusters $i < k$), the user will need to relay those transactions first.

A second potential issue is that an ignorant or irrational miner could mine transactions in clusters $i \geq 0$ without mining $T$ or $T_r$, such that $T_r$ is no longer more profitable than $T$. Presumably, the risk of this happening would be low, but it is worth acknowledging. What perhaps matters most is that the replacement is sufficiently profitable in expectation, providing the same DOS protection for relaying nodes as existing RBF.

Finally, given these considerations, defining $P_{min}$ as `min-relay-feerate` * $B_r$ may not be costly enough. We should also account for the cost of relaying the filling txids. This would make $P_{min}$ equal to `min-relay-feerate` * $(B_r + 32 \cdot N)$, where $N$ is the number of filling txids.

### Incentive Compatible?

This approach to a generalized RBF is an attempt on my part to better understand the constraints of the free relay problem and what a strictly incentive-compatible RBF would look like.

In summary, what I currently think is incentive compatible appears to differ slightly from the current approach of the cluster mempool:

> **Current approach:** Incentive compatibility maximizes the total fees over the mempool, at all mempool sizes.

> **Alternative approach:** Incentive compatibility maximizes the total fee of any block that contains transaction $T$.

These two approaches unify in the edge case where the block size is unbounded, making the second approach perhaps a generalization of the first.

Appreciate feedback on this concept, and looking forward to seeing cluster mempool in the next version of Core!

-------------------------

