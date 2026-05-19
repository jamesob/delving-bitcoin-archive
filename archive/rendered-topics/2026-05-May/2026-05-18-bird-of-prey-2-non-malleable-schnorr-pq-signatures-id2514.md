# Bird of Prey 2: non-malleable Schnorr + PQ signatures

sipa | 2026-05-18 22:02:22 UTC | #1

This [paper](https://link.springer.com/chapter/10.1007/978-3-032-25317-0_8) just presented at EuroCrypt 2026 may be interesting to some. This is isn't a proposal, or even suggestion, for use in Bitcoin. However, given the amount of interest in the discussion of PQC signature schemes, I thought it would be cool for people to be aware.

The question explored is how to construct hybrid SUF-CMA (called "non-malleable signatures" in Bitcoinese) signature schemes, if you have two individually SUF-CMA signature schemes. Specifically, one Schnorr-like one, and another possibly PQC one.

The naive solution is just to sign with both schemes once, and concatenate the signatures. That is unforgeable if at least one of the schemes remains unbroken, but not non-malleable. This is easy to see: if one of the schemes is broken, an attacker can learn the corresponding private key, and replace the signature of that scheme with a new one. They cannot sign a message of their choice as long as the other scheme remains secure, but they succeeded in malleating.

The `BoP-2` scheme in the paper offers a solution specifically for combining (a) a signature scheme based on a Fiat-Shamir transformed identification scheme like Schnorr and (b) an arbitrary other scheme that is used as a black box. If I understand it correctly (I only saw the presentation, and skimmed the paper), it would translate to something like this if applied to BIP340 + some arbitrary PQC scheme:

To sign message $m$ using private key $(d_\text{schnorr}, d_\text{pqc})$ and public key $(P_\text{schnorr}, P_\text{pqc})$:
* Generate random nonce $k$ (normally like in BIP340), and $R = k \cdot G$.
* Compute $m' = \text{H}(R || P_\text{schnorr} || P_\text{pqc} || m)$.
* Create signature $\sigma_\text{pqc}$ with key $d_\text{pqc}$ on message $m'$.
* Let $s = k + d_\text{schnorr} \times \text{H}(\sigma_\text{pqc})$.
* Return $\sigma = (s, \sigma_\text{pqc})$. 

To verify signature $(s, \sigma_\text{pqc})$ on message $m$ against public key $(P_\text{schnorr}, P_\text{pqc})$:
* Recover $R = s \cdot G - \text{H}(\sigma_\text{pqc})\cdot P_\text{schnorr}$.
* Recompute $m' = \text{H}(R || P_\text{schnorr} || P_\text{pqc} || m)$.
* Verify signature $\sigma_\text{pqc}$ with key $P_\text{pqc}$ on message $m'$.

Note that through the need to recover $R$ from $s$, this is not batch-verifiable, but saves 32 bytes. I believe it's easy to construct an alternative that just puts $R$ in the signature. Batch-verifiability may also just be an unmaintainable property in a PQC setting. For security, this also relies on $\sigma_\text{pqc}$ being an informational-theoretical commitment to $m'$ (and thus not reliant on the computational hardness of the signature scheme).

-------------------------

