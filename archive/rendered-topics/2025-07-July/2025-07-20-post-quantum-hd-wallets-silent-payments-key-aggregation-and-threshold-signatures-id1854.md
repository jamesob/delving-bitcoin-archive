# Post-Quantum HD-Wallets, Silent Payments, Key Aggregation, and Threshold Signatures

jesseposner | 2025-07-20 18:33:40 UTC | #1

The algebraic structure of lattices suggests that HD wallets, silent‑payment–style addresses, key aggregation, and threshold signatures can be built on post‑quantum primitives and could, in principle, align with the current draft BIP‑360 (P2QRH) and ML‑DSA (Dilithium, FIPS 204).

* **HD wallets & stealth addresses**  
  *“[A Secure Hierarchical Deterministic Wallet with Stealth Address from Lattices](https://www.sciencedirect.com/science/article/pii/S0304397524002895?via%3Dihub)”* proposes a deterministic tree based on basis‑delegation and a static public identifier from which any sender can derive a one‑time address.

* **Key aggregation (MuSig‑like)**  
  *“[Round‑Optimal Secure Multisignature Schemes from Lattices with Public Key Aggregation and Signature Compression](https://link.springer.com/chapter/10.1007/978-3-030-51938-4_14)”* shows that multiple lattice public keys can be aggregated into a single pk and signature. A more efficient follow‑up is *“[Efficient Multi‑Signature Scheme Using Lattice](https://academic.oup.com/comjnl/article-abstract/65/9/2421/6289877?redirectedFrom=fulltext)”*.

* **Threshold signatures (FROST‑like)**  
  *“[Finally! A Compact Lattice‑Based Threshold Signature](https://eprint.iacr.org/2025/872)”* gives a t‑of‑n protocol (t ≤ 8) whose final signature is the same size as a single Dilithium sig.

Taken together, these papers indicate there is nothing inherently blocking lattice‑based post‑quantum replacements for BIP‑32, BIP‑352 silent payments, MuSig, or FROST.

-------------------------

sanket1729 | 2025-07-22 18:55:36 UTC | #2

Awesome. It is great to know that there is nothing conceptually blocking these technologies in PQ world. 

Have you had time to investigate each scheme’s pros and cons compared to today’s ECDSA options? Maybe some of them might not be practical or some of them might be even more attractive than ECDSA alternatives. For example, it seems like Musig/Frost equivalent is only 1 round signing instead of 2 rounds which is big deal

-------------------------

jesseposner | 2025-07-28 19:05:27 UTC | #3

That's a great point! I will follow-up after I dig in deeper to understand all the tradeoffs.

-------------------------

