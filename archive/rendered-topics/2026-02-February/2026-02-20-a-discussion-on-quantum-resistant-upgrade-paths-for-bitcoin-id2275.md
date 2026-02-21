# A Discussion on Quantum-Resistant Upgrade Paths for Bitcoin

Noe | 2026-02-20 23:00:59 UTC | #1

\# Quantum-Resistant Upgrade Path for Bitcoin Core

\## A Technical, Cryptographic, and Economic Analysis

---

### ⸻ Abstract
Bitcoin’s long-term security depends on cryptographic primitives that may be vulnerable to future large-scale quantum computers. While SHA-256 maintains substantial security margins against Grover’s algorithm, ECDSA over secp256k1 is theoretically breakable via Shor’s algorithm once sufficiently capable quantum hardware exists.

This paper analyzes feasible upgrade paths for integrating post-quantum digital signature schemes into Bitcoin Core while preserving network determinism, economic continuity, and decentralized governance.

---

### ⸻ 1. Introduction
Cryptographic transitions at protocol scale require multi-year planning. Even if large quantum computers are decades away, protocol-level preparation must begin well in advance due to ecosystem coordination complexity.

The goal of this research is to outline technically and economically viable quantum-resistance strategies for Bitcoin Core.

### ⸻ 2. Cryptographic Foundations
#### 2.1 ECDSA and the Discrete Logarithm Problem
ECDSA security relies on the hardness of the elliptic curve discrete logarithm problem (ECDLP). Shor (1994) demonstrated that quantum computers can solve discrete logarithms in polynomial time.

If a sufficiently large quantum computer exists:
\* Public key → private key derivation becomes feasible
\* Exposed public keys become vulnerable
\* Unconfirmed transactions may be stealable

#### 2.2 Hash Functions Under Quantum Attack
Grover’s algorithm reduces brute-force complexity quadratically. 256-bit hash → \~128-bit effective security. This remains acceptable under current cryptographic standards. 
Conclusion: The signature layer is the primary quantum concern.

### ⸻ 3. Threat Model
1. A quantum adversary capable of executing Shor’s algorithm.
2. Access to public keys exposed in the blockchain.
3. Capability to perform key recovery within the transaction confirmation window.

### ⸻ 4. Design Constraints
Any quantum upgrade must satisfy:
\* Deterministic consensus validation.
\* Backward compatibility (if feasible).
\* Minimal network instability and UTXO set continuity.
\* Bounded resource growth (avoiding validation bottlenecks).

### ⸻ 5. Strategy I — Hybrid Signature Validation
Model: Each transaction input contains both a classical ECDSA signature and a post-quantum signature (e.g., CRYSTALS-Dilithium).
Validation Rule: Verify_ECDSA = true AND Verify_PQ = true.
Technical Implications: Requires modification in the signature verification pipeline and witness serialization. Signature size impact: \~35x increase in input size.

### ⸻ 6. Strategy II — Opcode-Based Quantum Address
Model: Introduce OP_CHECKSIG_PQ. Legacy addresses remain valid, but new PQ public key serialization is introduced via soft fork.
Advantages: Clean separation and voluntary migration.

### ⸻ 7. Strategy III — Full Hard Fork Replacement
Remove ECDSA entirely. This carries high coordination costs and risks of chain splits.

### ⸻ 8. Economic and Game-Theoretic Analysis
Miners optimize for fee revenue and propagation speed. Larger signatures increase block weight and orphan probability. Coordination equilibrium depends on hash power alignment and exchange adoption.

### ⸻ 9. Migration Framework
1. Phase 1: Optional PQ support.
2. Phase 2: Hybrid enforcement for new outputs.
3. Phase 3: Evaluate ECDSA deprecation.

### ⸻ 10. Open Research Questions
1. Can PQ signatures be aggregated?
2. Can block compression offset signature growth?
3. How to protect dormant UTXOs?

---

### ⸻ 11. Support this Research
This research is independent and self-funded. If you find this analysis valuable for the future of Bitcoin, you can support my work here:

BTC Address: bc1q8yfhgkhrhgggjzxmx2f6ga67v5lvqv2j70yjnj

---
\*This document is a research proposal only and does not alter the Bitcoin protocol.\*

-------------------------

