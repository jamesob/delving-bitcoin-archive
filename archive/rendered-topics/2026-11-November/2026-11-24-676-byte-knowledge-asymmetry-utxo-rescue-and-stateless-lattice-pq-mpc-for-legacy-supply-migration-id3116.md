# 676-Byte Knowledge-Asymmetry UTXO Rescue and Stateless Lattice PQ-MPC for Legacy Supply Migration

laudzakusuma | 2026-11-24 16:38:20 UTC | #1

Recent discussions around P2MR (BIP-360) and legacy address sunsetting (draft BIP-361) highlight two practical bottlenecks for Bitcoin's post-quantum transition:

1. **Institutional Custody Statefulness:** Custodians running distributed signing servers across isolated availability zones and restoring keys from cold backups cannot safely deploy stateful hash-based PQC schemes where counter desynchronization leads to key compromise.
2. **Legacy UTXO Rescue Overhead:** Roughly 34% of Bitcoin supply sits in legacy outputs exposed to quantum public-key recovery or scheduled freezing. Existing rescue proposals require large on-chain witnesses (3,420 B for Lifeboat and 3,904 B for Dropkick).

I have published a technical paper and reference implementation addressing both constraints as part of the Adaptive Quantum Migration Protocol (AQMP v2.0).

### 1. Knowledge-Asymmetry Frozen UTXO Rescue (676 B Total)

Instead of exposing raw elliptic-curve public keys on-chain during emergency migration, the protocol derives a post-quantum lattice key $s_{KA} \in \mathbb{F}_q^{32}$ from the unhardened BIP-32 parent chain code ($c_{\text{par}}$) and pre-compromise derivation path $(P_{\text{par}}, i)$.

* **Witness Breakdown:** 32 B chain-code commitment + 64 B lattice PoK transcript + 164 B Merkle inclusion proof + 320 B destination lattice-NIZK = **580 B**.
* **Front-Running Protection:** Bound to a **96 B** Dual-Commit XOR cycle ($C = F(x) \oplus F(x \parallel m)$) gated by the consensus confirmation deadline $\Delta^* = \lceil 4/(1 - \epsilon) \rceil$.
* **Comparison:** At **676 B total**, this reduces on-chain witness size by **80.23%** relative to Lifeboat (3,420 B), **82.68%** relative to Dropkick (3,904 B), and **87.15%** relative to naive ML-DSA-65 (5,261 B).

### 2. Stateless PQ-MPC Threshold Engine

For institutional custodians, the module implements Shamir secret sharing and Lagrange interpolation over $\mathbb{F}_q^{64}$ ($q = 8,380,417$) with synthetic HMAC-SHA3-256 nonces bound to the message digest, share commitment, active quorum set, and a backup-safe epoch salt.

* **Zero Mutable State:** Wiping local node state and restoring from cold backup produces identical, deterministic partial signatures with zero nonce reuse.
* **Measured Performance:** Verification stays constant between 3.90 ms and 4.68 ms across 2-of-3, 3-of-5, and 7-of-11 quorums. When paired with P2MR roots and 4-byte KeyRegistry Plonky3 aggregation, a 3-of-5 block at $N = 250$ transactions drops from 3,945,750 B (naive ML-DSA-65) to **11,300 B** (-99.71%).

### Links & Reproducibility

* **Paper** #2 **(PDF & CSV Dataset):** https://doi.org/10.5281/zenodo.24341314
* **BIP Draft & Reference Implementation (62 passing pytest tests):** https://github.com/laudzakusuma/aqmp-protocol

I welcome technical critique on the BIP-32 chain-code knowledge-asymmetry assumptions and the Dual-Commit XOR mempool window bounds.

![fig4_stateless_mpc_scaling|690x280](upload://iNXJqTvRAH5CnVENeZJ4LaGDdTp.png)

![fig5_utxo_rescue_comparison|690x281](upload://txtvx9jButNObw5k0pYcb1h8M8y.png)

-------------------------

