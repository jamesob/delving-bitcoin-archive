# Sibling Eviction for v3 transactions

glozow | 2024-01-24 14:31:36 UTC | #1

When we receive a transaction that would bust a mempool transaction's descendant limits, we usually reject it (unless it is eligible for CPFP carve out).

An alternative approach (e.g. when this transaction is very high feerate and it'd be a shame to let it go) is to consider evicting the least incentive compatible descendant in favor of this new transaction. That way we don't compromise on our anti-DoS-motivated descendant limit (like we do with CPFP carve out), but don't need to be stuck with the one we received earlier.

Ephemeral Anchors emulates sibling eviction by having a "must spend" anchor to basically force all children to conflict with each other (general idea credit to instagibbs who suggested this at that LN meeting in 2022).

A few concerns then were:
(1) This is full-rbf-y (what if the sibling didn't signal RBF? What if wallets aren't expecting any possible replacements?)
(2) It's very complex to choose which descendant(s) to evict. There are many possible combinations especially if the problem is that we hit the descendant vsize limit. This transaction might also have multiple ancestors. Short of cluster mempool / feerate diagram tools, this problem seemed intractable.
(3) Is this incentive compatible?

Problems (1) and (2) are easy to fix with v3. V3 transactions all signal replaceability, so the "full-rbf-iness" shouldn't be a concern. And (2) is trivial in v3, since there is only 1 possible descendant and this child cannot have multiple ancestors (credit to sdaftuar for pointing this out to me, leading me to try to implement it).

One way of looking at (3) is that if we had just received these two descendants in the opposite order, we would have kept the other one. With the framework that our descendant limit is the maximum we can handle before we have DoS and pinning problems, we should just be trying to get the most incentive compatible set of transactions that fits in that limit.

An implementation: https://github.com/bitcoin/bitcoin/pull/29306

We apply the RBF rules to these sibling evictions, not just because we get to reuse all the code, but because there are similar requirements wrt incentive compatibility increase and limiting the bandwidth/validation cost. For example, we don't want to allow 2 siblings to evict each other over and over again with 1 additional sat each time.

### Benefits:

(1) We can easily do the imbued v3 logic for existing LN commitment transactions, even though they have 2 anchors.

There was a [suggestion](https://delvingbitcoin.org/t/lightning-transactions-with-v3-and-ephemeral-anchors/418/2?u=glozow) to pattern-match LN commitment transactions and automatically enroll them under the v3 policy rules if it takes a while for LN to switch to using v3.

One obstacle is the fact that, currently, commitment transactions have two anchors, while v3 only supports a 1-parent-1-child topology. What about implementations that try to bump remote transactions in mempool? Even if everyone stops doing that, what if your remote broadcasts and spends your tx with a low-feerate child (thus blocking you from bumping yours)?

With sibling eviction, local and remote's CPFPs (each no more than 1000vB) can evict each other. We wouldn't need to roll a new 1p2c or v3+carveout topology.

(2) It makes v3 work nicely for transactions shared between n>2 parties who might want to CPFP without a dedicated anchor. Imagine e.g. a v3 coinjoin - if somebody wants to CPFP, they can do so using their output, evicting somebody else's child if they lowballed their bump.

-------------------------

