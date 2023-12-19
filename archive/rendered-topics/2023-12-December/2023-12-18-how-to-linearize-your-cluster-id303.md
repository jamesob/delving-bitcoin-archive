# How to linearize your cluster

sipa | 2023-12-19 00:45:56 UTC | #1

# How to linearize your cluster

Most transaction clusters are small. At least today, the majority consist of just a single transaction, and those which aren't usually have no more than a few transactions.  We plan to set the cluster size limit so that even at the limit, the ancestor-set based linearization algorithm completes in a reasonable time. Yet, most clusters will be smaller, and better algorithms can be used.

Here I build up an algorithm that eventually finds the optimal linearization. It can be run with a computation limit, in which case it'll find something at least as good as ancestor-set based linearization.

<div data-theme-toc="true"> </div>

## 1. Linearization overall

The most high-level description for pretty much any cluster linearization algorithm is:
* While there are remaining transactions:
  * Find a high-feerate subset of the remaining transactions in the cluster (or ideally, the highest-feerate)
  * Sort that subset according to some topologically valid order (doesn't matter which one, so e.g. sorting by number of unconfirmed ancestors suffices), append those transactions to the output, and remove them from the cluster.
  * Continue with the remainder of the cluster.
* Optionally run a post-processing algorithm on the output, like [this one](https://delvingbitcoin.org/t/linearization-post-processing-o-n-2-fancy-chunking/201/8).

Almost all the complexity (both in the computational sense and the implementation complexity sense) is in the "find high-feerate subset" algorithm. If we instantiate that with "pick highest-feerate ancestor set", we get ancestor-set based linearization. The next section will go into finding better subsets, but first there are a few high-level improvements possible to the overall algorithm.

In practice of course transactions aren't removed from the cluster, but instead of set of remaining transactions is kept, and all operations (connectivity checks, ancestor sets, descendant sets, ...) only care about the part of the cluster that remains. For readability we drop the $_G$ index to $\operatorname{anc}()$ and $\operatorname{desc}()$; it is always implicitly the part of the cluster that remains.

### 1.1 Splitting in connected components

A cluster is (by definition) always connected, but it need not remain connected once some subset of transactions have been included. For example:

```mermaid height=234,auto
graph BT
  A["A: 5"];
  B["B: 1 "] --> A;
  C["C: 4"] --> B;
  D["D: 2"] --> A;
  E["E: 3"] --> D;
```

The highest-feerate subset is [A], but once that is included the cluster breaks apart into two components.

Whenever the remainder of the cluster consists of multiple components, it is possible to run the linearization algorithm recursively on those components separately, and then merge them (by chunking  and merge-sorting the chunks).

### 1.2 Bottleneck splitting

Given a cluster $G$, define the set of its bottlenecks as

$$
B = \bigcap_{x \in S} \operatorname{anc}_G(x) \cup \operatorname{desc}_G(x) 
$$

These are the transactions that are either a descendant or an ancestor of every other transaction in the cluster. These transactions must be included in a fixed order, and by doing so, they partition the set into separate groups that can be linearized separately. For example

```mermaid height=167,auto
graph RL
  A;
  B --> A;
  C --> B; C --> D;
  D --> A;
  E --> D;
 
  F --> C; F --> E;
  G --> F;
  H --> F;
  I --> G; I --> H;
```

In this example, A, F, and I are bottleneck transactions. If there is a single root which everything descends from, or a single leaf that descends from everything, these will necessarily be bottlenecks, but the concept is more general and can include inner transactions too, like F above.

Bottleneck splitting consists of computing bottlenecks, and then linearizing the parts between them separately, and then combining them by concatenation. In a way, bottleneck splitting is the serial analogue of the parallel connected-component splitting. Here it would amount to invoking linearization recursively for BCDE and GH, and then outputting [A] + lin(BCDE) + [F] + lin(GH) + [I].

I'm not convinced bottleneck splitting is worth it as an optimization, as it only seems to help with clusters that are already relatively easy to linearize by what follows.

## 2. Finding high-feerate subsets

The bulk of the work is in the internal algorithm to find high-feerate subsets (or ideally, *the* highest-feerate subset). I conjecture that finding the highest-feerate subset is an NP-hard problem, so we're very likely limited to small (remainders of) clusters, or approximations.

### 2.1 Searching

Overall, the search for high-feerate subsets of a given (remainder of a) cluster $G$ follows an approach where a set of work items is maintained, each of which corresponds to some definitely-included transactions, some definitely-excluded transactions, and some undecided transactions. Then a processing loop follows which in every iteration "splits" one work item in two: one where the transaction becomes included, and one where the transaction becomes excluded.

* Let $W = \{(\emptyset,\emptyset)\}$, the set of work items, initialized with a single element $(\emptyset,\emptyset)$.
  * Each work item $(inc,exc)$ consists of two non-overlapping sets; $inc$ represents transactions that have to be included, and $exc$ represents transactions that cannot be included. Transactions that are not in either are undecided. $inc$ always includes its own ancestors $(\operatorname{anc}(inc)=inc)$, while $exc$ always includes its own descendants $(\operatorname{desc}(exc)=exc).$
  * The initial item $(\emptyset,\emptyset)$ represents "everything undecided".
* Set $best = \emptyset$, the best subset seen so far.
* While $W$ is non-empty and computation limit is not reached:
  * Take some work item $(inc, exc)$ out of $W$.
  * Find a transaction $t$ not in $inc$ and not in $exc$ (an undecided one).
  * Let $work_{add} = (inc \cup \operatorname{anc}(t), exc)$, the work item for $t$ being included.
  * Let $work_{del} = (inc, exc \cup \operatorname{desc}(t))$, the work item for $t$ being excluded.
  * For each $(inc_{new}, exc_{new}) \in \{work_{add}, work_{del}\}$:
    * If $\operatorname{feerate}(inc_{new}) > \operatorname{feerate}(best)$ (or $best = \emptyset$): set $best = inc_{new}$.
    * If there are undecided transactions left corresponding to $(inc_{new}, exc_{new})$:
      * Add $(inc_{new}, exc_{new})$ to $W$.
* Return $best$

Regardless of the choice of element to take out of $W$, or the choice of undecided transaction $t$ within it, this will iterate over all valid topological subsets of $G$, and thus put in $best$ the actual best subset.

It would be possible to restrict the choice of undecided transaction to only consider ones that share ancestry with $inc$ (if non-empty). Doing so will make the algorithm only consider *connected* subsets, and we know at least one connected highest-feerate subset always exists. This would result in a moderate speedup as it reduces the search space, but interferes with a much more important improvement later.

### 2.2 Bounding through potential sets

To avoid iterating over literally every topological subset, we can compute a conservative upper bound on how good (the evolution of) each work item can get. If that is not better than $best$, the item can be discarded.

This conservative upper bound is the *potential set* $pot$, which we will compute for every work item. For a given work item $(inc, exc)$, $pot$ is the highest-feerate set among *all* sets (not just topologically valid ones) that are compatible with $inc$ and $exc$: $inc \subset pot$ and $exc \cap pot = \emptyset$. This is easy to compute:
* Initialize $pot = inc$.
* For each $u$ not in $pot$ or $exc$, in decreasing individual feerate order:
  * If $\operatorname{feerate}(u) > \operatorname{feerate}(pot)$: set $pot = pot \cup \{u\}$.
  * Otherwise, stop iterating.

Observe that all elements of $(pot \setminus inc)$ have a strictly higher feerate than $pot$ itself (it's true for the last one added, and all previously-added ones have an even higher feerate), and all undecided elements not in $pot$ have a feerate not exceeding $pot$ (if they did, they'd have been included). Thus, adding any other undecided transactions to $pot$, or removing any non-$inc$ transactions from it, or any combination thereof, decreases its feerate. Therefore, it must be a maximum.

Incorporating this into the search algorithm we get:

* Let $W = \{(\emptyset,\emptyset)\}$.
* Set $best = \emptyset$.
* While $W$ is non-empty and computation limit is not reached:
  * Take some work item $(inc, exc)$ out of $W$.
  * Set $pot = inc$.
  * For each $u$ not in $pot$ or $exc$, in decreasing individual feerate order:
    * If $\operatorname{feerate}(u) > \operatorname{feerate}(pot)$: set $pot = pot \cup \{u\}$.
    * Otherwise, stop iterating.
  * If $\operatorname{feerate}(pot) > \operatorname{feerate}(best)$:
    * Find a transaction $t$ not in $inc$ and not in $exc$ to split on.
    * Let $work_{add} = (inc \cup \operatorname{anc}(t), exc)$.
    * Let $work_{del} = (inc, exc \cup \operatorname{desc}(t))$.
  * For each $(inc_{new}, exc_{new}) \in \{work_{add}, work_{del}\}$:
    * If $\operatorname{feerate}(inc_{new}) > \operatorname{feerate}(best)$ or $best = \emptyset$: set $best = inc_{new}$.
    * If there are undecided transactions left (not in $inc_{new}$ or $exc_{new}$):
      * Add $(inc_{new}, exc_{new})$ to $W$.
* Return $best$

This change helps the average case, but not the worst case, as it's always possible that the optimal subset is only found in the last iteration. However, it's a necessary preparation for the next improvement, which very much improve the worst case.

### 2.3 Jumping ahead

The potential set, as introduced in the previous section, has an important property: its non-$inc$ transactions each have a higher feerate than the highest-feerate set possible (compatible with the work item's $inc$ and $exc$), even when ignoring topology. Thus, if one is given a compatible set which lacks one or more transactions in $(pot \setminus inc)$, then adding those transactions will *always* be an improvement to the feerate.

This implies that if $pot$ contains any topologically-valid subset, that entire subset can be added to $inc$ as well. This works because regardless of what this work item evolves into (by including or excluding elements in its undecided set), adding a subset of $pot$ will always be an improvement. This effectively lets us jump ahead, by (possibly) including multiple transactions automatically without needing to split on each individually.

We also move the computation of $pot$ and updating of $best$ inside the addition loop, as we want to perform the jumping as soon as possible and update $best$ accordingly. This can replace the "if there are undecided transactions left" test, as a lack of undecided transactions implies $pot = inc$:

* Let $W = \{(\emptyset,\emptyset)\}$.
* Set $best = \emptyset$.
* While $W$ is non-empty and computation limit is not reached:
  * Take some work item $(inc, exc)$ out of $W$.
  * Find a transaction $t$ not in $inc$ or $exc$ to split on; this must exist.
  * Let $work_{add} = (inc \cup \operatorname{anc}(t), exc)$.
  * Let $work_{del} = (inc, exc \cup \operatorname{desc}(t))$.
  * For each $(inc_{new}, exc_{new}) \in \{work_{add}, work_{del}\}$:
    * Set $pot_{new} = inc_{new}$.
    * For each $u$ not in $pot_{new}$ or $exc_{new}$, in decreasing individual feerate order:
      * If $\operatorname{feerate}(u) > \operatorname{feerate}(pot_{new})$: set $pot_{new} = pot_{new} \cup \{u\}$.
      * Otherwise, stop iterating.
    * For every transaction $p \in (pot_{new} \setminus inc_{new})$:
      * If $\operatorname{anc}(p) \subset pot_{new}$: set $inc_{new} = inc_{new} \cup \operatorname{anc}(p)$.
    * If $\operatorname{feerate}(inc_{new}) > \operatorname{feerate}(best)$ or $best = \emptyset$: set $best = inc_{new}$.
    * If $\operatorname{feerate}(pot_{new}) > \operatorname{feerate}(best)$: add $(inc_{new}, exc_{new})$ to $W$.
* Return $best$

### 2.4 Choosing the transaction to split on

One thing that is unspecified so far is how to pick $t$, the transaction being added to $inc$ or $exc$ in every iteration.

The choice matters; there appear to be a number of "good" choices which combined with the jump ahead optimization above result in an ~$\mathcal{O}(1.6^n)$ algorithm (purely empirical number, no proof), while others yield $\mathcal{O}(2^n)$.

These all appear to be good choices, with no meaningful differences for the worst case between them:
* Use as $t$ the highest-individual-feerate undecided transaction.
* Use as $t$ the transaction for which splitting on its minimizes the search space the most (first maximize 
$\operatorname{min}(|exc|+|inc \cup \operatorname{anc}(t)|,|exc \cup \operatorname{desc}(t)|+|inc|)$, then maximize the $\operatorname{max}$ of those values). This can be done over:
  * All undecided transactions.
  * All undecided transactions in $pot$.
  * All undecided transactions that are ancestors or descendants of the highest-individual-feerate undecided transaction.

In particular the last two options appear to be good choices, with no clear winner among them. Specific clusters exist for which either of the algorithms performs significantly better than the other, but this works both ways.

### 2.5 Choosing which work item to process

Lots of heuristics for the choice of $(inc,exc) \in W$ are possible which can greatly affect the runtime in specific cases, but the worst case is unaffected by this choice.

Thus it's reasonable to stick to a simple choice: treating $W$ like a (LIFO) stack which work items get appended tot, and popped from. This effectively results in a depth first traversal of the search tree, with a stack size that cannot exceed the total number of transactions in the cluster. This is probably the best choice from a memory usage (and locality) perspective.

If introducing randomness is desired (which may be the case if the algorithm is only given a bounded runtime), it's possible to instead treat $W$ like a small (say, $k=4$) fixed-size array of $k$ LIFO stacks, and picking from $W$ and/or additions to it are appending to/popping from a random one. This retains DFS-ish behavior with (in almost all cases) only a small constant factor larger memory usage.

### 2.6 Caching feerates and the potential set

To avoid recomputing the feerates of the involved sets ($inc$, $pot$, and $best$, specifically), the fees and sizes can be precomputed and passed along wherever the sets themselves are passed (including inside the work items). When sets are updated, e.g. in $inc = inc \cup \operatorname{anc}(t)$, only the fees and sizes of $(\operatorname{anc}(t) \setminus inc)$ need to be looked up and added to the cached value.

By extending our definition of work item to $(inc, exc, pot)$, carrying the potential set $pot$ along between its computation and the item it is for, more duplicate work can be avoided. A $pot_{new}$ entry is added to the $work_{add}$ and $work_{del}$ variables, containing conservative subsets for $pot_{new}$ in these branches.

Finally, this also lets us move the check that $pot$ has higher feerate than $best$ to the beginning of the processing loop, which can catch cases where $best$ improved between adding a work item and it being processed. The check inside the addition loop can be weakened to $pot \neq inc$, which is sufficient to make sure undecided transactions remain, and faster than a feerate comparison.

* Let $W = \{(\emptyset,\emptyset, \emptyset)\}$.
* Set $best = \emptyset$.
* While $W$ is non-empty and computation limit is not reached:
  * Take some work item $(inc, exc, pot)$ out of $W$.
  * If $\operatorname{feerate}(pot) > \operatorname{feerate}(best)$:
    * Find a transaction $t$ not in $inc$ or $exc$ to split on; this must exist.
    * Let $work_{add} = (inc \cup \operatorname{anc}(t), exc, pot \cup \operatorname{anc}(t))$.
    * Let $work_{del} = (inc, exc \cup \operatorname{desc}(t), pot \setminus \operatorname{desc}(t))$.
    * For each $(inc_{new}, exc_{new}, pot_{new}) \in \{work_{add}, work_{del}\}$:
      * For each $u$ not in $pot_{new}$ or $exc_{new}$, in decreasing individual feerate order:
        * If $\operatorname{feerate}(u) > \operatorname{feerate}(pot_{new})$: set $pot_{new} = pot_{new} \cup \{u\}$.
        * Otherwise, stop iterating.
      * For every transaction $p \in (pot_{new} \setminus inc_{new})$:
        * If $\operatorname{anc}(p) \subset pot_{new}$: set $inc_{new} = inc_{new} \cup \operatorname{anc}(p)$.
      * If $\operatorname{feerate}(inc_{new}) > \operatorname{feerate}(best)$ or $best = \emptyset$: set $best = inc_{new}$.
      * If $pot_{new} \neq inc_{new}$: add $(inc_{new}, exc_{new}, pot_{new})$ to $W$.
* Return $best$

### 2.7 Seeding with best ancestor sets

Under no circumstances do we want to end up with a $best$ whose feerate is worse than the highest-feerate ancestor set, as that'd mean we're worse off than just ancestor set based linearization. It is possible to run ancestor set based linearization and a bounded version of the search algorithm developed so far and then [merge](https://delvingbitcoin.org/t/merging-incomparable-linearizations/209) the two, but it is less work to instead pre-seed the search algorithm with the best ancestor set.

All we need to do is pre-split the initial $W$ item:
* Find $a$, the transaction whose ancestor set feerate is highest.
* Let $W = \{(\operatorname{anc}(a), \emptyset, \operatorname{anc}(a)), (\emptyset, \operatorname{desc}(a), \emptyset)\}$.
* Let $best = \operatorname{anc}(a)$.
* While $W$ is non-empty ...

Alternatively, the entire body of the "For each $(inc_{new}, exc_{new}, pot_{new})$" loop can be abstracted out to a helper that potentially updates $best$ and potentially adds an element to $W$, and this helper can then be invoked for the initial populating of $W$ too. This makes the jump ahead optimization available to even that first step.

## 3. Current implementation

My current [implementation](https://github.com/sipa/bitcoin/blob/wip_memepool_fuzz/src/cluster_linearize.h) incorporates most of the ideas listed above, with a few deviations:
* The connected-component splitting isn't implemented for linearization in general, but done inside the find-high-feerate-subset algorithm. It's combined with the ancestor pre-seeding there ($W$ is initialized with 2 elements per connected component: each excludes every other component, but one includes the component's best ancestor set, and one excludes the best ancestor set's transaction's descendants). This is somewhat suboptimal, as the find-high-feerate-subset algorithm only finds one subset, so in case there are multiple components, work for the later ones is duplicated. However, it was a lot simpler to implement than having the linearize code perform merging of multiple sublinearizations, and still beneficial.
* Newly added work items are added to one of $k=4$ LIFO stacks, in a round-robin fashion. A uniformly random stack is chosen to pop work items to process from.
* The transaction to split on is, among the set of ancestors and descendants of the highest-individual-feerate undecided transaction, the one that minimizes the search space the most.

-------------------------

