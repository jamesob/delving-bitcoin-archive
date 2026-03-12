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

conduition | 2026-03-12 17:57:03 UTC | #4

Nice finds Jesse. My first major issue is that some of these articles are paywalled so I can't read them in totality. For those that I can read, i have some critiques.

*“[A Secure Hierarchical Deterministic Wallet with Stealth Address from Lattices](https://www.sciencedirect.com/science/article/pii/S0304397524002895?via%3Dihub)”*

- This paper is paywalled except for the introduction section. My opinions are entirely based on the introduction section.
- They state: _"note that even the above effort we made, we recognize that the size of signature and key in our scheme is far from practical for the blockchain system, even in the cryptocurrency. Constructing an efficient post-quantum one is the future work."_ I don't know how big the actual signatures are, because paywall, but i'm guessing they ran into the same problem [these guys](https://eprint.iacr.org/2026/380) did - namely that adding the necessary algebraic structure necessitates including the full matrix $A$ in the pubkey, rather than a pseudorandom seed as ML-DSA does.
- They state: _"In order to realize the unlinkability, that, the extended public key related information cannot be obtained from the derived public verification key 
, we just need the payee and the payer to know and have the same secret information. Therefore, we need to use a Public Key Encryption (PKE) scheme to encrypt the secret information."_ This will add a lot of complexity and additional security requirements beyond standard models. 


https://link.springer.com/chapter/10.1007/978-3-030-51938-4_14

- This paper doesn't give concrete parameters and so we can't say how big the keys and signatures would actually be. Based on their table in page 3, $|\text{apk}| = |\text{msig}| = \tilde{\mathcal{O}}(n^2)$, so signatures plus pubkeys would be $2 n^2$ units. I assume they mean bits, but the paper is vague. By the way if you're wondering what $\tilde{\mathcal{O}}$ means: https://www.johndcook.com/blog/2019/01/17/big-o-tilde-notation/
- They define $n = \mathcal{O}(\lambda)$ where $\lambda$ is the security parameter, e.g. 128, 192, 256. So without concrete parameters or units i'm left to assume $n^2 \approx 128^2 \approx 16384$, thus key+sig size would be at least $2n^2/8 \approx 4096$ bytes, only a few hundred bytes larger than ML-DSA.
- In page 4 they say: _"In our MS construction, a trusted third party
generates the public parameter set Y that contains a public matrix A."_ They don't elaborate on why this trusted third party is needed or what happens if they collude or misbehave.
- The follow-up paper is paywalled so I can't seek clarity from it.

https://eprint.iacr.org/2025/872

- See page 5 for sizes. Signature + pubkey for this scheme (5.2kb) is around 1400 bytes larger than ML-DSA (3.7kb). They bragged a lot on how their signatures are comparable to ML-DSA, but their pubkeys are twice as big.
- Other than the pubkey size, this scheme looks very clever.

-------------------------

