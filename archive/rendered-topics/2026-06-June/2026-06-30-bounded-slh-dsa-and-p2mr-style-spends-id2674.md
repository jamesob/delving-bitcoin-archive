# Bounded SLH-DSA and P2MR-style spends

kiwidream | 2026-06-30 20:46:36 UTC | #1

Hey all,

I've been working on a functional bounded30 SLH-DSA/SHA2 implementation in a separate Bitcoin-derived UTXO codebase. This is not Bitcoin Core, but I'm linking the repo so the parameter choices and assumptions are easy to inspect: [https://github.com/Qbit-Org/qbit-libbitcoinpqc](https://github.com/Qbit-Org/qbit-libbitcoinpqc)

The current profile has 32-byte public keys and 3680-byte signatures. That comes from a `2^30` per-key signing bound and WOTS+C/FORS+C compression.

In the chain I'm working on, some of the wallet and policy issues can be handled by designing around this profile from the start. Bitcoin is a different problem because it has existing wallet behavior, descriptors, relay policy, fee bumping norms, hardware wallet assumptions, and migration constraints.

So the narrow question I'm trying to ask is: if a bounded hash-based signature profile like this were used as a P2MR/BIP-360-style spend path, what do you think is the first Bitcoin-specific assumption that would fail?

It's reasonable to assume is that the cryptography is only one part of the problem. I'm especially interested in criticism around this implementation itself, or consequences around validation cost, wallet semantics, and whether the `2^30` bound materially changes the discussion for Bitcoin compared to one-time or very-low-use hash-based schemes.

-------------------------

