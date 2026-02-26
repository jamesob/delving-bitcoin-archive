# Binohash: Transaction Introspection Without Softforks

RobinLinus | 2026-02-25 16:55:26 UTC | #1

**Abstract.** We present Binohash, a collision-resistant hash function for Bitcoin Script that enables limited transaction introspection without consensus changes. By exploiting the FindAndDelete quirk in legacy OP_CHECKMULTISIG combined with proof-of-work signature grinding, Binohash creates a transaction digest directly readable in Script. This enables covenant-like functionality for trustless chain introspection for protocols like BitVM bridges.

https://robinlinus.com/binohash.pdf

-------------------------

garlonicon | 2026-02-26 09:37:53 UTC | #2

`OP_SIZE <60 - n > OP_EQUALVERIFY < pubkey > OP_CHECKSIGVERIFY`

Instead of `OP_EQUALVERIFY`, `OP_LESSTHAN OP_VERIFY` can be used. Then, smaller signatures are also accepted. Also, `OP_CHECKSEQUENCEVERIFY` or `OP_CHECKLOCKTIMEVERIFY` can provide an incentive, to produce the smallest signature in a given time, and then, broadcast a valid transaction earlier, than other competing miners.

-------------------------

