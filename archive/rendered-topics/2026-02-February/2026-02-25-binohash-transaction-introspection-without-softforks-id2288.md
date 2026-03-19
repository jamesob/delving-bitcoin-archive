# Binohash: Transaction Introspection Without Softforks

RobinLinus | 2026-02-25 16:55:26 UTC | #1

**Abstract.** We present Binohash, a collision-resistant hash function for Bitcoin Script that enables limited transaction introspection without consensus changes. By exploiting the FindAndDelete quirk in legacy OP_CHECKMULTISIG combined with proof-of-work signature grinding, Binohash creates a transaction digest directly readable in Script. This enables covenant-like functionality for trustless chain introspection for protocols like BitVM bridges.

https://robinlinus.com/binohash.pdf

-------------------------

garlonicon | 2026-02-26 09:37:53 UTC | #2

`OP_SIZE <60 - n > OP_EQUALVERIFY < pubkey > OP_CHECKSIGVERIFY`

Instead of `OP_EQUALVERIFY`, `OP_LESSTHAN OP_VERIFY` can be used. Then, smaller signatures are also accepted. Also, `OP_CHECKSEQUENCEVERIFY` or `OP_CHECKLOCKTIMEVERIFY` can provide an incentive, to produce the smallest signature in a given time, and then, broadcast a valid transaction earlier, than other competing miners.

-------------------------

AaronZhang | 2026-03-18 17:35:11 UTC | #3

Ran empirical parameter sweeps on the (n,t) subset grinding mechanism (#4.5) on Bitcoin Core regtest. FindAndDelete and legacy sighash were cross-checked against python-bitcoinlib. 50 trials per group:

* B (10,5), bits=10, C=252: success_rate=26%, avg_attempts_all=223
* D (12,6), bits=10, C=924: success_rate=44%, avg_attempts_all=688
* E (14,7), bits=10, C=3432: success_rate=98%, avg_attempts_all=921
* F (14,7), bits=14, C=3432: success_rate=14%, avg_attempts_all=3253
* G (16,8), bits=14, C=12870: success_rate=56%, avg_attempts_all=8731

\> verify(success)=100% across all groups.

Results support the E≈W2 tradeoff empirically: at bits=10, E≈3432 reaches \~98%; raising bits to 14 at the same space drops to \~14%; increasing space by 3.75x (G) recovers to \~56%.

Code and raw logs: github.com/aaron-recompile/binohash-experiments

-------------------------

AaronZhang | 2026-03-19 17:06:34 UTC | #4

From these experiments, another observation:

Binohash doesn’t give introspection in the usual sense — the script can’t read tx data.

But it seems to get close:

```plaintext
mutation → commitment → verification
```

Instead of read-access to tx fields, we get sighash mutation via subset selection, committed through CHECKSIG.

This feels like a form of pseudo-introspection.

Is that a useful way to think about it?

-------------------------

