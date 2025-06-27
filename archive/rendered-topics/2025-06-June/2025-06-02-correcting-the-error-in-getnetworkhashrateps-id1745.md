# Correcting the error in getnetworkhashrateps

zawy | 2025-06-02 10:53:31 UTC | #1

Getnetworkhashrateps overestimates hashrate by N/(N-1). Using an appeal to authority to be brief, PW tweeted about this error long ago in regards to the difficulty adjustment. It would be good to fix it, being sure to disallow N=1 which has no solution. By experiment N=1 overestimates by about 7x where N=2 and N=3 are only 2x and 1.5x. 

Besides multiplying by (N-1)/N to correct it, you could go back a fixed amount of time and count blocks instead of going back a fixed number of blocks to count time so that Poisson applies (as people expect) instead of Erlang distribution.

It surprises me that people don't seem to know about it. For example [Lopp's article](https://x.com/lopp/status/1929376331579408712?t=kOHywCEIiuxxoeFQobip-w&s=19).

I assume the $10 B mining industry knows about it.

-------------------------

sipa | 2025-06-27 15:03:08 UTC | #2

Is this about the difference between the expected value of an Erlang distribution, and the inverse of the expected value of an inverse Erlang distribution not being the same? That applies to the difficulty adjustment because it involves *dividing* by the hashrate, but would not apply to `getnetworkhashrateps` as it's not inverting (note that it doesn't matter that the actual difficulty logic is implemented using a `target` value which is inversely proportional to difficulty; that is just an implementation detail - difficulty is what matters as it's what scales with block duration).

If the actual hashrate is constant but unknown, then I would think that

$$
\dfrac{\sum_{b \in \mathrm{blocks}} E(\mathrm{hashes}(b))}{\sum_{b \in \mathrm{blocks}} \mathrm{time}(b)}
$$

gives an unbiased estimate for the amount of hashes per time. That is because the denominator here uses an *observed* quantity (time), rather than an *expected* quantity (hashes) - unlike in difficulty adjustment where one divides by the expected value, and thus wants the expectation of the inverse to be unbiased.

I still want to simulate this, though.

-------------------------

