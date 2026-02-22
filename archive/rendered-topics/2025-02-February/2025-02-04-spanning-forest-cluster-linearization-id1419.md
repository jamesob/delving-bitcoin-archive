# Spanning-forest cluster linearization

sipa | 2025-02-07 20:30:49 UTC | #1

# Spanning-forest cluster linearization

<div data-theme-toc="true"> </div>

This is a write-up about a work in progress cluster linearization algorithm that appeared (and may still be) promising as a replacement for the ones described in the currently-implemented algorithms (Bitcoin Core PRs [30126](https://github.com/bitcoin/bitcoin/pull/30286), [30285](https://github.com/bitcoin/bitcoin/pull/30285), [30286](https://github.com/bitcoin/bitcoin/pull/30286) and Delving posts [on linearization](https://delvingbitcoin.org/t/how-to-linearize-your-cluster/303) and [related algorithms](https://delvingbitcoin.org/t/introduction-to-cluster-linearization/1032)).

While this spanning-forest algorithm looks promising, the recent [discovery](https://delvingbitcoin.org/t/how-to-linearize-your-cluster/303/9) by @stefanwouldgo that well-known [minimal-cut](https://en.wikipedia.org/wiki/Minimum_cut)-based algorithms with good asymptotic complexity exist for the problem we call cluster linearization (the established term is the "maximum-ratio [closure problem](https://en.wikipedia.org/wiki/Closure_problem)") is even more promising.

I am still creating this write-up, because despite lacking known complexity bounds, it appears to be very elegant, fast, and practical (I have a [prototype implementation](https://github.com/sipa/bitcoin/commits/spanning_tree_linearization). It is possible that good bounds for it still get found, or that its insights help others build on top, or that it ultimately appears more practical for the real-life problems we face than more advanced algorithms. Or perhaps we discover it's actually equivalent to a well-known algorithm. I don't know, but before switching over to studying another approach, I want to have it written down in some form, also to have my own thoughts ordered.

## 1. Background: cluster linearization as an LP problem

(Skip this section if you're not interested in the background that led to this idea)

Cluster linearization can be done by repeatedly finding the highest-feerate topologically-valid subset, moving it to the front of a linearization, and then continuing with what remains of a cluster. This reduces the problem of optimal linearization to the problem of finding the highest-feerate topologically-valid subset. This problem can be formulated as a linear programming problem (credit to Dongning Guo and Aviv Zohar):

* For every transaction $i$, have a variable $t_i \in \mathbb{R}_{\geq 0}$. $t_i > 0$ means including transaction $i$ in the solution set; $t_i = 0$ means not including it. Using **real numbers** instead of booleans here may seem strange, but it will turn out that all non-zero $t_i$ will have the same value anyway.
* Enforce the topology constraints by adding an **inequality per dependency**: for each parent-child pair $(p,c)$,  require $t_p \geq t_c$. Thus, if a child is included (nonzero), so is the parent.
* Use as **goal function** $g = \sum_i t_i \operatorname{fee}(i) / \sum_i t_i \operatorname{size}(i)$, which is the feerate of the solution set. It doesn't matter that the non-zero $t_i$ may take on values different from $1$, because they contribute equally to numerator and denominator anyway, meaning that multiplying all of them with the same value has no effect.
* To make the goal function linear (it is a fraction so far), add one additional **normalization constraint** that the numerator is $1$: $\sum_i t_i \operatorname{size}(i) = 1$, and thus the goal becomes just $g = \sum_i t_i \operatorname{fee}(i)$.

It can be proven that any optimal solution to this problem is an optimal solution to the "find highest-feerate topologically-valid subset" problem.

This immediately has two implications:
* Cluster linearization can be done in **worst-case polynomial time**, through polynomial-time LP solving algorithms, including ones based on [Interior Point Methods](https://en.wikipedia.org/wiki/Interior-point_method). These are not necessarily the best approach for our problem, but their existence conclusively settles the fact that the problem is not NP-hard for example.
* The entire scientific domain of linear programming solution techniques becomes applicable to our problem.

### 1.1. The simplex algorithm for finding high-feerate sets.

Probably the best-known, and in some variation, most commonly used algorithm for solving linear programming problems is the [**simplex** algorithm](https://en.wikipedia.org/wiki/Simplex_algorithm).

Internally, the simplex algorithm requires that the problem, and the goal, be stated as equations over a number of non-negative real variables, with no inequalities besides non-negativity. Our $t_i$ variables are already non-negative, but we have inequalities left. We can introduce slack variables to get rid of those:
* Introduce a $d_j \in \mathbb{R}_{\geq 0}$ for every dependency $(p_j, c_j)$: $d_j = t_{p_j} - t_{c_j}$, replacing the dependency inequalities.

In a problem with $n$ transactions and $m$ dependencies, this means $n + m + 1$ variables ($t_{1 \ldots n}$, $d_{1 \ldots m}$, and $g$), and $m + 2$ equations:
* $d_j = t_{p_j} - t_{c_j}$, for $j = 1 \ldots m$ (the dependencies)
* $g = \sum_i t_i \operatorname{fee}(i)$ (the goal)
* $\sum_i t_i \operatorname{size}(i) = 1$ (the normalization)

The simplex algorithm then maintains a set of $n-1$ ***free* variables**, which are chosen to be zero. All other variables (the ***basic* variables**) can be computed from the free variables using the $m+2$ equations. In every iteration of the algorithm, one free variable is chosen to become basic, while a basic algorithm is chosen to become free. In this context:
* A free $t_i$ variable means transaction $i$ is excluded from the solution.
* A free $d_j$ variable means that transactions $p_j$ and $c_j$ are either both included, or both excluded.

There are rules that control which variables are picked to become free/basic, which generally boil down to not worsening the existing solution, and keeping the solution valid. In addition, various policies exist that attempt to maximize progress in every step.

### 1.2. Interpreting simplex steps in our problem space

Note that groups of transactions can exist that are reachable from one another through free $d_j$ variables, and thus they must all be included in the solution set, or all excluded from it. Let's call these groups "chunks", for reasons that will become clear later.

Because of further rules on the choices taken by the simplex algorithm, further properties hold:
* No "cycle" of free $d_j$ variables can exist within a chunk (because at least one $d_j = 0$ would be redundant).
* Within every chunk, there can at most be 1 free $t_i$ variable (a second one would be redundant).
* By a counting argument (exactly $n-1$ free variables) it follows that all chunks, except for one, will contain a free $t_i$ variable, and thus be excluded.

Overall this means that every state of the simplex algorithm corresponds to a **partitioning of the transaction graph into chunks**, each of which is connected internally by a **spanning tree** (because no cycles) of free $d_j$ variables, and among them **one is the solution set**. In every iteration of the algorithm:
* One free variable is made basic, which is either:
  * A $t_i$ variable: an excluded chunk whose feerate is higher than the included chunk's feerate is made not-excluded. It may become included, or end up being merged with another (included or excluded) chunk, see below.
  * A $d_j$ variable: a chunk breaks apart into two chunks, as the dependency between them is made basic (in a spanning tree, removing any link breaks the tree in two), where the "top" chunk (the one containing the parent of the $d_j$ variable) has higher feerate than the included chunk, or the "bottom" chunk has lower feerate than the included chunk.
* One basic variable is made free, which is either:
  * A $t_i$ variable: the previously-included chunk is made excluded, if the chunk made non-excluded or split off above has no (basic) dependencies on another chunk (i.e., it is topological).
  * A $d_j$ variable otherwise: the non-excluded or newly-split off chunk merges with another chunk it depends on.

### 1.3. From high-feerate subsets to full linearization

There appear to be several superfluous details to the algorithm above. It keeps track of (up to) one specific free $t_i$ variable within each chunk, but the choice of *which* transaction $i$ that is for each excluded chunk is almost entirely irrelevant to the algorithm. Furthermore, one can wonder what the point is of having to keep track of which chunk is included explicitly, as it will always just be the highest-feerate one among the chunks. We choose to make a few simplifications to the algorithm:
* Get rid of the $t_i$ variables. The included chunk is implicitly the one with the highest-feerate; the rest is excluded.
* Avoid comparing with the chunk feerate of the included set:
  * To determine which $d_j$ to make basic, do not compare the top or bottom would-be chunk with the included chunk, but with each other. The rule becomes that one can split a chunk in two along a dependency, if doing so results in a top chunk with higher feerate than the bottom one.
  * A $d_j$ variable can be made free if the chunk the child is in has a higher feerate than the top chunk (i.e., merging a chunk with a parent if it is not topological).
  * By doing this, we're effectively not just optimizing the included chunk, but **all chunks**. And this is why they're called chunks: they become the chunks of the full linearization when the algorithm ends, with credit to @murch for noting the similarity between the algorithm and the [chunking algorithm](https://delvingbitcoin.org/t/introduction-to-cluster-linearization/1032#h-22-feerate-diagrams-and-chunking-5).

Thus, overall, we obtain an algorithm whose state is a [spanning forest](https://en.wikipedia.org/wiki/Spanning_tree#Spanning_forests) (a collection of spanning trees covering the whole graph) for the overall cluster, and which aims to find the overall linearization of the whole thing, not just the first chunk.

## 2. The algorithm

Restated from scratch, without the whole LP/Simplex background that led to it, the spanning forest linearization algorithm is:
* Input:
  * A transaction graph for a cluster, with fees, sizes, and dependencies between the transactions.
* Output:
  * An optimal linearization for the graph.
* State:
  * For every dependency in the graph, maintain a boolean, "**active**". Initially all dependencies are inactive (active dependency corresponds to "free $d_j$" in the previous section, but "active" seems like a more appropriate name when dropping the simplex context).
  * The set of active dependencies implicitly partition the graph into **chunks**. Two transactions are in the same chunk if there is a path of active dependencies (each in either the parent-of or child-of direction) from one to another. If there is no such path, the transactions are in different chunks. This means that the initial state is every transaction in its own singleton chunk.
  * An invariant of the algorithm is that no cycles can exist within the active dependencies (again, ignoring direction). In other words, the active dependencies within each chunk form a spanning tree, and the active dependencies for the entire cluster form a **spanning forest** for it.
* Algorithm:
  * Perform one of these permitted steps as long as any apply:
    1. **Merging chunks**. If there is an inactive dependency, where the parent and child are in distinct chunks, and the child chunk has higher feerate than the parent chunk, make the dependency active, merging the two chunks.
    2. **Splitting chunks**. If there is an active dependency, consider the two chunks that would be created by making it inactive (due to the spanning-forest property, this is the case for every active dependency). If the would-be chunk with the parent of the dependency in has a higher feerate than the child chunk, make the dependency inactive, splitting the chunk in two.
  * Output the final chunks, sorted from high feerate to low feerate, tie-breaking by topology, each chunk internally sorted by an arbitrary for valid topological order (e.g., by increasing number of ancestors).

### 2.1. Correctness

Since several changes were made to the simplex algorithm along the way to obtain the spanning forest algorithm, it is not necessarily the case anymore that it is correct.

The algorithm above, **if it terminates**, outputs an optimal linearization, or put otherwise: the successively output chunks are each a highest-feerate topological subset of what remains of the cluster. The [theory thread](https://delvingbitcoin.org/t/cluster-mempool-definitions-theory/202) proves that this suffices for an optimal linearization. To see the output chunks are optimal:
* There cannot be any higher-feerate chunk that depends on a lower-feerate chunk, because if there was, the chunk merging rule above would still apply, and the algorithm has not terminated. Thus, the chunks that come out, from high feerate to low feeate, form a **valid linearization**.
* As for optimality, it holds that the highest-feerate chunk is actually the highest-feerate topologically-valid subset of the **active subgraph** of the cluster. Since fewer dependencies means fewer constraints, the highest-feerate set for just the active subgraph is certainly no worse than the highest-feerate set for the actual, more constrained, problem. To see why the highest-feerate chunk is the highest-feerate topologically-valid subset of the active subgraph:
  * It is certainly topologically-valid, because it is topologically-valid for the actual problem (see above).
  * Imagine a higher-feerate topologically-valid subset of the active subgraph existed. Consider its intersections with the final chunks in the algorithm. At least one of those must have a feerate above the highest-feerate chunk itself (if not, their combined feerate, which is a weighted average, would not be higher either). That intersection must be obtainable by cutting off certain branches of the spanning tree, and some of those branches must have a feerate lower than the chunk feerate (otherwise cutting them off would not be an improvement). However, because the splitting rule of the algorithm is not applicable (or the algorithm wouldn't have terminated), no such branch can exist (if it did, the chunk would have been split in two right there).
* All the properties above remain valid after the highest-feerate chunk is removed from the graph, which is exactly the state obtained before the next chunk is output.

Note that all of these properties are conditional on the assumption that the algorithm terminates. It is less clear under what conditions that is the case (see below).

### 2.2. Refining the merge and split choices

So far, we have left unspecified *which* dependencies are to be made active/inactive. From casual fuzzing, it appears that just making random choices is sufficient to obtain the optimal result, but random choices **can result in repeated states**, which means it can include pointless work which we'd like to avoid.

Thanks to @ajtowns and @ClaraShk for the discussions that led to the conclusions in this section.

#### 2.2.1. Prioritize merge over split

One easy refinement, which will simplify further analysis, is to prioritize merging over splitting. The idea is that merging is making the state *topological*, while splitting is about making the state *better*. If we, after every improving split step, perform merges as long as possible, we make the state topological as soon as possible again. This has the practical advantage that if we want to stop the algorithm (due to running out of time), we end up with something that's at least valid.

#### 2.2.2. Merge by highest feerate difference

When performing merges, there may be multiple inactive dependencies where the child chunk has a higher feerate than the parent, in which case it is unclear which one to pick. From casual fuzzing, it appears that prioritizing the one where the feerate *difference* between the child and parent is maximal has a number of advantages:

* The same state is never repeated, which - if true - must imply that some (possibly minute) strict improvement happens every iteration.
* It appears to be a good choice for minimizing the number of iterations needed in the worst case.

Furthermore, it simplifies reasoning about termination. Call an "**improvement step**" to be one application of the splitting rule (using whatever criterion to select which one), followed by as many applications of the merging rule (using maximum feerate difference first as selection strategy) as possible. If the state was topological (= no merge steps applicable) before an improvement step, we can compare the feerate diagrams of the linearizations that would be output before and after the improvement step.

Think of the state of the chunks implicitly as an ordered list, from higher-feerate to lower-feerate, plus the fact that chunks can only depend on chunks that come before it. When a split happens, there are two possibilities:
* The higher-feerate split-off chunk (the parent chunk) depends (through another inactive dependency) on the lower-feerate one (the child chunk). Since the produced child has lower feerate than any other chunk it may depend on, the maximum-feerate-difference rule in this case says that it must **merge with itself again**, even if other merges are possible. In this case, the feerate diagram remains unchanged, because we end up with the same chunks as before the improvement step, though with a different spanning tree in the re-merged chunk. Whether this can end up in an infinite loop, and if not, how many iterations improving the same chunk this way can happen is an open question.
* If such a self-merge does not happen (because the split-off parent chunk does not depend on the split-off child), imagine that the two new chunks initially take on adjacent positions in the sorted chunk list (which is now no longer properly sorted), where the split chunk used to, with the parent one first. We can look at what happens to the parent and child chunk separately:
  * The higher-feerate split-off chunk can in this case only depends on chunks that used to precede it, but it may have a higher feerate than those. The maximum-feerate-difference rule now effectively boils down to this new chunk "**bubbling up**" (think: bubble sort) until it either ends up in a position where its feerate is no longer higher than what precedes it, or it finds a lower-feerate chunk it depends on, merging with it, and then continuing the process bubbling this merged chunk further up possibly. This is very similar to the behavior inside the [post-linearization algorithm](https://delvingbitcoin.org/t/linearization-post-processing-o-n-2-fancy-chunking/201) described earlier.
  * The same process, but reversed, happens to the split-off child. Its feerate dropped compared to the earlier chunk, so now chunks that used to succeed it may now have higher feerate than it. So the split-off child chunk starts **bubbling down**, until it finds a position in the sort chunk list where it has higher feerate than what follows, or when it encounters a higher-feerate chunk that depends on it, where it merges, and possibly continuing further.

Whenever a self-merge does not happen, every improvement step strictly improves the feerate diagram, by moving a higher-feerate part up, and a lower-feerate part down, each potentially merging with other chunks. So the question of termination is just about improvement steps that result in a self-merge.

#### 2.2.3. Split by maximizing *(fee<sub>parent</sub> size<sub>child</sub> - fee<sub>child</sub> size<sub>parent</sub>)*

It remains to be discussed how to decide what split to perform within an improvement step when there are multiple possible options. If it is the case that every improvement step is guaranteed to make some kind of progress, it may be acceptable to just **make improvement steps randomly**, but perhaps we can do better.

Going back to the LP formulation of the problem, it is a requirement that the derivative of the goal variable $g$ w.r.t. the variable being made free (in the case of a split, the $d_j$ variable) is positive (this translated to needing a would-be parent chunk to having higher feerate than the child). A natural choice is picking the one with the **highest derivative** in the simplex phrasing. Working that out corresponds to maximizing the function

$$
\begin{equation}
\begin{split}
q(A, B) & = \,\, & \left(\operatorname{feerate}(A) - \operatorname{feerate}(B)\right) \operatorname{size}(A) \operatorname{size}(B) \\ 
& = & \left(\frac{\operatorname{fee}(A)}{\operatorname{size}(A)} - \frac{\operatorname{fee}(B)}{\operatorname{size}(B)}\right)\operatorname{size}(A)\operatorname{size}(B) \\
& = & \operatorname{fee}(A)\operatorname{size}(B) - \operatorname{fee}(B)\operatorname{size}(A)
\end{split}
\end{equation}
$$

for A the would-be parent chunk and B the would-be child chunk. And from casual fuzzing, this indeed appears to be a good choice, with apparently relatively low worst-case numbers of iterations. Furthermore, it appears to result in **never repeating the same *split*** (in the sense that the same chunk is never split in the same parent and child chunk, ignoring the exact spanning tree it has. Other rules (e.g. maximizing feerate difference directly when splitting) don't seem to have this property: while they don't repeat the exact same state, they do appear to result in sometimes repeating the same splits a finite number of times.

Another insight is that *if* a split step does not result in a self-merge (see above), $q(A,B)/2$ is a lower bound on **increase in surface area** (integral) under the feerate diagram curve, which is directly related to our overall goal (because the optimal linearization necessarily has the highest-possible surface area under the curve).

The $q()$ function has other interesting properties too, which may or may not matter:
* $q(A,B) = \sum_{i \in A} \sum_{j \in B} q(i,j)$ (i.e., it is a [bilinear map](https://en.wikipedia.org/wiki/Bilinear_map)).
* $q(A,B) = q(A, A \cup B) = q(A \cup B, B)$

### 2.3. Initial state

We started with stating that the initial state is all dependencies inactive (i.e., all transactions in their own chunk). But that state is typically not topologically valid, which is a requirement for the analysis of the improvement steps above.

It is possible to just start with a general merging step to avoid this: even just merging everything randomly would suffice. But we can do better. In practice, we always **start with a known linearization** for a cluster, and the goal is improving it, rather than finding a new linearization from scratch.

With that, we can use the following approach:
* Iterate over all transactions in the input linearization, from front to back. And for each:
  * Perform a bubbling-up merge on the chunk that transaction is in (initially a singleton).

The result is actually **exactly the [post-linearization algorithm](https://delvingbitcoin.org/t/linearization-post-processing-o-n-2-fancy-chunking/201)**, but in the spanning-forest setting. This means that the chunks that come out, even without any further improvement steps, will form a linearization at least as good as the input linearization. Any work done on top just makes it even better.

In other words, we can see this as natural linearization-improvement algorithm, avoiding the need for [LIMO](https://delvingbitcoin.org/t/limo-combining-the-best-parts-of-linearization-search-and-merging/825) or explicit [linearization merging](https://delvingbitcoin.org/t/merging-incomparable-linearizations/209). A downside is that it is unclear how to incorporate a "make sure every next chunk is at least as good as the highest-feerate ancestor set among what remains", without explicitly computing such a linearization and merging with it. It is unclear how big of a problem that is.

### 2.4. Resulting combined algorithm

Putting all the pieces above together, we get:

* SpanningTreeLinearize(C, L)
  * Helper function: MergeUpwards(T)
    * Loop:
      * Find the chunk H which transaction T is in.
      * Among all other chunks H', which H has an (inactive) dependency on, find the lowest-feerate one.
      * If no H' is found, stop loop.
      * Activate a dependency between H and H'.
  * Helper function: MergeDownwards(T):
    * Loop:
      * Find the chunk H which transaction T is in.
      * Among all other chunks H' which have an (inactive) dependency on H, find the highest-feerate one.
      * If no H' is found, stop loop.
      * Activate a dependency between H and H'.
  * Helper function: Improve(D)
    * Deactivate D, resulting in chunks Hp and Hc.
    * If Hc has a dependency on Hp, activate it and stop.
    * Otherwise:
      * MergeUpwards(Hp)
      * MergeDownwards(Hc)
  * Initialize state with all dependencies of C as inactive.
  * For each transaction T in L:
    * MergeUpwards(T)
  * Loop:
    * Among all active dependencies, find D, the one with the highest *q = fee(Hp)size(Hc) - fee(Hc)size(Hp)*, where *Hp* is the would-be parent chunk if it were deactivated, and *Hc* the would-be child chunk.
    * If *q <= 0*, or no active dependency exists, stop.
    * Improve(D)
  * For all chunks H, in decreasing chunk feerate order, tie-breaking by topology:
    * Output H in arbitrary topologically-valid order.
## 3. Speculation about complexity

In my prototype implementation, I have used a number of techniques to make this efficient.

The main one is to maintain at all times the precomputed **would-be parent chunk fees and sizes** for each active dependency. This is fairly expensive, but for each update (merge or split) can be computed at once for the entire chunk at roughly the same cost as would to compute it for just one dependency individually. To update after a merge:
* Let T and B be the top and bottom chunks being merged.
* Travel from the activated dependency outward, along active dependencies, and:
  * For each dependency D traversed downwards (reached the parent transaction before the child):
    * If D is inside T, add B's fee/size to D's would-be-parent fee/size.
    * If D is inside B, add T's fee/size to D's would-be-parent fee/size.

And similar for splits, except subtracting instead of adding.

This has cost $O(d)$ per update, where $d$ is the number of active and inactive dependencies that are inside or on the border of the chunk. As this may include all dependencies in the graph, $d$ may be up to $O(m)$, but by keeping track within each transaction which dependencies are active and inactive, it may be possible to bring it to $O(n)$ (as at most $n-1$ dependencies can be active at a given point in time).

Every split step requires looking over all active dependencies, comparing their would-be parent chunk feerate with the would-be child chunk feerate, to determine which to split, which is $O(n)$, as there are at most $n-1$ active dependencies. Let $s$ be the number of improvement steps performed. The number of splits is of course $s$: one per improvement. The number of merges may be up to $s+n-1$, as we can start with up to $n$ chunks, end with at least $1$, and each merge reduces the number of chunks by one, while each split increases the number by one. Lastly, each merge is $O(m)$ to figure out what to merge with, but it may be possible to bring this down to $O(n)$ by observing that no two dependencies from the currently-being-bubbled chunk to the same chunk are worth considering. So overall, I believe this leads to a cost of $O(m(s+n))$, and it may be possible to reduce it to $O(n(s+n)) = O(ns + n^2)$.
 
The big question is of course what $s$ is. Basic on fuzzing results, it seems to mostly depend on $m$, and be somewhat super-linear in it, perhaps $m^{3/2}$ or $m^2$. Given the fact that $m$ may be up to $n^2/4$ itself, this leads to an overall complexity of maybe (and again, this is just based on very basic fuzzing and extrapolating the worst case numbers seen) of somewhere between $O(n^4)$ and $O(n^6)$. It is also entirely possible that the worst case is exponential and I just haven't been able to discover the cases that exhibit this behavior, or it may even be the case that non-terminating examples exist.

### 3.1. Goals

Note however that while the best possible complexity is certainly an interesting research question, what matters practically is rather:
* What is the largest cluster size for which we can guarantee a "good enough" linearization in a very short timeframe (think ~50 µs). So far, we have defined "good enough" as "each chunk is at least as good as the best remaining ancestor set", as that is both efficient to compute ($O(n^2)$), and roughly matches the existing CPFP-aware mining algorithm in Bitcoin Core. Since this is a whole-linearization algorithm, and not an individual find-good-subset algorithm, it may be hard to incorporate actual ancestor set finding in it, but maybe an argument can be made that $k$ iterations of spanning-forest improvement steps is qualitatively as good as ancestor set, for some function $k$.
* Beyond the minimal "good enough" above, I think the real question is which algorithm has the best worst-case "improvement per unit of time", for cluster sizes within the bound established above.

-------------------------

sipa | 2025-02-07 20:20:58 UTC | #2

I cleaned up my implementation a bit, and pushed it [here](https://github.com/sipa/bitcoin/commits/spanning_tree_linearization), replacing the `Linearize()` function currently merged in Bitcoin Core.

Here are benchmarks with the existing examples in the codebase. These are the timing for full, optimal, linearizations from scratch, for a rather arbitrarily-selected collections of "hard" clusters actually seen on the network. "hard", of course, is based on examples that were hard with the existing approach, by a few different metrics, but this may not carry over to the new approach. I entirely expect that these are not anywhere close to the worst cases for the spanning-forest algorithm.

|               µs (master) | µs (this)  | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | #tx | benchmark
|--------------------:|----------------------------:|--:|-|:-------
|           70.6 |  21.1 || 71 | `LinearizeOptimallyExample00`
|           94.9 |  30.4 || 81 | `LinearizeOptimallyExample01`
|           55.3 |  29.5 || 90 | `LinearizeOptimallyExample02`
|           54.7 | 31.1 ||  87 | `LinearizeOptimallyExample03`
|            2.3 | 4.1 || 35 | `LinearizeOptimallyExample04`
|           12.6 | 9.7 ||  60 | `LinearizeOptimallyExample05`
|           37.2 | 35.6 ||  99 | `LinearizeOptimallyExample06`
|            8.7 |  7.5 ||  52 | `LinearizeOptimallyExample07`
|           67.5 | 22.0 || 69 | `LinearizeOptimallyExample08`
|           15.2 | 24.9 || 77 | `LinearizeOptimallyExample09`
|            9.8 | 9.6 || 48 | `LinearizeOptimallyExample10`
|      371677.6 | 24.1 || 77 | `LinearizeOptimallyExample11`
|          491.4 | 7.2 || 40 | `LinearizeOptimallyExample12`
|           59.5 | 32.9 || 96 | `LinearizeOptimallyExample13`
|          361.3 | 34.6 || 93 | `LinearizeOptimallyExample14`
|       14003.9 | 9.7 || 55 | `LinearizeOptimallyExample15`
|       10864.3 | 43.4 || 76 | `LinearizeOptimallyExample16`
|       13992.6 | 50.6 || 98 | `LinearizeOptimallyExample17`
|       14029.4 | 45.4 || 99 | `LinearizeOptimallyExample18`
|       15198.4 | 32.6 || 78 | `LinearizeOptimallyExample19`

-------------------------

sipa | 2025-05-24 14:34:59 UTC | #3

I have [posted](https://delvingbitcoin.org/t/how-to-linearize-your-cluster/303/68) a comparison of this algorithm (SFL) with the GGT parametric min-cut algorithm, and the existing candidate set searching (CSS) algorithm from the other thread.

---

Here are some of the open problems still with SFL:
* **Termination**: there is a proof that if SFL terminates, the result is an optimal linearization. ~~However, there is no proof that it always will, even though fuzzing seems to indicate that the "merge chunks by maximum feerate difference" heuristic is sufficient to never cause the same state to be repeated (which would suffice for termination, as there is only a finite number of possible states).~~ EDIT: it is not guaranteed to terminate.
  * **Complexity bound**: it would be good to have some bound on how many iterations the algorithm can go through. If the above observation (no repeated states) holds, then there is some bound on the number of iterations, as there are at most $2^m$ states, but this is not a very interesting bound. Somewhat stronger bounds are possible by taking into account that no more than $n-1$ dependencies can be active, and that they form a spanning forest, but it remains exponential.
* ~~**Equal-feerate chunk splitting**: The algorithm works if splits are applied whenever the top subset has *strictly* higher feerate than the bottom, and merges whenever the bottom has *higher or equal* feerate as the top. If instead splits are done when the top has higher-or-equal feerate, the algorithm may loop forever. If instead merged are only done when the bottom has strictly higher feerate than the top, the result may not be topological. The result of this is that the chunks that come out may be contain multiple equal-feerate components that could be split in a topologically-valid manner, i.e., become separate equal-feerate chunks. It would be nice to find an algorithm that can efficiently find these, whether that's done as part of SFL, or as a separate post-processing step. Note that this is not simply finding connected components within the chunks: the SFL chunks are *always* connected already.~~ EDIT: solved, see [post](https://delvingbitcoin.org/t/spanning-forest-cluster-linearization/1419/9) below.
  * **Sub-chunk linearization**: Somewhat related, SFL provides no way to order the transactions within a chunk. This only matters for block building with [sub-chunk granularity](https://delvingbitcoin.org/t/cluster-mempool-block-building-with-sub-chunk-granularity/1044), but intuitively, it feels like there is some information within the SFL state that may be useful. When a chunk is repeatedly split and re-merges with itself, it feels to me like it is really improving some implied sub-chunk linearization, until that sub-chunk linearization becomes good enough that it permanently splits the chunk in two. An "optimal sub-chunk linearization" would imply splitting equal-feerate chunks too, though beyond that, it's not clear how to define optimality here.
* **LIMO-like improvements**: It would be useful if one could take an SFL state, and a topologically-valid subset, and make directed improvement steps such that the corresponding linearization is at least as good as moving that subset to the front of the linearization. This would permit things like mixing in ancestor sets into the state.
  * **Merging states**: a more general question is, can one, given two SFL states (sets of active dependencies) efficiently find a third SFL state whose linearization is at least as good as both linearizations corresponding to the input states? This is trivially possibly by extracting linearizations from both, applying the [linearization merging](https://delvingbitcoin.org/t/merging-incomparable-linearizations/209/1) algorithm, and converting those back to SFL, which is all possible in $\mathcal{O}(n^2)$, but this loses valuable information. A final SFL state lets one decide that the corresponding linearization is optimal (by not having any more splits or merges to apply) in $\mathcal{O}(n^2)$ time, while doing that for just a linearization is as far I know equivalent to computing a (single) min-cut, which is $\mathcal{O}(n^3)$ even with the most state-of-the-art algorithms.

-------------------------

sipa | 2025-04-23 11:11:19 UTC | #4

I [posted](https://delvingbitcoin.org/t/how-to-linearize-your-cluster/303/73) more extensive benchmarks of the spanning-forest linearization algorithm, as well as the old exponential candidate-set search algorithm, and the minimum-cut based parametric breakpoints algorithm from the GGT paper in the other thread.

-------------------------

sipa | 2025-05-12 12:12:10 UTC | #5

### Fairness

There are a number of options for finding which split operation to perform:
1. Among all active dependencies, pick the one with the highest-$q()$. This is what the post above describes.
2. Round-robin over all chunks, until one is found with at least one positive-$q()$ dependency. Pick the highest-$q()$ active dependency within that chunk.
3. Round-robin over all transactions, until one is found with at least one positive-$q()$ dependency among its children. Pick the highest-$q()$ dependency among them.

(3) has arguably the best "fairness", as it balances activity somewhat equally over all transactions, giving each in turn a chance to improve its feerate (by splitting off a worse child + subtree). That was the approach I used initially in the benchmark I linked to above, but I've since changed the approach to (2), as it seems (3) does not actually guarantee that every step is an improvement (@ajtowns found that the same state can be repeated), even though it seems extremely hard to hit this when randomness is added into the process. (2) still distributes work somewhat fairly, at least as soon as different potential parties' transactions have been split into separate chunks - and if they can't, then the chunk may well be optimal already and it doesn't matter. More complex cases may exist though, where the optimal chunks consists of combinations of multiple parties' transactions, in which case it's unclear to me what approach is better.

Further, if (1) has the property that it always makes progress, then so does (2). Assume it doesn't, and a state exists for which (2) cycles. We know that any split that is not immediately followed by a self-merge strictly improves the area under the diagram, so afterwards no repetition of earlier states is possible anymore; thus, the cycle can only consist of splits + self-merges. If one removes from this cycling state all chunks but one splittable one (which must exist, as otherwise the algorithm has terminated), one obtains a state that cycles for (1), which is in contradiction with the assumption.

~~Note that we don't know (yet) whether (1) or (2) actually always make progress, just that if (1) does, then so does (2).~~ EDIT: neither always makes progress.

### Randomization

Even though it appears that nearly all real clusters (based on simulations using replay of 2023 P2P activity) can be linearized optimally by SFL in a very reasonable time, it's also clear that it's not hard to construct clusters that take longer, and I believe that it's possible to construct pathological examples where it takes a very long time.

Because of that, I think the right approach is to having randomization in the linearization algorithm, so that no attacker can deterministically trigger problems across the network, like specific subsets of a cluster being badly linearized, causing things like say a simple CPFP not working. There are some downsides to non-determinism, like in extreme cases getting inconsistent relay behavior, but I think it's preferable to deterministic attacks, and given that we expect that pretty much all non-adversially-created clusters will be linearized optimally in negligible time with SFL, it feels like a non-issue.

Here is a list of the types of randomness I have currently implemented in my prototype implementation (which the benchmarks are for):
* All transactions are given a random index value at startup. This index value is used for:
  * When performing a split, and there are multiple active dependencies within a chunk with the same $q()$, one with the lowest-index parent transaction is used.
  * When performing a merge, and there are multiple inactive dependencies between the chunks being merged, one with the lowest-index child (when bubbling up) or lowest-index parent (when bubbling down) is used.
* The dependencies (both active and inactive, and both parent and children) per transaction are kept in lists that are ordered randomly at startup. When a dependency is activated, it moves to the end of the active list. When a dependency is deactivated, it is placed in a random position in the inactive list. These lists are used to:
  * When performing a split, and there are multiple active dependencies within a chunk with the same $q()$ and the same parent transaction, the first one from the list is used.
  * When performing a merge, and there are multiple inactive dependencies between the two chunks with the same child (when bubbling up) or same parent (when bubbling down), the first one from the list is used.
* There is a FIFO queue of chunks which should be considered for splitting. This queue is randomly shuffled initially.
* If no existing linearization is provided, a random one is constructed by shuffling the transactions randomly, and then sorting by number of ancestors.

-------------------------

sipa | 2025-05-11 22:39:54 UTC | #6

It seems that even when restricting splits to maximum-$q$, it is possible that SFL repeats the same state.

This 15-transaction cluster, with 30 dependencies, allows for a cycle of 24 splits + merges that returns to the same state:

![out|690x100](upload://2hL77yic98KiusXGyjX7aWAowF6.png)

The black edges form the initial spanning tree (T1T0, T6T0, T12T0, T2T1, T3T1, T7T4, T7T5, T12T5, T14T5, T10T7, T2T8, T2T9, T13T11, T2T13). From there, a sequence of 24 steps are possible that each deactivate one dependency, and active another one (possibly an initially grey one). Those steps are -T12T0+T13T4, -T7T4+T13T5, -T2T1+T3T8, -T2T8+T3T9, -T2T9+T6T5, -T6T0+T3T7, -T13T5+T14T11, -T14T5+T12T11, -T12T5+T1T4, -T13T4+T3T13, -T3T7+T10T8, -T3T8+T10T9, -T3T9+T6T11, -T6T5+T7T11, -T3T13+T14T0, -T14T11+T12T0, -T12T11+T7T4, -T1T4+T7T0, -T7T11+T2T9, -T10T9+T2T8, -T10T8+T6T0, -T6T11+T2T1, -T7T0+T14T5, -T14T0+T12T5.


This was found by building a graph whose nodes are the *states* of SFL (i.e., set of active dependencies in the cluster above), and whose edges represent steps SFL can take (splits + self-merges that can follow, if any), and then exploring random parts of this state diagram. This approach was suggested and first implemented by @ajtowns, and the example above was found thanks to @gmaxwell running it on 832 CPU cores worth of hardware.

---

Overall, this means that SFL really does not have a termination guarantee. That's unfortunate, because it means there is no amount of "guaranteed progress" it'll make over time. Practically however, these repeatable states seem hard to find, and even in clusters that admit them, they have some 50%-80% chance of being escaped from in every step, when randomization is involved. The only real concern would be the existence of a cluster which *inescapable* repeating states.

-------------------------

gmaxwell | 2025-05-12 21:42:32 UTC | #7

So far all looping states are extraordinarily likely to be immediately exited, and so probably have no effect on the runtime.   No one complains when they look at RNG code and see something like a while(res==0)res=random_number();   That's the kind of 'unbounded'  operation that is (apparently) rarely possible for SFL.   Though it's existence probably makes it inordinately hard to prove anything about its runtime.  (Sadly, also that there don't exist ones that are more like while(res)res=random_number();  :( )

Even if it were possible for an attacker to construct an inescapable loop, this code is always run with a time limit-- and it's a limit that is low enough that for large inputs totally ordinary non-looping inputs would hit it too.  And since SFL is generally so much faster than alternatives for non-trivial inputs, *and* easily randomized I think it's likely less vulnerable to being forced into an unfair conclusion in spite of the potential for loop states.

One should also keep in mind that given the complexity of these algorithms I wouldn't be shocked if real implementations of the proven worst case bound algorithms could also loop, either because the proof was just wrong or (more likely) implementation errors.  Consider, it's common for programmers to get bisection wrong and its a much simpler algorithm that any closure problem solver.. and it's much easier to detect that bisection gets the boundary wrong than to detect some failures in these algorithms.  It's likely people here are applying much more scrutiny, both because computing power has become very cheap and just the culture around Bitcoin Core.  (I wouldn't be surprised if it took more computing power to casually find one counter example to max-q than had been spent on all computation in the history of mankind  back in the mid 1960s when these algorithms were first being explored).  So don't be too hard on the rare failures here. :)

[Aside, I found several more loop states, which are pending validation by Sipa]

That said, I think it may be worth giving https://onlinelibrary.wiley.com/doi/10.1002/net.1012  a read as it discusses a 1960's algorithm which has been preferred by industry over GGT in spite of the latter's theoretical benefits (perhaps for the same reasons we've seen for SFL over GGT), and the paper goes on to make small modifications to give it O(n^3 log n) class worst case bound.  The algorithm is flowless and sounds (to my lay ears) at least somewhat morally similar to SFL... and so some of the techniques there may be of use even if the algorithm itself isn't.

A good worst case complexity bound is a nice thing to optimize for, but given that for larger problem even the best available will not run to termination, I don't know if its really relevant.  And as we've seen from sipa's GGT implementation (and from comments in the literature, suggesting that it's not just sipa) there are substantial constant factor differences that easily dominate at the sizes of interest here.

-------------------------

sipa | 2025-05-17 13:13:41 UTC | #8

I have made a few changes to the merge and split selection works:
* When deciding which dependency between two to-be-merged chunks to activate, pick a uniformly random one. It turns out this can be done in $\mathcal{O}(n)$ time anyway, without needing to maintain randomly-sorted lists of dependencies per transaction. The previous approach picked a randomized (though not uniformly random) dependency, but always from the first transaction in the cluster that had any. This change alone seems to dramatically improve worst-case performance in synthetic clusters that are very dense (a factor 1.5x in some settings), but it also reduces attackers' ability to reliably trigger bad decisions even further.
* Keep track, per chunk, how many consecutive split attempts have failed on it (because the top part that was split off contains a dependency on the bottom part). Whenever that number is 1 below a multiple of 3, a uniformly random dependency within the chunk (among those with positive $q$) rather than the one with highest $q$ is split. Benchmarks show that max-$q$ is generally better on all types of graphs I can construct, but it does raise a small concern that by being more deterministic, it might be the case that adverserially-constructed clusters could reliably cause it make bad choices. I have not found any way of doing that, especially when merging is already done randomly, but out of an abundance of caution, this introduces an even more random step occasionally. By only triggering every 3rd attempt, its performance impact is minimal on realistic clusters, for which splits generally just don't fail.

I have updated the benchmarks in the [other thread](https://delvingbitcoin.org/t/how-to-linearize-your-cluster/303/73) to reflect this.

With these changes, I'm quite confident that SFL is still the way to go.

-------------------------

sipa | 2025-05-24 19:37:28 UTC | #9

[quote="sipa, post:3, topic:1419"]
The result of this is that the chunks that come out may be contain multiple equal-feerate components that could be split in a topologically-valid manner, i.e., become separate equal-feerate chunks. It would be nice to find an algorithm that can efficiently find these, whether that’s done as part of SFL, or as a separate post-processing step.
[/quote]

Solved!

The idea is that equal-feerate chunks can be split using the existing algorithm, by perturbing the transaction feerates slightly. If we pick an arbitrary transaction in a chunk, and:
* give it an infinitesimally **higher** feerate than it really has, then if a way to split the chunk exists such that this transaction is in the **top** part, it will be found using the normal split/merge algorithm
* give a transaction an infinitesimally **smaller** feerate than it really has, then if a way to split the chunk exists such that this transaction is in the bottom part, it will be found using the normal split/merge algorithm.

If we try both the infinitesimally-higher and infinitesimally-smaller approaches for the same transaction, then if a split exists it must be found, as this transaction must appear in either the top or the bottom of the split.

We do not actually need to modify the transaction feerates to implement this. Instead, for increased feerate we can look for $q=0$ dependencies where the modified transaction is in the top part, as $q(a \cup \epsilon, b) = q(a,b) + q(\epsilon,b) = q(\epsilon,b) > 0$, where $\epsilon$ is an imaginary transaction with very high feerate, but infinitesimal size. Similarly, for decreased feerate we can look for $q=0$ dependencies with the modified transaction in the bottom. If the chunks are already optimal otherwise, the resulting split components cannot merge with any other chunk, so we only need to attempt self-merges, and do not need a full merge sequence.

More concretely, given a chunk:
* Pick an arbitrary transaction $t$ in it.
* Loop, trying to split the chunk with $t$ in the top set:
  * Find an active dependency $d_1$ in it with $q \geq 0$ (but note that $q > 0$ is not possible anymore if the normal optimization process has finished), and which has $t$ in its **top** set. If no such dependency is found, stop.
  * Deactivate $d_1$.
  * Find an inactive dependency $d_2$ whose child is in $d_1$'s top and whose parent is in $d_2$'s bottom. If none can be found, recurse into minimizing the created top and bottom chunk.
  * Activate $d_2$.
* Loop, trying to split the chunk with $t$ in the bottom set:
  * Find an active dependency $d_1$ in it with $q \geq 0$ (but note that $q > 0$ is not possible anymore if the normal optimization process has finished), and which has $t$ in its **bottom** set. If no such dependency is found, stop.
  * Deactivate $d_1$.
  * Find an inactive dependency $d_2$ whose child is in $d_1$'s top and whose parent is in $d_2$'s bottom. If none can be found, recurse into minimizing the created top and bottom chunk.
  * Activate $d_2$.

I have implemented this in the [Bitcoin Core PR](https://github.com/bitcoin/bitcoin/pull/32545) for adding SFL.

When running this on the replayed 2023 data, it appears that out of 24693846 clusters with 2 or more transactions, 136709 of them (0.55%) gain one or more chunks by running this chunk minimization procedure (at least for some random seeds), adding on average 1.58 chunks per affected cluster, or 0.0087 chunks per cluster over the whole dataset.

-------------------------

blockchainhao | 2026-01-16 09:28:23 UTC | #10

[quote="sipa, post:1, topic:1419"]
while a basic algorithm is chosen to become free
[/quote]

typo, 'algorithm --> ' 'variable'

-------------------------

blockchainhao | 2026-01-16 15:21:23 UTC | #11

[quote="sipa, post:1, topic:1419"]
The simplex algorithm then maintains a set of n-1 ***free* variables**, which are chosen to be zero. All other variables (the ***basic* variables**) can be computed from the free variables using the m+2 equations.
[/quote]

I think here $g$ as the optimization target shouldn't be seen as a variable and $g = \sum_i t_i \mathrm{fee}$ as the goal function shouldn't be seen as an equation. So it's better to say: the $m + 1$ basic variables can be computed from the free variables using the $m + 1$ equations

-------------------------

sipa | 2026-02-22 15:05:58 UTC | #12

Update: SFL ([PR 32545](https://github.com/bitcoin/bitcoin/pull/32545), [PR 34259](https://github.com/bitcoin/bitcoin/pull/34259), [PR 34023](https://github.com/bitcoin/bitcoin/pull/34023)) has been merged into Bitcoin Core's master branch, and is planned to be in the upcoming [31.0 release](https://github.com/bitcoin/bitcoin/issues/33607).

-------------------------

