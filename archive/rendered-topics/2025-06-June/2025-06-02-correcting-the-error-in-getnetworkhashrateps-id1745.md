# Correcting the error in getnetworkhashrateps

zawy | 2025-06-02 10:53:31 UTC | #1

Getnetworkhashrateps overestimates hashrate by N/(N-1). Using an appeal to authority to be brief, PW tweeted about this error long ago in regards to the difficulty adjustment. It would be good to fix it, being sure to disallow N=1 which has no solution. By experiment N=1 overestimates by about 7x where N=2 and N=3 are only 2x and 1.5x. 

Besides multiplying by (N-1)/N to correct it, you could go back a fixed amount of time and count blocks instead of going back a fixed number of blocks to count time so that Poisson applies (as people expect) instead of Erlang distribution.

It surprises me that people don't seem to know about it. For example [Lopp's article](https://x.com/lopp/status/1929376331579408712?t=kOHywCEIiuxxoeFQobip-w&s=19).

I assume the $10 B mining industry knows about it.

-------------------------

