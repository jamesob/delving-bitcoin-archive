# Binohash: Transaction Introspection Without Softforks

RobinLinus | 2026-02-25 16:55:26 UTC | #1

**Abstract.** We present Binohash, a collision-resistant hash function for Bitcoin Script that enables limited transaction introspection without consensus changes. By exploiting the FindAndDelete quirk in legacy OP_CHECKMULTISIG combined with proof-of-work signature grinding, Binohash creates a transaction digest directly readable in Script. This enables covenant-like functionality for trustless chain introspection for protocols like BitVM bridges.

https://robinlinus.com/binohash.pdf

-------------------------

