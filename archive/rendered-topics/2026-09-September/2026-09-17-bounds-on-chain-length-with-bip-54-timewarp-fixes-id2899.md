# Bounds on chain length with BIP-54 timewarp fixes

sipa | 2026-09-17 15:20:49 UTC | #1

<div data-theme-toc="true"> </div>

### 1. Timewarp fix rules

[BIP-54](https://github.com/bitcoin/bips/blob/09e21036a4001fe6c9ba65c1d3a39b737768132f/bip-0054.md) proposes (among other things) two rules to curb undesirable tricks that can be played with block header timestamps:
1. The timestamp of the first block of a retarget period must not be more than 7200 seconds before the timestamp of the preceding block (to prevent the [ArtForz timewarp attack](https://bitcointalk.org/index.php?topic=43692.msg521772#msg521772)).
2. The timestamp of the last block of a retarget period must not be before the timestamp of the first block of the same period (to prevent the [Murch-Zawy alternating attack](https://delvingbitcoin.org/t/zawy-s-alternating-timestamp-attack/1062)).

It is worth asking whether these two rules are sufficient, or whether other attacks remain undiscovered. The short answer is that these rules indeed **suffice to bound chain growth**.

### 2. Results

Specifically, it can be shown that any chain in which no block timestamp exceeds the genesis timestamp by more than $t + 7200$ seconds (7200 seconds being how far in the future a block timestamp may be), with no more than $w$ chain work, has at most $u(t,w)$ blocks where
$$
\begin{split}
u(t,w) &= \frac{2016}{2016\cdot600-7200}\left(t + 2016\cdot600\cdot\ln\frac{2016+(3+\frac{3}{2^{15}})\cdot\frac{w}{2^{32}+2^{16}}}{2016+3+\frac{3}{2^{15}}}\right) + 1 \\
  &= \frac{7}{4175}\,t + \frac{338688}{167}\,\ln\frac{66060288+\dfrac{98307\,w}{4295032832}}{66158595} + 1 \\
  &\approx \frac{t}{596.4285} + 1405.753\log_2 w - 58189.3
\end{split}
$$
This formula is proven in Lean to hold for all $t \geq 0$ and $w \leq 2^{208}$ (far beyond the point where proof-of-work security breaks down), see Section 4. The decimal approximation additionally requires $w \geq 2^{57}$. The bound holds under all of Bitcoin's consensus rules, including the integer rounding in Bitcoin's chain work and difficulty adjustment computations.

If we limit $w \leq 2^{128}$ (the amount of work at which security assumptions break down anyway) we get $u(t,w) \leq t/596.4285 + 121746.97$. In other words, **the long-term block rate is bounded to about one block per 9m56s**, plus a bounded number of extra blocks that must be paid for with difficulty increases.

As of 2026-Sep-09 21:30 UTC, with $t = 557982895$ seconds after genesis, and a real chain at height 966,270 with total work $w = 2^{96.349377}$, this formula gives a limit of 1,012,794 blocks. That is a 3300x improvement over what would be possible without the timewarp fixes. Both rules are needed: with either one missing, the limit remains over 3.34 billion blocks. This shows that **both rules are necessary and together sufficient** to get a tight bound.

Note that the formula only provides a conservative upper bound, and not the actual maximum number of blocks. For the values above, the longest known chain is 1,007,326 blocks long, but it is not proven to be the longest.

### 3. Derivation

In what follows, we derive the formula above from Bitcoin's consensus rules, showing it is a true upper bound.

First, some constants so we can reason about everything symbolically:
* $R = 4$, the factor to which the retarget clamps the ideal difficulty adjustment ratio, in both directions
* $R_\text{max} = 4 + 3\cdot2^{-15}$, the maximum ratio by which the difficulty can actually increase per period (slightly more than $R$ due to rounding; see the note below)
* $N_\text{period} = 2016$, the number of blocks per period
* $T_\text{period} = 1209600 = 600\cdot N_\text{period}$, the intended time for one period (2 weeks)
* $T_\text{future} = 7200$, how far in the future a block timestamp can be
* $T_\text{grace} = 7200$, the grace period permitted by rule (1): how long before the end of the previous period the next one may start
* $W_\text{unit} = 2^{32} + 2^{16} = 4295032832$, a lower bound on the ratio of any block's chain work to its difficulty: slightly below the unrounded work-per-difficulty ratio $2^{48}/65535$, to allow for rounding effects (see the note below)

> **A note on rounding.** Two integer-rounding effects in Bitcoin's consensus rules matter here, and the constants above account for both.
>
> * **Chain work.** A block's chain work is computed as $c = \lfloor 2^{256}/(g+1)\rfloor$ for its target $g$, while its difficulty is $d = 65535\cdot2^{208}/g$. Writing $2^{256} = c\,(g+1) + r$ with $0 \leq r \leq g$ gives $cg = 2^{256} - c - r$. Every target is at most $65535\cdot2^{208}$, and $c \leq 2^{208}$ for every block of a chain with at most $w \leq 2^{208}$ total work, so $cg \geq 2^{256} - 65535\cdot2^{208} - 2^{208} = 2^{256} - 2^{224}$, i.e., $c \geq \frac{2^{256} - 2^{224}}{65535\cdot2^{208}}\,d = (2^{32} + 2^{16})\,d = W_\text{unit}\,d$: the per-block bound used below. The choice of $W_\text{unit}$ therefore accounts for all possible effects of the actual chain work calculation if we require $w \leq 2^{208}$.
> * **Difficulty adjustment.** The retarget computes the next target as $\lfloor g\cdot\text{clamp}(s)/T_\text{period}\rfloor$, with the timespan ratio clamped to $[1/R,\,R]$, and then rounds it down to the compact form $v\cdot256^{e}$ with $2^{15} \leq v < 2^{23}$. Rounding a target down raises the difficulty, so ignoring the rounding at worst underestimates difficulty (which is fine for our length overestimate, see Section 3.3). The rounding does however raise the maximum difficulty adjustment factor $R_\text{max}$ above $4$. If the target $g$ has mantissa $4i+3$ for $i \geq 2^{15}$, and the period span $s$ is (clamped to) its minimum value (3.5 days), the adjustment will divide the target's mantissa by $R = 4$, yielding $i$, without changing the exponent. That ratio $\frac{4i+3}{i}$ is at most $4 + 3\cdot2^{-15}$. This is the worst case overall: mantissas below $2^{17}$ divide exactly (the exponent decreasese), and for any longer span the value being rounded is at least $g/4$, so by monotonicity of the rounding the resulting target is at least the one obtained here. Hence $R_\text{max} = 4 + 3\cdot2^{-15}$.

We model the chain with the following variables:
* The chain has $n$ blocks, consisting of $k+1$ difficulty adjustment periods: $k$ complete ones with $N_\text{period}$ blocks each, and a possibly-incomplete tail period with $m$ blocks ($m \in [1,\ N_\text{period}]$). Thus, $k = \lfloor (n-1)/N_\text{period}\rfloor$, and $n = k\cdot N_\text{period} + m$.
* Each block in period $j$ has difficulty $d_j$ (the maximum target divided by the block's target), and thus chain work at least $W_\text{unit}\,d_j$ (see the note above), where $d_0 = 1$ and $d_{j+1} \in [d_j/R,\ R_\text{max}\cdot d_j]$.
* The "span" of period $j$ is $s_j$, the difference between the timestamps of its last and its first block. Timewarp rule (2) implies $s_j \geq 0$ for all complete periods, i.e., for $j < k$. If the tail period is complete, $s_k$ does exist and is subject to it as well, but plays no role in what follows.
* The "net duration" of period $j$ is $p_j$, the difference between the first-block timestamps of the next period and of this one. This is defined for $j < k$ only, as the tail period has no successor (there is no $p_k$). Timewarp rule (1) implies $p_j \geq s_j - T_\text{grace}$ for all $j < k$.

Finding the exact longest chain that satisfies the time and work bounds is a complicated optimization problem, and does not lead to a simple formula. To obtain a closed-form expression, we make a number of conservative overestimation steps.

The strategy will be as follows, where each numbered step corresponds to a subsection below.
1. **Bound on tail difficulty.** For every possible tail period length $m$ we derive an upper bound $q_m(w)$ on the difficulty of the tail block(s). Then we simplify the problem by looking for an upper bound on the length of any chain that has at most that tail difficulty, rather than bounding the total amount of work. Because $q_m(w)$ is an overestimate, this means computing an upper bound on the length of a *superset* of the set of chains we were considering before, meaning it remains a valid (even if less tight) upper bound.
2. **Ignoring interior timestamps.** Only a few rules have any bearing on the interior timestamps in each period. We ignore these rules, simplifying our problem. Removing rules gives more freedom on allowed chains, and can thus only increase the maximum chain length.
3. **Weakening the difficulty adjustment.** The difficulty adjustment rule is complicated, so we replace it with an approximation that never results in higher difficulties than the real one. This again can only increase our eventual upper bound.
4. **Avoiding span variables.** Next we avoid having $s_j$ variables separate from the $p_j$ variables, by showing how assuming $s_j = p_j + T_\text{grace}$ cannot change the length of the longest chains.
5. **Constructing the longest chain.** After all earlier simplifications, we construct an actual longest chain satisfying all remaining requirements, with a tail period of $m$ blocks, and compute its length $u_m(t,w)$.
6. **Maximizing over tail length.** Finally we get rid of the $m$ variable by taking the maximum over all $u_m(t,w)$ to obtain the overall bound $u(t,w)$.

#### 3.1 Bound on the tail difficulty

There exists a simple upper bound on the tail difficulty. The tail period contains $m$ blocks of difficulty $d_k$, each with chain work at least $W_\text{unit}\,d_k$, so $w \geq m\,W_\text{unit}\,d_k$, and $d_k \leq w/(m\,W_\text{unit})$ would be a valid upper bound, but we can do better.

Start by stating the work limit in terms of a sum of per-period difficulties, using the per-block work bound from the rounding note, and dividing by $W_\text{unit}$:

$$
\frac{w}{W_\text{unit}} \geq N_\text{period}\cdot(d_0+d_1+d_2+\ldots+d_{k-1}) + m\,d_k
$$

Since $d_k \leq R_\text{max}^{k-j}\,d_j$, i.e. $d_j \geq R_\text{max}^{j-k}\,d_k$,

$$
\begin{split}
\frac{w}{W_\text{unit}} &\geq N_\text{period}\cdot(R_\text{max}^{-k}+R_\text{max}^{1-k}+\ldots+R_\text{max}^{-1})\cdot d_k + m\,d_k \\
                   &= N_\text{period}\cdot\frac{1-R_\text{max}^{-k}}{R_\text{max}-1}\cdot d_k + m\,d_k \\
                   &= \frac{N_\text{period}}{R_\text{max}-1}(d_k - R_\text{max}^{-k}\,d_k) + m\,d_k
\end{split}
$$

Applying $R_\text{max}^{-k}\,d_k \leq d_0 = 1$, we get

$$
\frac{w}{W_\text{unit}} \geq \frac{N_\text{period}}{R_\text{max}-1}(d_k-1) + m\,d_k
$$

Solving for $d_k$ gives $d_k \leq q_m(w)$, with
$$
q_m(w) = \frac{N_\text{period}+(R_\text{max}-1)\,w/W_\text{unit}}{N_\text{period}+(R_\text{max}-1)\,m}
$$

From now on, we will look for an upper bound on the length of chains that satisfy the time limit $t$ and have tail difficulty $\leq q_m(w)$, instead of staying below chain work $w$. All chains that stay below chain work $w$ have a tail limit $\leq q_m$, so this is a superset of the earlier set of chains, and the upper bound can only go up.

#### 3.2 Ignoring interior timestamps

There are only two constraints in the current optimization problem that involve the timestamps of interior blocks (all blocks of a period except its first and last):
* The "median-time past" (MTP) rule, which states that each block timestamp must be strictly larger (expressed in integer seconds) than the median of the 11 blocks preceding it. On its own this rule limits chain growth to 6 blocks per second, but combined with the BIP-54 timewarp fixes it no longer affects the longest possible chain, except in extreme cases.
* The upper bound of $t + T_\text{future}$ on all timestamps (relative to genesis).

We simplify both. The MTP rule is dropped entirely, and the timestamp bound will only be applied to the last block in the penultimate period $k-1$. This gives more freedom for chains and may thus increase the length of the longest chain, which is fine for an overestimate. In practice, this increase is minor.

A corollary of this is that all interior timestamps in the chain become irrelevant. They are not constrained by any rule, and do not constrain any other timestamps. This means we can leave them out of consideration entirely, and only care about the first and last timestamps in each period. Assigning timestamp 0 to the genesis block, all relevant timestamps are then determined by the $s_j$ and $p_j$ variables:
* The timestamp of the first block of period $j$ is $\sum_{i=0}^{j-1} p_i$.
* The timestamp of the last block of period $j$ is $s_j + \sum_{i=0}^{j-1} p_i$.

Another corollary is that timestamps may now go below the genesis block's timestamp (0). In real chains the MTP rule prevents that, since every block's timestamp must exceed a median that ultimately traces back to genesis.

#### 3.3 Weakening the difficulty adjustment

Bitcoin's difficulty adjustment rule, in our variables, gives

$$
d_{j+1} \geq \max\left(1, d_j\cdot\max\left(\frac{1}{R}, \min\left(\frac{T_\text{period}}{s_j}, R\right)\right)\right)
$$

with equality except for the rounding of targets, which can only increase $d_{j+1}$ (see the note in the constants list). It is possible to work with this rule directly (see Section 5), but it is complicated. In what follows, we will instead use the conservative approximation:

$$
d_{j+1} = d_j\cdot\exp\left(1 - \frac{s_j}{T_\text{period}}\right)
$$

The plot below compares the real adjustment factor ($\text{adjust}$) with the approximation ($f$) as functions of the span $s_j$; note the logarithmic Y axis.
![adjust_vs_span|690x430](upload://iwYwrK6G4jQO9m8QR9wDfFUeYlk.png)

The approximation never exceeds the real multiplier for any $s_j \geq 0$, and touches it at $s_j = T_\text{period}$, so it is accurate for small adjustments and a (possibly significant) underestimate for large ones.

This makes it an acceptable change, because its only effect is reducing difficulties. As a result, given any chain that satisfies all constraints (timestamp of last block in penultimate period is $\leq t + T_\text{future}$, last block difficulty is $\leq q_m(w)$), applying the approximation instead of the real rule yields a chain that still satisfies them. So the longest chains under the real rule remain within the set our formula bounds.

Note that this requires $R \geq \mathrm{e}$. If $R$ were lower, the approximation at the $s_j=0$ point would be above the real adjustment factor ($R$), so it would no longer be an underestimate. Indeed, if $R$ were so low, we would need a different approach, and a different eventual formula. See Section 5 for more about that.

Many approximations are possible; this one is chosen because its logarithm is linear in $s_j$ (a straight line in the plot above), which will prove very useful below.

With all the changes made so far, our problem statement has turned into: find the maximum number $k$ of $(p_j, s_j)$ pairs satisfying the following constraints:
$$
\begin{align}
\text{(timewarp 1)} \quad & p_j \geq s_j - T_\text{grace} & \text{for } j = 0, \ldots, k-1 \\
\text{(timewarp 2)} \quad & s_j \geq 0 & \text{for } j = 0, \ldots, k-1 \\
\text{(time bound)} \quad & s_{k-1} + \sum_{i=0}^{k-2} p_i \leq t + T_\text{future} \\
\text{(difficulty bound)} \quad & \prod_{i=0}^{k-1} \exp\left(1 - \frac{s_i}{T_\text{period}}\right) \leq q_m(w)
\end{align}
$$

#### 3.4 Avoiding span variables

Next we show why we can assume $s_j = p_j + T_\text{grace}, \forall j \in [0,\,k-1]$, which will allow us to get rid of the $s_j$ variables.

Consider any chain satisfying the constraints above, given as its sequence of $(p_j, s_j)$ pairs, and make the following changes:
* Move the tail period backward as much as possible, without changing any spans. Set $p_{k-1} = s_{k-1} - T_\text{grace}$. This is the minimum timewarp rule (1) allows.
* Grow the spans of all earlier periods as much as possible, without moving any starting time. Set $s_j = p_j + T_\text{grace}, \forall j \in [0, k-2]$. This is the maximum timewarp rule (1) allows.

The result will still satisfy all constraints above. The first change only affects $p_{k-1}$, which only appears in the timewarp rule (1). The second only affects the $s_j$ with $j < k-1$, which appear in both timewarp rules and in the difficulty bound; in the latter, increasing $s_j$ can only reduce the difficulty.

So for every chain in the solution set there is a corresponding chain, with the changes applied, that is also in the set and has the same length. We can therefore restrict attention to chains of that form, since (one of) the longest chains must be among them.

Substituting $s_j = p_j + T_\text{grace}, \forall j \in [0,\,k-1]$ in the problem definition, the problem becomes: find the maximum number $k$ of values $p_j \in [-T_\text{grace},\, \infty)$ satisfying:
$$
\begin{align}
\text{(time bound)} \quad & \sum_{i=0}^{k-1} p_i \leq t + T_\text{future} - T_\text{grace} \\
\text{(difficulty bound)} \quad & \prod_{i=0}^{k-1} \exp\left(1 - \frac{p_i + T_\text{grace}}{T_\text{period}}\right) \leq q_m(w)
\end{align}
$$
Timewarp rule (1) has become trivial, and timewarp rule (2) is now the lower end of the range of $p_j$. From here on we write $h = t + T_\text{future} - T_\text{grace}$ for the right-hand side of the time bound: the available time budget, i.e. the timestamp window minus one grace period. We will also require $h > 0$ (or $t > T_\text{future} - T_\text{grace}$); we only care about long-term behavior and this will simplify things further on.

#### 3.5 Constructing the longest chain

The problem is now close to solvable.

Taking the logarithm of both sides of the difficulty bound, the problem becomes:
$$
\begin{align}
\text{(time bound)} \quad & \sum_{i=0}^{k-1} p_i \leq h \\
\text{(difficulty bound)} \quad & \sum_{i=0}^{k-1} \left(1 - \frac{p_i + T_\text{grace}}{T_\text{period}}\right) \leq \ln q_m(w)
\end{align}
$$

Defining the logarithmic difficulty adjustment as a function of the net period duration,
$$
a(p) = 1 - \frac{p + T_\text{grace}}{T_\text{period}}
$$

this becomes:
$$
\begin{align}
\text{(time bound)} \quad & \sum_{i=0}^{k-1} p_i \leq h \\
\text{(difficulty bound)} \quad & \sum_{i=0}^{k-1} a(p_i) \leq \ln q_m(w)
\end{align}
$$

Note that the ordering of the $p_j$ values no longer matters. That is a consequence of enforcing only a maximum tail difficulty $q_m(w)$, on the blocks of period $k$, instead of the total work limit $w$. For the actual longest chain, the ordering of $p_j$ values absolutely matters: changes in the earlier ones have a compounding effect on all later periods. This makes it clear that replacing the total work limit by a tail difficulty limit leads to an overestimate.

In fact, the constraints can be written purely in terms of the averages $\bar{p}$ of the $p_j$ and $\bar{a}$ of the $a(p_j)$, plus one equation guaranteeing that these averages correspond to actual $p_j$ values. Because $a(p)$ is linear, that equation is simply $\bar{a} = a(\bar{p})$:

$$
\begin{align}
\text{(time bound)} \quad & k\cdot\bar{p} \leq h \\
\text{(difficulty bound)} \quad & k\cdot\bar{a} \leq \ln q_m(w) \\
\text{(correspondence)} \quad & \bar{a} = a(\bar{p}) = 1 - \frac{\bar{p} + T_\text{grace}}{T_\text{period}}
\end{align}
$$

Eliminating $\bar{a}$ and solving for $k$, assuming for now that $0 < \bar{p} < T_\text{period} - T_\text{grace}$:

$$
\begin{align}
\text{(time bound)} \quad & k \leq \frac{h}{\bar{p}} \\
\text{(difficulty bound)} \quad & k \leq \frac{T_\text{period}\,\ln q_m(w)}{T_\text{period} - \bar{p} - T_\text{grace}}
\end{align}
$$

The count $k$ is bounded by the smaller of the two right-hand sides. One is increasing in $\bar{p}$ while the other is decreasing in it, so the largest attainable bound is found at their intersection.

![k_bounds_linear_log|690x430](upload://bjD2J2EddMGvazKruUrCxeNilJR.png)

That intersection satisfies
$$
\frac{h}{\bar{p}} = \frac{T_\text{period}\,\ln q_m(w)}{T_\text{period} - \bar{p} - T_\text{grace}}
$$
with solution
$$
\bar{p} = \frac{(T_\text{period}-T_\text{grace})\,h}{h + T_\text{period}\,\ln q_m(w)}
$$

In other words, *any* chain whose net period durations (excluding the tail period) average to this value can reach the maximum length; giving every period exactly this duration is the simplest way.

Substituting into either bound and rounding down to an integer gives:
$$
k = \left\lfloor\frac{h + T_\text{period}\,\ln q_m(w)}{T_\text{period}-T_\text{grace}}\right\rfloor
$$

The division above assumed $0 < \bar{p} < T_\text{period} - T_\text{grace}$. Provided that:
* $h > 0$ (already assumed)
* $T_\text{period} > T_\text{grace}$ (a period that leaves the difficulty unchanged takes positive net time)
* $w > m\,W_\text{unit}$ (more work than the tail alone can carry at minimum difficulty)

the intersection does lie in that range. Nothing is lost by the restriction either: for $\bar{p} \leq 0$ the time bound holds for every $k$, and the difficulty bound alone caps $k$ at $T_\text{period}\,\ln q_m(w)/(T_\text{period} - T_\text{grace})$. For $\bar{p} \geq T_\text{period} - T_\text{grace}$ the difficulty bound holds for every $k$, and the time bound alone caps $k$ at $h/(T_\text{period} - T_\text{grace})$. The expression inside the floor above is exactly the sum of these two caps, so under the same three conditions it exceeds both, and the intersection is the maximum over all $\bar{p}$. The derivation also presumed $k \geq 1$; chains with no complete period have $n = m$, and the bound below covers them since $\lfloor k\rfloor$ is at least 0 under these conditions.

Thus our upper bound for chain length becomes $n \leq u_m(t,w)$ where
$$
\begin{align}
u_m(t,w) &= N_\text{period}\cdot k + m \\
&= N_\text{period}\cdot\left\lfloor\frac{h + T_\text{period}\,\ln q_m(w)}{T_\text{period}-T_\text{grace}}\right\rfloor + m \\
&= N_\text{period}\cdot\left\lfloor\frac{h + T_\text{period}\,\ln\dfrac{N_\text{period}+(R_\text{max}-1)\,w/W_\text{unit}}{N_\text{period}+(R_\text{max}-1)\,m}}{T_\text{period}-T_\text{grace}}\right\rfloor + m
\end{align}
$$

#### 3.6 Maximizing over tail length

The last step is to eliminate $m$, the number of blocks in the tail period, which is unknown but which the formula so far requires. The obvious approach is to evaluate $u_m(t,w)$ for every $m$ and take the largest. For our example $t$ and $w$ values, this yields 1,012,339 blocks.

![u_m|690x430](upload://sUI2T4orZqCMK7RaySBnUXhTPMv.png)

To obtain a closed-form formula instead, we first drop the floor, which can only increase the value:

$$
u_m(t,w) = N_\text{period}\cdot\frac{h + T_\text{period}\,\ln\dfrac{N_\text{period}+(R_\text{max}-1)\,w/W_\text{unit}}{N_\text{period}+(R_\text{max}-1)\,m}}{T_\text{period}-T_\text{grace}} + m
$$

Its second derivative with respect to $m$ is:
$$
\frac{\partial^2 u}{\partial m^2} = \frac{N_\text{period}\,T_\text{period}\,(R_\text{max}-1)^2}{(T_\text{period}-T_\text{grace})(N_\text{period}+(R_\text{max}-1)\,m)^2}
$$

which is strictly positive, so our upper bound is a convex function in $m$. This means that the maximum over all $m$ is reached at one of the ends of the domain, i.e., either $m=1$ or $m = N_\text{period}$.

Solving $u_1(t,w) \geq u_{N_\text{period}}(t,w)$ shows that $m=1$ wins whenever
$$
\ln\frac{R_\text{max}\,N_\text{period}}{N_\text{period}+R_\text{max}-1} \geq \frac{N_\text{period}-1}{N_\text{period}}\cdot\frac{T_\text{period}-T_\text{grace}}{T_\text{period}}
$$

which holds for our constants, so $u_1(t,w) \geq u_m(t,w)$ for all $m$. Setting $m=1$ gives the final formula:

$$
u(t,w) = N_\text{period}\cdot\frac{h + T_\text{period}\,\ln\dfrac{N_\text{period}+(R_\text{max}-1)\,w/W_\text{unit}}{N_\text{period}+R_\text{max}-1}}{T_\text{period}-T_\text{grace}} + 1
$$

which holds whenever:
* $h = t + T_\text{future} - T_\text{grace} > 0$
* $w > W_\text{unit}$ (true for any chain, as the genesis block alone has chain work $\lfloor 2^{48}/65535\rfloor = 4295032833$)
* $w \leq 2^{208}$ (needed for the per-block work bound; see the rounding note)
* $R \geq \mathrm{e}$
* $\ln\!\bigl(R_\text{max}\,N_\text{period}/(N_\text{period}+R_\text{max}-1)\bigr) \geq (N_\text{period}-1)\,(T_\text{period}-T_\text{grace})/(N_\text{period}\,T_\text{period})$
* $T_\text{period} > T_\text{grace}$

Note that eliminating $m$ relied on removing the floor to make the expression convex, and the floor cannot be reinstated afterwards: $m=1$ is the maximum only without it. With the floor, the maximum can occur at any $m \in [1,\, N_\text{period}]$, as the plot above shows.

Substituting the constants with their values gives the formula presented at the top. Note that $h = t$, since $T_\text{future} = T_\text{grace}$.

### 4. Lean proof

The formula from Section 2 has been proven in Lean 4 (with Mathlib). The [proof](https://bitcoin.sipa.be/bip54proof.tgz) was created by an LLM and has not been reviewed by me. It can be verified independently by Lean's kernel, without `sorry`, and depends only on the three standard axioms (`propext`, `Classical.choice`, `Quot.sound`). Assuming no bugs in Lean and Mathlib, this means only the statement being proven needs to be reviewed:

```lean
chain_length_le : ∀ (n : ℕ) (x : ℕ → ℤ) (g : ℕ → ℕ) (t : ℕ) (w : ℝ),
  1 ≤ n →
  x 0 = 0 →
  (∀ i < n, x i ≤ ↑t + 7200) →
  (∀ (j : ℕ), 0 < j → 2016 * j < n → x (2016 * j - 1) - 7200 ≤ x (2016 * j)) →
  (∀ (j : ℕ), 2016 * j + 2015 < n → x (2016 * j) ≤ x (2016 * j + 2015)) →
  g 0 = powLimit →
  (∀ (j : ℕ), 2016 * j + 2015 < n → g (j + 1) = nextTarget (g j) (x (2016 * j + 2015) - x (2016 * j))) →
  w ≤ 2 ^ 208 →
  ∑ i ∈ Finset.range n, ↑(work (g (i / 2016))) ≤ w →
  ↑n ≤ u t w
def u : ℕ → ℝ → ℝ := fun t w =>
  2016 / (2016 * 600 - 7200) *
  (↑t + 2016 * 600 * Real.log ((2016 + (3 + 3 / 2 ^ 15) * (w / Wunit)) / (2016 + (3 + 3 / 2 ^ 15)))) + 1
def Wunit : ℝ := 2 ^ 32 + 2 ^ 16
def work : ℕ → ℕ := fun g => 2 ^ 256 / (g + 1)
def nextTarget : ℕ → ℤ → ℕ := fun g s => compactRound (min powLimit (g * clampSpan s / 1209600))
def clampSpan : ℤ → ℕ := fun s => (max (1209600 / 4) (min (4 * 1209600) s)).toNat
def compactRound : ℕ → ℕ := fun x => setCompact (getCompact x)
def getCompact : ℕ → ℕ × ℕ := fun x => if 2 ^ 23 ≤ mant x then (mant x / 2 ^ 8, nSize x + 1) else (mant x, nSize x)
def setCompact : ℕ × ℕ → ℕ := fun p => if p.2 ≤ 3 then p.1 / 2 ^ (8 * (3 - p.2)) else p.1 * 2 ^ (8 * (p.2 - 3))
def mant : ℕ → ℕ := fun x => if nSize x ≤ 3 then x * 2 ^ (8 * (3 - nSize x)) else x / 2 ^ (8 * (nSize x - 3))
def nSize : ℕ → ℕ := fun x => (x.size + 7) / 8
def powLimit : ℕ := 65535 * 2 ^ 208
```

In which $n$ is the chain length, $t$ is the time bound, $w$ is the work bound, $x$ are the block timestamps, $u$ is the Section 2 formula, and $g$ are the per-period targets. $g\,j$ is the target of blocks $2016j$ to $2016j+2015$, so block $i$ uses $g\,(i/2016)$; $g\,0$ is the maximum target, and later ones follow from the retarget rule. The arrows `↑` are casts between number types (natural numbers to integers or reals).

The hypotheses, in order, are:
* at least one block
* genesis at timestamp 0
* the timestamp window
* timewarp rule (1), the first block of each period being at most 7200 seconds before the previous block
* timewarp rule (2), the last block of each complete period not being before its first
* the genesis target
* the retarget rule, computing each period's target from the previous period's target and the timestamps of that period's first and last blocks
* the cap $w \leq 2^{208}$ needed for the per-block work bound (see the rounding note)
* the work budget, the total chain work being at most $w$

Retargeting and the chain work calculation are modelled exactly as in Bitcoin's consensus rules, including the clamping of the timespan, the truncation of targets to the compact `nBits` encoding (`GetCompact` in Bitcoin Core), and the integer divisions.

The theorem is somewhat stronger than the derivation in Section 3: it needs neither $h > 0$ nor $w > W_\text{unit}$. The proof follows a slightly different route from the text, one that does not need those conditions.

### 5. More accuracy

In Section 3.3 we made a conservative approximation to the difficulty adjustment formula. This is not strictly necessary: a tighter bound can be obtained that is still a closed-form expression.

We do still ignore the minimum-difficulty rule (difficulty $\geq 1$), as it is hard to model and has no effect on the longest chain for realistic values; where it does have an effect, ignoring it only underestimates difficulty, and so overestimates length. The adjustment rule is thus:

$$
d_{j+1} \geq d_j\cdot\max\left(\frac{1}{R}, \min\left(\frac{T_\text{period}}{s_j}, R\right)\right)
$$

where, as in Section 3.3, the inequality is an equality up to the rounding of targets. Replacing the actual rule by this lower bound can only enlarge the set of admissible chains, so we use it as the adjustment rule from here on.

Restating the problem of Section 3.5 with this adjustment rule in place of the approximation from Section 3.3: find the maximum number $k$ of values $p_j \in [-T_\text{grace}, \infty)$ satisfying:
$$
\begin{align}
\text{(time bound)} \quad & \sum_{i=0}^{k-1} p_i \leq h \\
\text{(difficulty bound)} \quad & \sum_{i=0}^{k-1} \ln\max\left(\frac{1}{R}, \min\left(\frac{T_\text{period}}{p_i + T_\text{grace}}, R\right)\right) \leq \ln q_m(w)
\end{align}
$$

Defining
$$
b(p) = \ln\max\left(\frac{1}{R}, \min\left(\frac{T_\text{period}}{p + T_\text{grace}}, R\right)\right)
$$

this becomes:
$$
\begin{align}
\text{(time bound)} \quad & \sum_{i=0}^{k-1} p_i \leq h \\
\text{(difficulty bound)} \quad & \sum_{i=0}^{k-1} b(p_i) \leq \ln q_m(w)
\end{align}
$$

As with $a(p)$ in Section 3.5, this can be stated in terms of the averages $\bar{p}$ of the $p_j$ and $\bar{b}$ of the $b(p_j)$:
$$
\begin{align}
\text{(time bound)} \quad & k\cdot\bar{p} \leq h \\
\text{(difficulty bound)} \quad & k\cdot\bar{b} \leq \ln q_m(w)
\end{align}
$$

We again need a correspondence condition guaranteeing that $(\bar{p}, \bar{b})$ is the average of actual $(p_j, b(p_j))$ pairs. Consider what $b(p)$ looks like:

![b_hull|690x430](upload://b7t6He8Od4QQiaDaXFfDR3gyeNT.png)

Each $(p_j, b(p_j))$ is a point on the blue curve. Their average $(\bar{p}, \bar{b})$, however, can be a combination of multiple such points, and lie beyond the curve itself. The exact set of those combinations is complicated, but it is bounded by the convex hull of the curve (light blue). As a further relaxation of the problem, we accept that the average can lie anywhere in that hull. Note in particular that this includes the area on the far left *under* the blue curve. The lower border of the hull (dotted orange) is:
$$
\underline{b}(p) =
\begin{cases}
\ln R - \dfrac{R}{\mathrm{e}}\cdot\dfrac{p + T_\text{grace}}{T_\text{period}},
  & -T_\text{grace} \leq p \leq \dfrac{\mathrm{e}}{R}\,T_\text{period} - T_\text{grace} \\[3ex]
\ln\dfrac{T_\text{period}}{p + T_\text{grace}},
  & \dfrac{\mathrm{e}}{R}\,T_\text{period} - T_\text{grace} \leq p \leq R\,T_\text{period} - T_\text{grace} \\[3ex]
-\ln R,
  & p \geq R\,T_\text{period} - T_\text{grace}
\end{cases}
$$
with its three pieces (below the curve on the left, on the curve in the middle, on the maximum-adjustment floor on the right) separated by the orange dots. This is correct for $R \geq \sqrt{\mathrm{e}}$; otherwise the middle section disappears, and the first section needs a different formula.

The correspondence condition is then $\bar{b} \geq \underline{b}(\bar{p})$, and using it to solve for $k$ gives:

$$
\begin{align}
\text{(time bound)} \quad & k \leq \frac{h}{\bar{p}} \\
\text{(difficulty bound)} \quad & k \leq \frac{\ln q_m(w)}{\underline{b}(\bar{p})}
\end{align}
$$

As before, the maximum $k$ is at the intersection of the two right-hand sides, and the solution depends on which of the three pieces of $\underline{b}$ it lands on. Skipping the derivation, the result is:

$$
u_m(t,w) =
\begin{cases}
N_\text{period}\cdot\left\lfloor
\dfrac{h + \dfrac{\mathrm{e}}{R}\,T_\text{period}\,\ln q_m(w)}{\dfrac{\mathrm{e}}{R}\,T_\text{period}\,\ln R - T_\text{grace}}
\right\rfloor + m,
  & \lambda \geq \lambda_\text{max} \\[5ex]
N_\text{period}\cdot\left\lfloor
\dfrac{\ln q_m(w)}{W_0\left(\lambda\,T_\text{period}\,\mathrm{e}^{\lambda\,T_\text{grace}}\right) - \lambda\,T_\text{grace}}
\right\rfloor + m,
  & 0 < \lambda \leq \lambda_\text{max} 
\end{cases}
$$
where

$$
q_m(w) = \frac{N_\text{period}+(R_\text{max}-1)\,w/W_\text{unit}}{N_\text{period}+(R_\text{max}-1)\,m},\qquad \\[3ex]
\lambda = \frac{\ln q_m(w)}{h},\qquad
\lambda_\text{max} = \frac{\ln R - 1}{\dfrac{\mathrm{e}}{R}\,T_\text{period} - T_\text{grace}}
$$

and where $W_0$ is the principal branch of the [Lambert W function](https://en.wikipedia.org/wiki/Lambert_W_function), the inverse of $x\,\mathrm{e}^x$. Note that $\lambda \leq 0$ is not possible, as we assume $h > 0$ and $w > m\,W_\text{unit}$ (as in Section 3.5).

Restricting to the $0 < \lambda \leq \lambda_\text{max}$ case and applying the convexity analysis of Section 3.6 gives:
$$
u(t,w) = N_\text{period}\cdot
\frac{h\,\ln q_1(w)}
{h\,W_0\left(\dfrac{T_\text{period}\,\ln q_1(w)}{h}\;q_1(w)^{T_\text{grace}/h}\right) - T_\text{grace}\,\ln q_1(w)}
+ 1
$$

The convexity is no longer automatic here, but holds whenever the bound allows at least one complete period even at $m = N_\text{period}$. Also, whether $m=1$ or $m=N_\text{period}$ wins may now depend on $t$ and $w$ as well, though for our constants $m=1$ always wins. For our example, this gives 1,009,930 blocks; evaluating the floored form over all $m$ (restricted to $m < w/W_\text{unit}$, the tail lengths for which a chain can exist at all; for our example that is every $m$) gives 1,009,199, our tightest formula-based bound.

In the $\lambda > \lambda_\text{max}$ case, $(\bar{p},\bar{b})$ lies on the leftmost piece of the orange border, below the blue curve. A chain achieving this mixes periods with $p_j = -T_\text{grace}$ (the leftmost point of the curve) with periods at the tangency point $p_j = (\mathrm{e}/R)\cdot T_\text{period} - T_\text{grace}$. This is exactly the (weak) variant of the Murch-Zawy attack I pointed out [here](https://delvingbitcoin.org/t/zawy-s-alternating-timestamp-attack/1062/9) as becoming possible if $R < \mathrm{e}$. That turns out not to be necessary: it is possible for $R=4$ too, but only when the affordable difficulty increase per unit of time is several times higher than in the current chain. If $R \leq \mathrm{e}$, however, this mixing strategy becomes optimal for all inputs.

### 6. Conclusion

By making a number of conservative simplifications to the problem, we obtained several upper-bound formulas for the length of any possible chain with a specified time limit and work limit. One of those was proven correct in Lean. All of these are within 5% of the real chain's length, showing that the two timewarp rules together do bound chain growth tightly.

For our running example $t$ and $w$, we get:

| Description | n | k | m |
|---|---|---|---|
| Section 3, no floor, $m=1$ (proven formula) | 1,012,794 | 502.38 | 1 |
| Section 3, $\lfloor k\rfloor$, maximum over $m$ | 1,012,339 | 502 | 307 |
| Section 5, no floor, $m=1$ | 1,009,930 | 500.96 | 1 |
| Section 5, $\lfloor k\rfloor$, maximum over $m$ | 1,009,199 | 500 | 1199 |
| Longest known chain | 1,007,326 | 499 | 1342 |
| Real chain (height 966,270) | 966,271 | 479 | 607 |

### Acknowledgements

Thanks to @AntoineP for the idea to look into this problem, and reviewing the text.

-------------------------

zawy | 2026-09-21 16:59:05 UTC | #2

Replacing the two rules with a monotonic timestamp requirement should be a lot easier to prove, safer, and removes the need to use or be aware of MTP. [edit: this statement was wrong] 

Allowing "negative solvetimes" can be thought of as allowing the DAA to see "negative work", greatly lowering difficulty. 

A different way to view it is if a consensus mechanism needs to progress from one state to the next, it needs causality which needs ordering. The height isn't sufficient. We only use it as a label. The proof of work (DAA) looks at the timestamps to keep the consensus ordering correct. Allowing negative work messes it up.

-------------------------

sipa | 2026-09-18 14:31:45 UTC | #3

[quote="zawy, post:2, topic:2899"]
Replacing the two rules with a monotonic timestamp requirement should be a lot easier to prove
[/quote]

I don't think it matters. As far as the first/last timestamps of blocks are concerned (which are the only ones that matter for difficulty adjustment), such a rule would be equivalent to the two BIP-54 timewarp rules, but with grace time 0s instead of 7200s. A proof for an upper bound on its maximum length should be nearly equivalent to this here.

[quote="zawy, post:2, topic:2899"]
Allowing “negative solvetimes” can be thought of as allowing the DAA to see “negative work”, greatly lowering difficulty.
[/quote]

Timewarp rule 2 disallows negative period durations. It would be reasonable to assume that permitting time to go backward messes things up, but this work shows that together with the two timewarp rules, that really only has a minor impact.

A monotonic timestamp rule would mean an approximate bound of
$$
u(t,w) \approx \frac{t}{600} + 1397.385\log_2 w - 57830.9
$$
blocks rather than the BIP-54 bound of
$$
u(t,w) \approx \frac{t}{596.4285} + 1405.753\log_2 w - 58189.3
$$

So around a 0.6% difference in maximum chain length (pretty much exactly 7200s per two weeks).

-------------------------

zawy | 2026-09-19 16:30:42 UTC | #4

[quote="sipa, post:1, topic:2899"]
As of 2026-Sep-09 21:30 UTC, with $t = 557982895$ seconds after genesis, and a real chain at height 966,270 with total work $w = 2^{96.349377}$, this formula gives a limit of 1,012,794 blocks. ... the longest known chain is 1,007,326 blocks long, but it is not proven to be the longest.
[/quote]

I used ChatGPT to check this. I asked it to maximize the number of blocks given the above t and w and it got a limit that was 0.6% smaller, 1,005,984 blocks. The reason it's smaller is because I restricted the AI to an end point that falls on a full 2016 period which implies I could be up to 2,016 blocks less than the optimum. It showed that if the history of the chain was longer by ~830 blocks, it could get complete another period  of 2016 blocks to satisfy my prompt. The difference is 2016 - 830 + 1,005,984 = 1,007,170.  Going the other way, if the history were shorter, it would have been 1,170 + 1,005,984 = 1,007,154. So 1,007,160 seems about right, 166 blocks shy of PW's "longest known".

I didn't attempt to adjust for nBits error, the 2016/2015 timespan error that increases difficulty too much, or the Erlang error that over-estimates work by 2016/2015. I believe these last two factors cause difficulty to be constantly set too high by (2016/2015)^2. It over-estimates the actual work that was performed by about 0.1%, giving me 1,006,161 based on actual work instead of chain work.

The 7200 manipulation enabled 6% more "excess" blocks verses monotonic (76,000 instead of 72,000). 

Simple solutions the AI found involved setting the timestamps the same to increase difficulty by about 7% every 2016 period until the last two periods where it set timespan to about 1/10, exploiting the 1/4 timespan limit. This was probably my motivation:  I wanted to see an outline of how optimal solutions worked.

**The prompt:**

Notation:

- A, B, C ... = difficulty = hashes required for each 2016 period.
- A = 1st period at genesis equal to number of hashes in that period
- a, b, c ... = timespan for each of the above
- t =  a period of time = 600 * 2016 scaled to 1. 
- w = work = A+B+C ...
- k = adjustment for cheating with the FTL at the end of each period.

This was my prompt:

> There are N+1 terms A, B, C .... The sum to the first N terms is w = 2^96.349. Let A = 2016 \* 2^32, B = A/(a \* k), C = B/(b \* k), D=C/(c \* k) ... where k = 2016 \* 600/(2016 \* 600 - 7200). The sum a+b+c+d+... to the Nth term is a constant t = 461.295 [clarification this periods since genesis = t / 2016 / 600 ]. If any of those divisors a \* k, b \* k, c \* k, d \* k, ... are < 0.25 then use 0.25 in the divisors. If one is >4, then the divisor is 4. The a, b, c ... values are greater than 0. The B, C, D, ... terms cannot be less than A. Select a, b, c, d ... to maximize N. After finding the solution, check to see if anda solution just as good can use a = b = c .... to N-2 as a constant and a little smaller than 1 and the last two terms a lot closer to zero.

https://chatgpt.com/share/6aaeaf4a-f984-83ea-8068-e0415ec4d0bf

-------------------------

sipa | 2026-09-19 17:26:02 UTC | #5

That sounds correct. My longest known chain for the example $t, w$ was found by Claude Fable 5.1, and seems to agree with your solution (while also satisfying the exact consensus rules including `nBits` rounding). If I ask it to restrict to $m=2016$ (complete periods only), it also finds 1,005,984 blocks (499 periods). Interestingly, they are exactly the same as the any-$m$ longest, with the tail cut off.

[Here](https://bitcoin.sipa.be/longest_chain_periods.csv) is a CSV with the full period data. It consists of ~460 periods around 13 days each, then increasingly shorter periods down to 10 days at period 496, two period with net duration 0, and then a tail period of 1342 blocks all with the minimum timestamp.

Absent integer / rounding effects, you can show that an optimal chain will have periods of monotonically decreasing length (because a longer period followed by a shorter period can be swapped, resulting in less total work), which can be seen in this solution.

The inverse gamma distribution effect that causes difficulty to be off by a factor 2016/2014 (rather than the 2016/2015 that would be expected due to the off-by-one in difficulty adjustment) does not apply here. That is an effect that honest miners are subject to who use real timestamps for blocks. An adversary that can choose timestamps freely is not bound by probabilistic effects like these, they make their own luck.

-------------------------

zawy | 2026-09-19 20:45:21 UTC | #6

It appears the solutions are using the timespan limit in the last 3 timestamps to profit. It says there are no solutions as good if it doesn't let timespan go less than 0.25, so there seems to be a hack on it. The gain from doing it is 932 blocks. Setting every timespan the same costs 4,669 blocks (optimal was 7.12% increase per period).

-------------------------

zawy | 2026-09-20 00:01:48 UTC | #7

With honest work w we read from the chain we know the actual work is w' = 2016 / 2014 * w, so we use w' in the equation (which give 1.9 more blocks than w), but not in other places that start with an accurate w.  [edit: think i should have said 2014 / 2016 instead of 2016 / 2014 ]

-------------------------

sipa | 2026-09-19 21:17:16 UTC | #8

If you want to use a correction factor like that, it should be 2016/2015, not 2016/2014. The latter is only relevant for honest miners that set timestamps to real time (causing periods to have an Erlang distribution). Attackers are not subject to that probability distribution, they can choose their timestamps strategically.

-------------------------

zawy | 2026-09-19 21:42:00 UTC | #9

Yes, I understood that. What I'm saying is that you read w from the existing bitcoin blockchain in this particular example I quoted. I believe the chain used mostly honest timestamps and has not had any correction to w that we see as the total chain work. So you can't use that particular w for modelling or the boundary equation to say "an attacker with this w can get u blocks in time t" because it's not the true w that the blockchain had. The attacker should get the same w' that the honest miners got when we compare them.

-------------------------

sipa | 2026-09-19 21:48:15 UTC | #10

Ah, I see what you mean now. I believe that's not the case, at least for my use case / inspiration for this problem.

$w$, as read from the chain by summing real blocks' $\lfloor 2^{256} / (\text{target}+1)\rfloor$, is (a good approximation of) the actual work performed by miners. The fact that difficulty adjustments are slightly biased isn't relevant here. The difficulties are lower than "intended" perhaps, but the amount of work performed was also lower, per block, and compensated for by creating more blocks. An attacker creating their chain also needs to perform work (very close to) their chain's cumulative work, computed by the same formula. Thus, comparing the attacker's total chain work to the honest chain's total chain work is fair.

Even more practically, the motivation for investigating this was how the adoption of BIP-54 (if deployed, activated, and eventually buried) would affect the [headers presync](https://bitcoin.stackexchange.com/a/121235/208) DoS protection logic. In that context, we have a `minchainwork` variable, which is the larger of a hardcoded value in the software ("we know a chain with at least this much work exists") and the best known chain's total chainwork so far. The DoS protection is intended to prevent downloading and filling disk with low-difficulty spam headers that never amount to a full chain. If a peer can provide a chain that meets `minchainwork`, they're not an attacker, because they have a legitimate PoW-rich chain we are interested in. Thus, attackers are limited to chains with work $\leq w = \text{minchainwork}$, and to abide by our time rules (timestamps $< t$), and we would like to bound how large such chains can get. Right now, that logic assumes $6 t$ because that's what's possible with the MTP rule. With the BIP-54 timewarp fix rules in place, and I was curious in knowing how much. It turns out the answer is drastically, and the maximum chain length would become just a bit longer than the actual chain.

-------------------------

zawy | 2026-09-20 00:40:15 UTC | #11

It seems like you could weed out a lot of DoS by requiring the lowest-hash blocks first.

-------------------------

sipa | 2026-09-20 00:48:17 UTC | #12

Absolutely, that was suggested as far back as [2012](https://en.bitcoin.it/wiki/User:Gmaxwell/Reverse_header-fetching_sync) even.

But by 2024 the P2P changes needed for that hadn't been worked on, and we [needed](https://bitcoincore.org/en/2024/09/18/disclose-headers-oom/) a solution fairly urgently, as headers-spam attacks had become fairly cheap. The presync idea solved it completely, without needing any P2P protocol changes or waiting for deployments, at the cost of downloading headers twice (a miniscule amount of data compared to the full blockchain).

-------------------------

zawy | 2026-09-20 14:45:43 UTC | #13

It's nice that working to stop an attack requiring >50% hashrate (i.e. something of dubious utility except on testnet) could result in something important to daily protection. Same thing with monotonicity: side benefits would pop up.

I had an error in checking it for the 7200 hack. 499 periods can be found in 460.50455756 periods of real time for the 2^96.349377 work. That's 77,607 extra blocks, 8.36% extra. 460.504 periods is less than the 461.295 periods in the example, so the following result is a slight under-estimate. Add those 77,607 extra blocks to 461.295 periods of real time = 1,007,571 blocks. 

I asked ChatGPT to develop a formula like u(t,w), but I asked it backwards, to find t(u,w).  The result is very nice with **only 0.029% error too low** for the above example.

Given that an attacker can find U periods of blocks with w hashes, what real time T periods does he need to do it?
\[
T_{\min}(w,U)\approx
U\left[
\left(\frac{w_0}{w}\right)^{1/U} - \frac{7200}{2016 * 600}
\right]
\]
 $w_0$ is the hashes required for the genesis period, 2016 * 2^32. Maybe this equation is just the result of setting all the timespans to the same value.

The prompt:

> There are N+1 terms A, B, C .... The sum to the first N terms is w = 2^96.349377. Let A = 2016 \* 2^32, B = A/a, C = B/b, D=C/c .... The sum a+b+c+d+... to the Nth term is t+N \* 7200/600/2016 [edit: real time t is shorter than sum of timespans which can be +7200 more per block from the cheat]. If any of those divisors a, b, c, d ... are < 0.25 then use 0.25 in the divisors. If one is >4, then the divisor is 4. The a, b, c ... values are >= 1/2015. The B, C, D ... values are > A. Let N=499. Select a, b, c, d ... to minimize t.

**[The Output](https://chatgpt.com/share/6aafd71e-bc64-83ea-95a3-1366927e026f)**

We mentioned it previously, but it's annoying (if not concerning) that the 7200 allowance enables difficulty to be constantly lowered 0.6% in every epoch without advancing time, or holding time back by 0.6% without difficulty increasing, or doing a combination of lowering difficulty * time = 0.6%.  It is only assisting in getting more blocks in this situation because increasing difficulty by 5% has a much larger effect..

-------------------------

zawy | 2026-09-20 15:25:19 UTC | #14

I said the 7200 couldn't cheat enough to get unlimited blocks because 7200 / 2016 / 600 is too small at only 0.6%. From the T(w,U) equation it looks like a lower w and higher U can result in a negative T which means unlimited blocks in finite time. I may have mentioned this two years ago, but it was dismissed as not something the chain was going to exhibit anytime soon. The geometric mean of the w / w_0 increase over periods would have to be less than 0.6%.

-------------------------

sipa | 2026-09-20 15:24:40 UTC | #15

The formula presented in my document, if correct, guarantees that infinite blocks in finite time are impossible. Do you disagree with that conclusion, or are you talking about something else?

-------------------------

zawy | 2026-09-20 15:28:14 UTC | #16

I am saying if the geometric mean (based on U periods) of total the chain work increase over genesis is less than 7200 / 600 / 2016  (= 0.6% per period) then the conclusion is wrong.

-------------------------

sipa | 2026-09-20 15:32:32 UTC | #17

The Lean proof covers all cases where:
* There is at least one block
* $t \geq 0$ and no blocks have a timestamp more than $t + 7200$ seconds after genesis
* The work bound $w \leq 2^{208}$, and the total amount of work, as computed by $\lfloor 2^{256} / (\text{target} + 1) \rfloor$, with consensus-accurate difficulty adjustment does not exceed $w$.

Are you claiming something about conditions outside of this, or that there is a bug in the statement being proven, or that there is a bug in Lean?

-------------------------

zawy | 2026-09-20 18:38:58 UTC | #18

I believe I was mistaken and confused.

-------------------------

zawy | 2026-09-21 20:16:28 UTC | #19

I haven't replied because I have trouble understanding your posts and they will take me a long time to digest.  You might seen my discussion with PW on delving bitcoin and something relevant to your needs came up: 

This equation can be used in any DAA (for the most part) to determine minimum time T it takes to get U blocks with work $w$ in T over and above the public chain's work $w_0$ in T.  This equation is only valid for large $w/w_{0}$

$T_{\min}(w,U) \approx U\left[ \left(\frac{w_0}{w}\right)^{1/U} \right] $

- $U$ is the number of blocks he'll get in units of "effective averaging windows" for the DAA. In BTC, U = 2 for 2 * 2016 blocks.
- $T_{min}$ is the time it will take him in units of the averaging window.   In BTC, T = 2 for 2 * 2016 * 600 seconds.
- $w / w_0$ is attacker's ratio of work in done in T (aka hashrate) over the public chain's work done in the first averaging period. 

(continued in next comment)

-------------------------

zawy | 2026-09-21 19:08:55 UTC | #20

I want to use the equation above in the context of an attacker doing a private more to get more blocks than he should. Honest chain with $w_{0} hashes in the first averaging window of T windows. If difficulty doesn't change $w = w_{0} * T$.  Time in seconds is t = T * [difficulty averaging window timeframe]. For ASERT's half-life of 288 * 600, the "mean lifetime" is 288/ln(2) = 514 * 600.  One T in ASERT or EMA is therefore 2 * 512 * 600 seconds. For SMA and Bitcoin, it's just the number of blocks in the window times the block time.  For other DAA's there's an "effective window" adjustment factor. 

Let attacker have X more hashrate than network so that $w/w_0 = X * T$.  There's a way to assign timestamps to get the max number of blocks (U) in the least T due to the DAA not being able to keep up and thereby getting U > T.

The timestamps (how fast the attacker claims he solved each of the U difficulty windows) matter a lot for smaller X and smaller U.  The equation is terrible for small X and N. These are the actual results:

![image|690x217](upload://48S2PE9Zh6jQ2e5wByggspP8C8y.png)

So T > U but not by much due to the DAA not keeping up.

For small X and U, T can be calculated with some difficulty: (N = U)

![image|653x499](upload://taSHbusE6MWmFXhXJqrlcmnr4R2.png)


A fix for this kind of attack (and the more common simple "hit and run" mining) was [recently proposed](https://github.com/zawy12/difficulty-algorithms/issues/89). In short, the rewards are distributed once per week, divided equally between the blocks that were found during the week.  So if too many were found, rewards are less. This makes a lot of sense if miners are paid to "competitively advance the chain in time" instead of for "competitively hashing".

-------------------------

sipa | 2026-09-21 21:33:23 UTC | #21

@zawy It's a nice idea to write it as a lower bound on the time $t$ in function of a lower bound on the chain length $n$ and an upper bound on the amount of work $w$. It yields significantly simpler expressions.

For the approach with the simplified difficulty adjustment from Section 3.3, I get:
$$
  t \;\geq\; \frac{n-1}{N_\text{period}}\left(T_\text{period} - T_\text{grace}\right)
   \;-\; T_\text{period}\,
   \cdot\ln\frac{N_\text{period} + (R_\text{max}-1)\,\frac{w}{W_\text{unit}}}
           {N_\text{period} + R_\text{max} - 1}
$$

For the approach without that simplification, i.e., the inverse of the formula with the Lambert W from Section 4, I get:
$$
  t \;\geq\; \frac{n-1}{N_\text{period}}
   \left[\,T_\text{period}
   \cdot\left(\frac{N_\text{period} + (R_\text{max}-1)\,\frac{w}{W_\text{unit}}}
              {N_\text{period} + R_\text{max} - 1}\right)^{\textstyle -\dfrac{N_\text{period}}{n-1}}
   \;-\; T_\text{grace}\right]
$$
which looks similar in structure to your approximate one. I don't have a Lean proof for these, but they're created using the same approach that was proven correct already, so I believe they are proper lower bounds on $t$. Both of them come with some constraints on the constants (which are satisfied for us), and the second one requires $n$ to be sufficiently large (to be past the alternating span regime, see Section 4).

---

Inspired by that, it may be even more interesting to have a lower bound on $w$ given a lower bound on chain length $n$ and an upper bound on time $t$, i.e., "how much work does an attacker need to do to create an attack chain with certain properties?".

For that, without the simplification (which seems of little help in this case), I get:

$$
  w \;\geq\; \frac{W_\text{unit}}{R_\text{max}-1}\left[
   \left(N_\text{period} + R_\text{max} - 1\right)
   \left(\frac{T_\text{period}}
              {\frac{N_\text{period}\,t}{n-1} + T_\text{grace}}\right)^{\textstyle \frac{n-1}{N_\text{period}}}
   \;-\; N_\text{period}\right]
$$

For the Bitcoin + BIP-54 constants, that is:

$$
  w \;\geq\; \frac{4295032832}{10923}
  \left[7350955\left(\frac{4200\,(n-1)}{7t+25\,(n-1)}\right)^{\dfrac{n-1}{2016}}
  -\;7340032\right]
$$

-------------------------

zawy | 2026-09-22 00:43:53 UTC | #22

It had crossed my mind that a lower bound on w might be useful. <strike>Using 2 of 3 boundaries is interesting in creating a hard limit on the 3rd instead of it having a long statistical tail.</strike> The t comes down as the other two go up. 

Mean hashrate per block = w / t / n.  I guess that value describes a <strike>volume</strike> surface of possibilities. I believe that's  the scarce resource that creates our trilemma.

I was trying to conceptualize the new problem and found thinking about hashrate per block helps: He has more than w done in less than t, so it's a > hashrate that must result in more than n blocks.  Or maybe given more than n blocks, the minimal hashrate must have been...

-------------------------

sipa | 2026-09-22 01:12:08 UTC | #23

Ok, [proven](https://bitcoin.sipa.be/bip54proof.tgz) now.

If:
* The number of blocks $n$ is at least 1.
* No block has timestamp more than $t + 7200$ seconds after genesis, with $t \geq 0$.
* The timestamps of the blocks satisfy both BIP-54 timewarp fix rules.
* Difficulty adjustment follows existing consensus rules, including integer rounding and conversion to `nbits`.
* The total chainwork $w$ of the blocks (computed as $\sum \lfloor 2^{256} / \text{target} \rfloor$) is at most $2^{208}$.

Then:
$$
\begin{gathered} 
  w \;\geq\; \frac{4295032832}{10923}\left[ 
  7350955\,{\alpha'}^{-\frac{n-1}{2016}} 
  -\;7340032\right]\\[2.5ex] 
  \alpha=\frac{t}{600\,(n-1)}+\frac{1}{168},\qquad 
  \alpha'=\begin{cases} 
  \alpha, & \alpha\ge\mathrm{e}/4\\[1ex] 
  \mathrm{e}^{4\alpha/\mathrm{e}}/4, & \alpha\le\mathrm{e}/4 
  \end{cases} 
  \end{gathered}
$$

The factor $\alpha$ here is the expected period span divided by 2 weeks, and can normally be used directly. When it is very low however (average period spans under ~9.5 days, or average difficulty adjustments over 1.47x), the corrected $\alpha'$ needs to be used. This is the regime where it becomes profitable to mix/alternative span-0 periods and longer periods.

The exact proven statement is

```lean
work_ge_wlow : ∀ (n : ℕ) (x : ℕ → ℤ) (g : ℕ → ℕ) (t : ℕ),                                                                                                                                    
  1 ≤ n →
  x 0 = 0 →
  (∀ i < n, x i ≤ ↑t + 7200) →
  (∀ (j : ℕ), 0 < j → 2016 * j < n → x (2016 * j - 1) - 7200 ≤ x (2016 * j)) →
  (∀ (j : ℕ), 2016 * j + 2015 < n → x (2016 * j) ≤ x (2016 * j + 2015)) →
  g 0 = powLimit →
  (∀ (j : ℕ), 2016 * j + 2015 < n → g (j + 1) = nextTarget (g j) (x (2016 * j + 2015) - x (2016 * j))) →
  ∑ i ∈ Finset.range n, ↑(work (g (i / 2016))) ≤ 2 ^ 208 →
  wlow n t ≤ ∑ i ∈ Finset.range n, ↑(work (g (i / 2016)))

def BIP54.wlow : ℕ → ℕ → ℝ := fun n t => 4295032832 / 10923 * (7350955 * α' n t ^ (-((↑n - 1) / 2016)) - 7340032)
def BIP54.α' : ℕ → ℕ → ℝ := fun n t => if Real.exp 1 / 4 ≤ α n t then α n t else Real.exp (4 * α n t / Real.exp 1) / 4                                                                                                   
def BIP54.α : ℕ → ℕ → ℝ := fun n t => ↑t / (600 * (↑n - 1)) + 1 / 168                                                                                                                                                   
def BIP54.Wunit : ℝ := 2 ^ 32 + 2 ^ 16                                                                                                                                                                              
def BIP54.work : ℕ → ℕ := fun g => 2 ^ 256 / (g + 1)                                                                                                                                                                   
def BIP54.nextTarget : ℕ → ℤ → ℕ := fun g s => compactRound (min powLimit (g * clampSpan s / 1209600))
def BIP54.clampSpan : ℤ → ℕ := fun s => (max (1209600 / 4) (min (4 * 1209600) s)).toNat
def BIP54.compactRound : ℕ → ℕ := fun x => setCompact (getCompact x)
def BIP54.getCompact : ℕ → ℕ × ℕ := fun x => if 2 ^ 23 ≤ mant x then (mant x / 2 ^ 8, nSize x + 1) else (mant x, nSize x)                                                                                                        
def BIP54.setCompact : ℕ × ℕ → ℕ := fun p => if p.2 ≤ 3 then p.1 / 2 ^ (8 * (3 - p.2)) else p.1 * 2 ^ (8 * (p.2 - 3))
def BIP54.mant : ℕ → ℕ := fun x => if nSize x ≤ 3 then x * 2 ^ (8 * (3 - nSize x)) else x / 2 ^ (8 * (nSize x - 3))
def BIP54.nSize : ℕ → ℕ := fun x => (x.size + 7) / 8
def BIP54.powLimit : ℕ := 65535 * 2 ^ 208
```

-------------------------

