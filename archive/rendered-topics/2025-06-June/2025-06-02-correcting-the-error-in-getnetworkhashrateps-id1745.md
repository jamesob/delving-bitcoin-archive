# Correcting the error in getnetworkhashrateps

zawy | 2025-06-02 10:53:31 UTC | #1

Getnetworkhashrateps overestimates hashrate by N/(N-1). Using an appeal to authority to be brief, PW tweeted about this error long ago in regards to the difficulty adjustment. It would be good to fix it, being sure to disallow N=1 which has no solution. By experiment N=1 overestimates by about 7x where N=2 and N=3 are only 2x and 1.5x. 

Besides multiplying by (N-1)/N to correct it, you could go back a fixed amount of time and count blocks instead of going back a fixed number of blocks to count time so that Poisson applies (as people expect) instead of Erlang distribution.

It surprises me that people don't seem to know about it. For example [Lopp's article](https://x.com/lopp/status/1929376331579408712?t=kOHywCEIiuxxoeFQobip-w&s=19).

I assume the $10 B mining industry knows about it.

-------------------------

sipa | 2025-06-27 18:57:42 UTC | #2

EDIT: all of my conjecture below is wrong, see the further [post](https://delvingbitcoin.org/t/correcting-the-error-in-getnetworkhashrateps/1745/4).

---

Is this about the difference between the expected value of an Erlang distribution, and the inverse of the expected value of an inverse Erlang distribution not being the same? That applies to the difficulty adjustment because it involves *dividing* by the hashrate, but would not apply to `getnetworkhashrateps` as it's not inverting (note that it doesn't matter that the actual difficulty logic is implemented using a `target` value which is inversely proportional to difficulty; that is just an implementation detail - difficulty is what matters as it's what scales with block duration).

If the actual hashrate is constant but unknown, then I would think that

$$
\dfrac{\sum_{b \in \mathrm{blocks}} E(\mathrm{hashes}(b))}{\sum_{b \in \mathrm{blocks}} \mathrm{time}(b)}
$$

gives an unbiased estimate for the amount of hashes per time. That is because the denominator here uses an *observed* quantity (time), rather than an *expected* quantity (hashes) - unlike in difficulty adjustment where one divides by the expected value, and thus wants the expectation of the inverse to be unbiased.

I still want to simulate this, though.

-------------------------

zawy | 2025-06-27 18:43:30 UTC | #3

If your simulation indicates I'm correct, my reasoning is based on an excellent quote from Wikipedia on the Erlang distribution:

> The Erlang and Poisson distributions are complementary, in that while the Poisson distribution counts the events that occur in a fixed amount of time, the Erlang distribution counts the amount of time until the occurrence of a fixed number of events.

Like the difficulty calculation, ```getnetworkhashrateps``` has a fixed number of blocks from which it measures time. So time isn't fixed as is required for Poisson to apply. Both difficulty and ```getnetworkhashrateps``` are using a fixed amount of blocks which indicates Erlang is needed, and therefore the adjustment. To make Poisson apply (no adjustment), it seems we need to have a fixed amount of time from the current time to be looking into the past to count the blocks seen. 

Another way to view it without needing to know the about Erlang distribution comes from the Poisson and another tweet you made, i.e. that if you randomly select a point in time, what's the time to the previous block plus the time to the next block from that point? From the memorylessness property, it's 1 blocktime before and 1 blocktime after, i.e. 2 block times. Let's assume the most recent block timestamp is random and is OK to use. Poisson says we have to go a fixed amount of time in the past and count blocks in that timespan, so the initial time we choose for a fixed timespan should be as random as the most recent timestamp. We expect a random initial time point to have zero blocks for 1 blocktime before and 1 blocktime after it, but we're selecting a timestamp for the time that is "on top" of a block. So that block from which we get our initial timestamp is exactly 1 block too many, so we need the adjustment (N-1)/N to subtract it out.

-------------------------

sipa | 2025-06-27 22:25:19 UTC | #4

You are correct, @zawy.

Working this out a bit more formally:
* Let $r$ be the unknown but fixed hashrate (in hashes/second). We assume it does not change during the time of our observation.
* Let $W_i$ be the amount of work in block $i$, a known constant (equal to its expected number of hashes), so:
  * $W_i = 2^{256} / (\mathrm{target}_i + 1)$
  * $W_i = 2^{48} / (2^{16}-1) \cdot \mathrm{difficulty}_i$
* Let $t_i$ be the durations of each block in seconds, which we assume are exactly observable (not true in practice, but the best we can do).

The hashes per block is a random variable that follows a geometric distribution, but can be approximated well as an exponential one due to the enormous number of trials involved:
$$
h_i \sim \mathrm{Exp}(\lambda=1/W_i)
$$
and the time per block $t_i = h_i / r$, and thus by the fact that $\lambda$ is an inverse scale parameter, 
$$
t_i \sim \mathrm{Exp}(\lambda=r/W_i)
$$
To simplify what follows, introduce $\alpha_i = t_i / W_i$, which measures how long blocks took relative to their difficulty (its unit is seconds per hash). For these, we have:
$$
\alpha_i \sim \mathrm{Exp}(\lambda=r)
$$
So all $\alpha_i$ are identically distributed. They can also be shown to be independent, even when there are difficulty adjustments in between the measured blocks. Their PDF is
$$
f(\alpha) = r \exp(-r\alpha)
$$

Our goal is estimating $r$, based on a series of $n$ observations (blocks) $\bar{\alpha}$. To start, we can build a [maximum-likelihood estimator](https://en.wikipedia.org/wiki/Maximum_likelihood_estimation) for $r$, which is the value $\hat{r}_\mathrm{MLE}$ for which the function
$$
\begin{split}
\hat{l}(r;\bar{\alpha}) \, & = \, & \sum_{i=1}^n \log f(\alpha_i) \\
& = \, & \sum_{i=1}^n \log \left( r \exp(-r\alpha_i) \right) \\
& = \, & n \log(r) + \sum_{i=1}^n \log ( \exp(-r\alpha_i)) \\
& = \, & n \log(r) - r \sum_{i=1}^n \alpha_i \\
\end{split}
$$
is maximal. The derivative in $r$ is
$$
\hat{l}'(r;\bar{\alpha}) = \frac{n}{r} - \sum_{i=1}^n \alpha_i
$$
which is $0$, and maximizes $\hat{l}$ in
$$
\hat{r}_\mathrm{MLE} = \dfrac{n}{\sum_{i=1}^n \alpha_i}
$$

If the difficulty is constant within the window, then this is equal to the current formula in `getnetworkhashps`:
$$
\hat{r}_\mathrm{RPC} = \dfrac{\sum_{i=1}^n W_i}{\sum_{i=1}^n t_i}
$$

---

So far so good. The formula being used is the maximum-likelihood estimator, at least when the difficulty does not change within the measured interval. And if the difficulty does change within it, then the starting assumption that the true but unknown hashrate is a constant throughout the interval probably doesn't hold anyway, and it may be reasonable to deviate from it.

However, the real question is whether this estimator is unbiased. To determine that, we compute the expected value of the estimation $\hat{r}_\mathrm{MLE}$ when repeating the experiment many times (each experiment consisting of $n$ block measurements), with a *known* true hashrate $r$.

$$
\mathrm{E}[\hat{r}_\mathrm{MLE}] = \mathrm{E}\left[\dfrac{n}{\sum_{i=1}^n \alpha_i}\right]
$$
Let
$$
\beta = \sum_{i=1}^n \alpha_i
$$
which is distributed as $\beta \sim \mathrm{\Gamma}(n, r)$. Then
$$
\begin{split}
\mathrm{E}[\hat{r}_\mathrm{MLE}] \, & = \, & n \cdot \mathrm{E}[\beta^{-1}] \\
& = \, & n \cdot \frac{r}{n-1} \\
& = \, & \frac{n}{n-1} r
\end{split}
$$
which is indeed a factor $\frac{n}{n-1}$ higher than what it should be. An unbiased estimator can be created by correcting for this factor, and we get
$$
\begin{split}
\hat{r} & \, = & \, \frac{n-1}{\sum_{i=1}^n \alpha_i} \\
& \, = & \, \frac{n-1}{\sum_{i=1}^n \frac{t_i}{W_i}}
\end{split}
$$

I believe it can be shown that this unbiased estimator is [sufficient](https://en.wikipedia.org/wiki/Sufficient_statistic) and [complete](https://en.wikipedia.org/wiki/Completeness_(statistics)), which would [imply](https://en.wikipedia.org/wiki/Lehmann%E2%80%93Scheff%C3%A9_theorem) it is the [minimum-variance unbiased estimator](https://en.wikipedia.org/wiki/Minimum-variance_unbiased_estimator)(MVUE).

-------------------------

zawy | 2025-06-28 07:49:41 UTC | #5

I like the idea of using a fixed time window to get hashrate instead of a fixed number of blocks because it's valid at the moment you run the query. This allows minor errors in timestamps from (usually) affecting the result. It also allows getting hashrates in the past at specific times such as a daily estimate at midnight. BTW the estimate isn't at that time or height, but at the midpoint of the heights or time range from which you're summing work.

Instead of adjusting by (N-1)/N, it might be as accurate to use the current time (of the query) in place of the most recent timestamp.  If you run the query at random, it's expected to be 1 blocktime since the most recent block, so it seems to work the same as applying (N-1)/N by making the timespan 1 block longer. This also allows me to get an estimate of hashrate at N=1 which is otherwise not possible due to the divide by zero. But by experiment and forcing timespan = 1 second if timespan < 1 second, for N=1 I get:

hashrate =~ difficulty * 2^32 / timespan / 6

BTW, an N-1 is also seen when estimating total chain work from the Nth lowest hashes ever seen. Interestingly, knowledge of difficulty and timestamps isn't needed.

chain_work =~ 2^256 * (N-1) / Nth_lowest_hash

StdDev of the error =~ 1/SQRT(N-1) for N > 10

on the condition that difficulty target was never < Nth_lowest_hash and all orphans with low hashes are included (if needed) in N, but this also counts all orphans in chainwork.

If given only the lowest hash:

chain_work =~ (2^256 -1) / 6 / lowest_hash

StdDev =~ 5000%

P.S. My fault: we should have been calling it ```getnetworkhashps```

-------------------------

zawy | 2025-06-29 14:43:12 UTC | #6

Correcting / clarifying my comment that Erlang should be used, we're estimating hashrate which has time in the denominator which means we need to use the expected value of the *inverse* of an Erlang-distributed random variable. The key point here is that the error comes from 

1/E[T] != E[1/T]

We try to use 1/E[T] when we need to use E[1/T]

So less formally (to get the desired result): 

* T = timespan
* k = N = n blocks 
* λ = 1/(mean_blocktime):

E[1/T] = λ / (k-1) for k > 2 via Grok.

* W = sipa's sum of work.
* estimated λ = k/T from Erlang λ = k / E[T].
* estimated λ = k/T from MLE for Poisson.

E[W/T] = W * E[1/T] = W / T * k / (k-1)

This is the expected, biased, hashrate measurement, so we have to multiply (k-1)/k to get W/T, the true, unbiased hashrate.

-------------------------

sipa | 2025-06-28 12:18:04 UTC | #7

By substituting $\alpha_i = t_i / W_i$ and $W_i = 2^{256} / (\mathrm{target}_i+1)$ in the formula for the corrected, unbiased, estimate:

$$
\hat{r} = (n-1) \cdot \frac{2^{256}}{\sum_{i=1}^n t_i (\mathrm{target}_i+1)}
$$

Which is quite practical if one already has 256-bit arithmetic for target computations anyway.

-------------------------

zawy | 2025-06-30 12:45:13 UTC | #8

Years ago I was testing this this time-weighted target calculation and it had substantial error if both difficulty and hashrate changed a lot for small N.  I changed hashrate and difficulty when a block was found. 

Using

* hashrate = total_work / timespan * (N-1) / N 

had much less error in that case. This latter method had a slight error (verses 0% in the time-weighted target method) if hashrate changed but difficulty didn't.

[edit: I'll make an argument for the following in the next post] 

This means sum of difficulties needs to be multiplied by (N-1)/N to get the correct amount of work in making PoW consensus decisions to prevent errors or exploits in something like selfish mining. This applies even if accurate timestamp enforcement were used to prevent selfish-mining.  The time-weighted method (multiplied by the timespan) doesn't give the correct amount of work if difficulty and hashrate are changing.   

If 2 competing tips have the same final solvetime but one is 2 blocks after their common ancestor and the other is 3 blocks, then the leading tip doesn't have 50% more hashrate & work, but averages (3-1)/(2-1) = 100% more.  This may be important in something like DAGs. The sum of descendant work perfectly orders DAGs if difficulty changes every block, but only now do I realize it may need the (N-1)/N adjustment to make some decisions and it may fix an objection Yonatan Sompolinsky pointed out to me long ago & why I couldn't get them to use sum of descendant difficulties to find the earliest blocks (instead of k nearest-neighbor). The problem is that the decision in ordering based on it doesn't "compound" quickly into a decision. An attacker could use a smallish hashrate to keep the ordering from reaching closure for as long as he attacks. With (N-1)/N, closure might come quicker. A complicating factor is that the median hashrate may be more important than the average in making decisions.

-------------------------

zawy | 2025-06-30 13:16:54 UTC | #10

The number of hashes (W) in N blocks with hashrates H<sub>i</sub> and solvetimes t<sub>i</sub> is 

W  = sum(H<sub>i</sub> * t<sub>i</sub> )

If our hashrate estimate is correct:

W_estimate = H_estimate * timespan

Experimentally & theoretically, the 2nd equation is (N-1)/N lower than the first. The mean number of hashes performed in many trials obeys the 1st. The 2nd is an underestimate of hashes actually performed, and yet we know the hashrate estimate is correct, and therefore it should be as correct as the 1st. My only guess / intuition to resolve this is that the observed timespan creates a conditional probability, so that the 2nd equation is the correct one to use. The 1st is a fact and a prediction. The 2nd is an observation where the hashrate fact is unknown but estimated. Another intuition is that the first equation is more-often-than-not lucky. The 2nd equation removes the excess luck.

To make another argument for the 2nd equation: in a network partition, we want the partition with the highest hashrate we've observed (the most dedicated mining equipment), not the "predicted" highest work.

PoW should be an observation of what has occurred, not a "prediction" of what will occur if something is repeated many times.

-------------------------

sipa | 2025-06-30 14:04:44 UTC | #11

Work (defined as the *expected* number of hashes for an individual block) is an exactly known quantity, not subject to a probability distribution. It's exactly proportional to difficulty, or inversely proportional to the target. To estimate number-of-hashes-performed, we just need to sum up the work for each block; I believe that is both the maximum likelihood estimate and unbiased. I'll try to work out the math to make that formal.

-------------------------

zawy | 2025-06-30 15:56:54 UTC | #12

If I'm wrong, why can't our estimated hashrate not simply be multiplied by the timespan to get the total work? 

Here's my "math":

* W = total hashes, assume constant target
* EE [  ] = expected value of inverse of an Erlang-distributed random variable
* T = true timespan = N / &lambda;

E [ hashrate ] = E [ W / T ] =  W * EE[1/T]

EE[1/T] = &lambda; / (N-1) = ( N / T ) / (N-1)

E [ W / T ]  = W / T * N / (N-1)

This means W / T = hashrate is as we said above: 

W / T = E [ W / T ] * (N-1) / N 

This "math" is claiming our hashrate isn't the E [hashrate]. 

Can I multiply this by T_observed to get W? T_observed is the only estimate of T that I have, and it's not in the denominator which is where I would normally a problem. So I get:

W / T * T_observed = W = Expected [ W / T ] * (N-1) / N 

My intuitive excuse for why this might be correct, despite giving the wrong W over many trials, is that an observation doesn't give us the benefit of many trials. It's only correct for a single observation. We want the work we observe, not the work we expect.

-------------------------

sipa | 2025-06-30 17:03:02 UTC | #13

I think the answer is that this would work if you want to estimate the amount of hashes performed **under the assumption that the hashrate is constant**. But there is no reason to make this assumption (it's almost certainly an approximation) if you want to estimate work. It's just a necessary assumption for estimating hashrate.

-------------------------

zawy | 2025-07-01 09:39:37 UTC | #14

It's very confusing that hashrate * hashing_time doesn't equal work. I wonder if hashrate is more important than work.  I prefer the partition that has a higher rate of lottery ticket purchases over the one with the most winning lottery tickets. The security model depends on non-repurpose-able equipment (hashrate).  

Let's say we have a strange PoW protocol where difficulty is selected by miners in the event of a partition and a partition occurs that splits them into 2 groups. One group selects a difficulty 50% harder (D=1.5) than the other (D=1).  That group finds their 2nd block at the same time as the other finds their 3rd block.  They have equal work (W=3) but statistically we know the 2nd group with lower difficulties had 33% more hashrate (Hashrate = 3/T * (3-1) /3  verses 3/T * (2-1)/ 2) and therefore should have performed 33% more hashes.

-------------------------

zawy | 2025-07-02 11:45:17 UTC | #15

I found the problem. Chain work isn't the sum of D's that starts at the block after a timestamp at height H and ends at the timestamp at H+N, but the sum of D's between those 2 timestamps which has N-1 blocks. Starting and ending on a block is a biased selection of the timespan.  

But the sum of Ds in the past N blocks is the work done up until current time (a randomly chosen point in time to perform the query, not an ending timestamp).

Hashrate at any height h in the past is

2^32 * sumD(h+1 to h+N) / timespan(h to h+N+1)

And the work in that timespan is just this times that timespan. 

I think this is slightly more accurate than the prior method which was:

2^32 * sumD(h+1 to h+N) / timespan(h to h+N) *(N-1) / N

because the correction is necessary due to using a timespan that's not exactly correct for the work.

In my competing tips example, pretend the last block has not been found, but the ending time is the same and is local time which is the expected time to be randomly looking at both chains. Then both our hashrate and work calculations agree that the tip with the easier difficulty did 33% more work.

If you want the work done in the solvetime of N blocks, and you sum up the difficulties for those N blocks, then you have to apply the (N-1)/N correction to get the ~correct amount of work in that timespan.

In deciding a leading tip, you just sum the difficulties as usual because you want the work up until current time, not the last timestamp.

-------------------------

zawy | 2025-08-06 11:23:27 UTC | #16

I deleted and rewrote this into a more coherent article [here](https://github.com/zawy12/difficulty-algorithms/issues/84).

-------------------------

zawy | 2025-08-07 00:45:21 UTC | #18

work = (2^256 - 1)  / 2nd_lowest_hash

The 2nd lowest hash is an "effective difficulty" that the lowest hash "solves".

-------------------------

sipa | 2025-08-07 21:56:29 UTC | #19

Interesting. This works with any “k'th lowest hash”, actually.

If $H_{(k)}$ is the k’th lowest hash seen (the lowest one having $k=1$), normalized to $[0 \ldots 1]$, then $\frac{k-1}{H_{(k)}}$ is an estimate for the amount of work.

If we model this as $n$ uniformly random samples being taken from $[0 \ldots 1]$ (which is what each hash attempt is), then
$$
H_{(k)} \sim  \mathrm{Beta}(k, n+k-1)
$$
For such a distribution
$$
\mathrm{E}\left[\frac{k-1}{H_{(k)}}\right] = n \\
\mathrm{Var}\left[\frac{k-1}{H_{(k)}}\right] = \frac{n(n-k+1)}{k-2}
$$
which means that using higher $k$ gives a better estimate.

However, for any small constant $k$, this variance is still $\mathcal{O}(n^2)$. Using the number of blocks is a much better estimators. Let $b$ be the number of blocks seen, and $w$ the expected amount of work per block:
$$
w = \frac{2^{256}-1}{\mathrm{target}}
$$
Then
$$
b \sim \mathrm{Poisson}\left(\frac{n}{w}\right)
$$
And we can esimate the amount of work as $bw$:
$$
\mathrm{E}\left[bw\right] = n \\
\mathrm{Var}\left[bw\right] = n
$$
Which has just $\mathcal{O}(n)$ variance.

-------------------------

