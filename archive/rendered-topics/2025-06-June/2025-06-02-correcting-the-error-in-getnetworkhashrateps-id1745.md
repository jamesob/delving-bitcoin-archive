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

sipa | 2025-06-27 19:28:17 UTC | #4

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

However, the real question is whether this estimator is unbiased. To determine that, we compute the expected value of the estimation $\hat{r}$ when repeating the experiment many times (each experiment consisting of $n$ block measurements), with a *known* true hashrate $r$.

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

-------------------------

