# LIMO: combining the best parts of linearization search and merging

sipa | 2024-04-24 14:47:07 UTC | #1

# LIMO: Linearization through Incremental Merging of Optimizations

<div data-theme-toc="true"> </div>

## 1. Introduction

Consider the linearization algorithm as suggested in [How To Linearize Your Cluster](https://delvingbitcoin.org/t/how-to-linearize-your-cluster/303), excluding the approach from Section 1.2 (Bottleneck Splitting). In broad lines:
* While transactions remain in the cluster:
  * Use a computationally-bounded search algorithm which pre-splits on the best remaining ancestor set to find a good topologically valid set of transactions in what remains of the cluster.
  * Output the transactions of that topologically valid set as the next linearized transactions.
  * Remove the set from the cluster and repeat.

One might expect that this algorithm will always produce a linearization which is (by the convex-hull [feerate diagram](https://delvingbitcoin.org/t/cluster-mempool-definitions-theory/202#linearizations-and-chunks-3) metric) at least as good as straight up picking the best ancestor sets of what remains, as this is just additionally doing search on top. It turns out this is **not the case**. In fact, it may (rarely) be strictly worse.

Consider the following example cluster:

```mermaid height=234,auto
graph BT
  T0["B: 11"];
  T1["C: 7"];
  T2["D: 10"];
  T3["E: 7"];
  T4["A: 1"];
  T0 --> T4;
  T2 --> T1 --> T4;
  T3 --> T4;
```

**Ancestor-set based linearization.** The consecutive remaining best ancestor sets are AB (6), CD (8.5), and E (7), and the resulting [A,B,C,D,E] linearization is in fact optimal, chunked as [ABCD (7.25), E (7)].

**Computationally-bounded search.** However, ACDE (6.25) has higher feerate than AB (but worse than ABCD), and thus a (very) bounded search might end up with ACDE as first set to include. The resulting [A,C,D,E,B] linearization, chunked as [ACDEB (7.2)], is not optimal, and strictly worse than [A,B,C,D,E].

It is not a very satisfactory situation that an algorithm that performs strictly more work can end up with a worse solution.

## 2. Incremental merging

Of course, we have a good algorithm for combining the best parts of two linearizations already: [merging](https://delvingbitcoin.org/t/merging-incomparable-linearizations/209).

With that, it is possible to compute two separate linearizations, one using just ancestor sets, and one using bounded search, and then merging the two. But that has downsides too; either:
* We only perform the optimal ancestor set finding once, as part of the ancestor linearization, but then the bounded search cannot take advantage of this information, as it's only available during the merge at the very end.
* We perform the search for optimal ancestor sets twice (once inside the ancestor linearization, and once inside the pre-splitting during search), meaning duplicate work.

Overall, it feels like using merging to address this comes "too late". Ideally, we would incorporate the findings of ancestor sort as input to the search. This can be accomplished by turning the overall linearization algorithm into a improvement algorithm:
* Start with an initial linearization $L$, e.g. the ancestor-based linearization.
* While transactions remain in $L$:
  * Use bounded search to find a high-feerate topologically valid subset $S$ of what remains of $L$.
  * Perform an optimization step that reorders $L$ without worsening it, and such that the initial part of its diagram is at least as good as the diagram of $L[S]$.
  * Output the highest-feerate prefix of $L$ and continue with what remains.

**Definition.** The notation $L \triangleleft S$, for a linearization $L$ of graph $G$ and set $S$ is used to mean $L[S] + L[G \setminus S]$, i.e. $L$ but with the subset $S$ moved to the front.

The optimization step above can then be written as $\operatorname{merge}(L, L \triangleleft S)$. This is a strict improvement over the existing linearization algorithm, which can be seen as switching to $L \triangleleft S$ directly. In addition to guaranteeing a result that is as good as the combinations of prefixes of found subsets, the optimization step also guarantees a result that is as good as the initial linearization. And contrary to the merge-at-the-end strategy, the subset searches get to take advantage of the quality of the initial linearization too, as that affects what remains in $L$ (and more, see below).

## 3. Single-set improvement steps

The approach above requires performing a $\operatorname{merge}$ operation for every search step, which can be up to cubic in complexity, as $\operatorname{merge}$ may take up to $\mathcal{O}(n^2)$ time, and we may need to run it up to $n$ times. This would make the overall operation potentially significantly slower than just merging once at the end.

To address that, observe that the [merging algorithm](https://delvingbitcoin.org/t/merging-incomparable-linearizations/209#prefix-intersection-merging-5) itself works by incrementally moving high-feerate subsets to the front. If instead of performing a full merge in every step, we just determine what the first to-be-moved subset would be for the resulting merge, and output that before continuing with what remains of the linearization, we are back to quadratic complexity.

The result is the LIMO algorithm:

**Definition.** $\Pi(L)$ denotes the highest-feerate prefix of $L$ (i.e, its first chunk).

* Given an initial linearization $L$:
  * While there are transactions left in $L$:
    * Let $l = \Pi(L)$.
    * Find a high-feerate topologically-valid subset $S$ of the transactions in $L$ (search).
    * Let $s = \Pi(L[S])$.
    * Let $b = s$ if $\operatorname{feerate}(s) > \operatorname{feerate}(l)$; $b = \Pi(L[l] \triangleleft s)$ otherwise.
    * Append $L[b]$ to output linearization.
    * Remove $b$ from $L$ and repeat.

As long as the consecutive $S$ sets do not degrade in quality, the resulting linearization will be as good as all its combined prefixes.

For every search step an initial guess $l$ is known: the highest-feerate prefix of what remains of the initial linearization. This $l$ can be used as the initial $\operatorname{best}$ inside the [search algorithm](https://delvingbitcoin.org/t/how-to-linearize-your-cluster/303) (instead of $\varnothing$), which may allow earlier pruning of work queue items whose $\operatorname{pot}$ isn't better (see [Section 2.2](https://delvingbitcoin.org/t/how-to-linearize-your-cluster/303#h-22-potential-set-bounding-7)), and can reduce the initial size of $\operatorname{imp}$ (see [Section 2.3](https://delvingbitcoin.org/t/how-to-linearize-your-cluster/303#h-23-best-bounding-of-potentials-8)).

## 4. Improving existing linearizations

So far, we have considered LIMO as a replacement for a "cold-start" linearizations, for clusters which do not have a linearization already, or for merging as a linearization retry with an existing one. It could however also be used to improve existing linearizations, by passing in that existing linearization as initial $L$, rather than an ancestor-based linearization.

In that setting, it would be useful if the algorithm could merge in two distinct $S$ sets in every iteration, e.g. one found through ancestor-set linearization and one through search, effectively moving the ancestor-set logic into the algorithm itself rather than using it as an input.

It gets more complicated to have two sets if we want to guarantee a result that's as good as both, and as good as the initial linearization, but this appears to work (no proof, just a lot of fuzzing...). Let's call it Double LIMO:

* Given an initial linearization $L$
  * While there are transactions left in $L$:
    * Set $b = \Pi(L)$.
    * Find high-feerate topologically-valid subsets $S_1$ and $S_2$ of the transactions in $L$:
      * $S_1$ could be the best ancestor set in what remains of $L$.
      * $S_2$ could be the result of a computationally bounded search in $L$, using $b$ as a starting point. This search can be delayed until after the $S = S_1$ iteration below, which may provide a better $b$.
    * For $S \in \{S_1, S_2, S_1 \cap S_2\}$:
      * Let $s = \Pi(L[S] \triangleleft b)$ (during the first iteration it holds that $L[S] \triangleleft b = L[S]$).
      * Set $b = s$ if $\operatorname{feerate}(s) > \operatorname{feerate}(b)$; $b = \Pi(L[b] \triangleleft s)$ otherwise.
    * Append $L[b]$ to output linearization.
    * Remove $b$ from $L$ and repeat.

It appears this algorithm even generalizes to higher numbers. E.g. Triple LIMO would involve three subsets and $S$ would loop over their 7 non-empty intersections $\{S_1, S_2, S_1 \cap S_2, S_3, S_1 \cap S_3, S_2 \cap S_3, S_1 \cap S_2 \cap S_3\}$.

Double LIMO can be used in cluster update situations:
* Start with an existing linearzation $L$ for a cluster.
* With a new transaction/package coming it, remove from $L$ the conflicts and append (at the end) the replacements, leaving the order otherwise the same.
* [Post-process](https://delvingbitcoin.org/t/linearization-post-processing-o-n-2-fancy-chunking/201) $L$.
* Perform Double LIMO on $L$ (with $S_1 =$ best ancestor set, $S_2 =$ bounded search result).
* Maybe post-process again?

The result will be at least as good as ancestor sort, at least as good as the combination of prefixes found by bounded search, and at least as good as the result of post-processing the remainder of the original linearization after replacements, all while letting the ancestor sort and bounded search operate on the best state so far.

## Acknowledgements

Thanks to @ajtowns for the notational suggestions.

-------------------------

instagibbs | 2024-04-24 21:48:53 UTC | #2

Is running this over the non-empty `S_i` intersections crucial to this strategy or "just" an optimization that takes advantage of multiple searches finding interesting structure in some combined way?

-------------------------

sipa | 2024-04-24 23:21:27 UTC | #3

It appears to be necessary to process these intersections too, though I don't exactly understand why.

I have a fuzz test that implements Double and Triple LIMO, with the $S_i$ sets calculated as "whatever remains of $n$ static fuzz-derived topologically-valid sets", on a fuzz-derived initial linearization for a fuzz-derived cluster, and then verifies:
* The resulting linearization is topological
* The resulting linearization's diagram is at least as good as the initial one.
* The resulting feerate diagram at size $\operatorname{size}(S_i)$ is at least $\operatorname{fee}(S_i)$, for each $i$.

If I drop any of the $2^n-1$ intersections, the test finds failures in the last condition.

-------------------------

ajtowns | 2024-04-25 01:11:42 UTC | #4

I wonder if doing the intersections first would be an optimisation? ie $S_1, S_1 \cap S_2, S_2, S_1 \cap S_2 \cap S_3, S_1 \cap S_3, S_2 \cap S_3, S_3$

-------------------------

sipa | 2024-04-25 10:34:41 UTC | #5

Interestingly not all orderings work. These do not (will update if I find more):
* $S_1 \cap S_2, S_1 \cap S_3, S_2 \cap S_3, S_2, S_3, S_1 \cap S_2 \cap S_3, S_1$

-------------------------

sipa | 2024-04-25 12:14:15 UTC | #6

Huh, much simpler construction that also seems to work:

* Given an initial linearization $L$
  * While there are transactions left in L:
     * Compute high-feerate topologically-valid sets $S_1$ and $S_2$.
     * Let $b$ be the highest-feerate set in:
       * $\Pi(Q)$ for $Q \in \{L, L[S_1], L[S_2], L[S_1 \cap S_2]\}$
    * Append $L[b]$ to output linearization.
    * Remove $b$ from $L$ and repeat.

(also generalizes to triple variant)

-------------------------

