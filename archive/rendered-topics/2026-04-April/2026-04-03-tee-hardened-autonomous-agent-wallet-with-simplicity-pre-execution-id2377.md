# TEE-Hardened Autonomous Agent Wallet with Simplicity Pre-Execution

Laz1m0v | 2026-04-03 23:35:00 UTC | #1

Hello everyone,

We are sharing the architecture and empirical results of **PRECOP** (Predictive Covenant Oracle Protocol), an execution environment designed to explore how autonomous agents can securely manage Bitcoin L1 capital under formally verified spending constraints.

Rather than proposing a new consensus protocol, we have built a **TEE-hardened autonomous wallet architecture** that embeds a Simplicity Bit Machine to simulate covenant enforcement locally before generating any BIP-341 Schnorr signature. This makes PRECOP a **client-side-validated overlay** comparable in trust posture to RGB or client-validated protocols not a consensus-enforced system. We want to be precise about this from the outset.

The architecture is designed with a natural upgrade path: if Simplicity is ever activated via soft fork, the identical compiled programs (same CMR hashes) transition to full consensus enforcement without modification. We do not assume or advocate for any specific soft fork timeline.

The complete yellowpaper (v0.0.2) and codebase are available at:
**Repository:** https://github.com/BitcoinWorldTrustFoundation/precop

---

## 1. Motivation: The Autonomous Agent Problem

The immediate use case is not DeFi in the traditional sense. It is the problem of **AI-managed capital**: how does an autonomous agent (e.g., an LLM) execute Bitcoin transactions without the ability to violate spending constraints, even if the agent is compromised, hallucinates, or is maliciously prompted?

PRECOP addresses this by interposing a **Guardian architecture** between the agent and the Bitcoin network:

```
Untrusted Agent Intent  →  TEE Guardian (AWS Nitro)  →  Simplicity Bit Machine  →  BIP-341 Signature

```

1. The agent constructs a payment intent and generates a PSBT.

2. Inside the Enclave, the PSBT is introspected: the Enclave overrides the `witness_utxo` context with its own internal `scriptPubKey` to prevent malicious PSBT blinding.

3. The Enclave runs the **production Simplicity engine** (`simplicity-unchained`, Blockstream, tag v1.9.2) the same Rust implementation, the same jet definitions, the same Bit Machine semantics that a future consensus-level evaluator would execute. This is not a simplified reimplementation or a simulation; it is the canonical Simplicity evaluator linked as a library crate inside the Enclave binary. The `.simf` contracts are compiled via `simc` into Commitment Merkle Roots (CMRs) that are bit-identical to what a consensus node would verify.

4. Only if the Simplicity contract evaluates to `TRUE` does the Enclave generate a Schnorr signature.

The private key never leaves the Enclave memory. The agent cannot sign independently.

![agent_btc_wallet-ezgif.com-video-to-gif-converter|690x320](upload://c8PH4JtS7OJgysHoky4dpkk7VVS.gif)

### 1.1 The Trust Anchor (Addressing the TEE Objection)

We are fully aware that this is client-side covenant execution. The global Bitcoin network does not enforce the Simplicity logic today. If an attacker forks the agent binary, disables the TEE checks, and obtains the key material (which requires compromising AWS Nitro’s physical security or KMS), they could sign transactions that violate the covenant rules and the network would accept them.

The covenant currently exists as a **local defense mechanism** protecting the agent’s UTXOs from its own failures and from external compromise of the host. It is not a consensus-level guarantee. The mitigation for TEE failure is a `OP_CHECKSEQUENCEVERIFY` timelock fallback (144 blocks ≈ 24h) that degrades control to a multisig quorum of human administrators.

### 1.2 Relationship to DLCs

A natural question: why not Discreet Log Contracts? DLCs elegantly solve oracle-to-contract binding for bilateral bets with discrete outcomes. PRECOP targets a different design space: **continuous-state machines** CDPs with variable collateral ratios, AMM pools with evolving reserves, stability fees accruing per-block. In a DLC, each possible state requires a pre-signed Contract Execution Transaction. A CDP with 64-bit collateral and 64-bit debt has a combinatorial state space that makes CET enumeration impractical. PRECOP solves this by encoding state algebraically in a single Taproot tweak and validating transitions via Simplicity introspection, rather than pre-signing discrete outcomes.

The tradeoff is explicit: PRECOP requires a Simplicity evaluator (currently TEE-local, potentially consensus-level), while DLCs work with Bitcoin Script today.

---

## 2. The Upgrade Path: From Soft to Hard Covenants

This is the architectural property we consider most interesting for the community.

By generating valid **5-Leaf MAST structures** where each leaf is a compiled Simplicity program (using `simplicity-unchained`, Blockstream, tag v1.9.2), we create a dual-regime execution model:

| Regime | Enforcement | Trust Assumption |
|----|----|----|
| **Today: Soft Covenant** | TEE evaluates Simplicity locally before signing | AWS Nitro physical security + KMS |
| **Future: Hard Covenant** | Bitcoin consensus evaluates Simplicity at spend time | Bitcoin’s consensus rules (no additional trust) |

The transition requires zero code changes. This is not an approximation: the Enclave links the same `simplicity-unchained` crate that would be embedded in a consensus node. The `.simf` sources compile to the same CMR hashes via `simc`. The MAST topology is identical. The only variable is *who* evaluates the Simplicity program today, the Enclave; after activation, every validating node. Any behavioral divergence between these two regimes would constitute a bug in `simplicity-unchained` itself, not in PRECOP.

This makes the current architecture a **live testing ground** for Simplicity-based covenants under real adversarial conditions (mainnet fee markets, mempool congestion, reorgs), generating empirical data that may be useful for the Simplicity activation discussion.

---

## 3. Template-Enforced UTXO State Machines (TUSM)

To track continuous state without bloating the UTXO set or pre-signing discrete CETs, we encode a 31-byte canonical state vector directly into the Taproot public key tweak (**Astrolabe Pattern**):

```
Offset  Size     Field             Encoding
0x00    1 byte   version (0x05)    —
0x01    8 bytes  vault_id          Big-Endian
0x09    1 byte   phase             {3=MINT, 4=REPAY, 5=LIQUIDATE, 7=FREEZE}
0x0A    8 bytes  collateral (sats) Big-Endian
0x12    8 bytes  debt (10^8 scale) Big-Endian
0x1A    5 bytes  reserved          Zero-padded

```

The state is hashed and used as a scalar tweak:

```
H_S    = SHA256(encode(S))
tag    = SHA256("PRECOP/State")
tweak  = tagged_hash(tag, P || H_S)
Q_t    = P + tweak · G

```

**Why include P in the tweak?** Following BIP-341 convention: two different internal keys P₁ ≠ P₂ with the same state S produce distinct output keys Q₁ ≠ Q₂, preventing cross-vault key reuse attacks. The domain tag `"PRECOP/State"` isolates PRECOP tweaks from BIP-341 taptweak, BIP-86 keypath, and other tagged-hash protocols.

**Collision resistance:** Two distinct states S₁ ≠ S₂ producing Q₁ = Q₂ under the same key P requires `SHA256(encode(S₁)) = SHA256(encode(S₂))` a SHA-256 collision.

**Why this is not OP_RETURN.** Traditional approaches store protocol state in OP_RETURN payloads data that any node or indexer is free to ignore, that bloats the UTXO set with unspendable outputs, and that has no cryptographic binding to the spending conditions. The Astrolabe Pattern is fundamentally different: the state is physically integrated into the Taproot address itself. The address *is* the state. Every state transition produces a new output key Q\_{t+1}, which means the UTXO rotates to a fresh address you cannot replay an old state against a new address, because the tweak will not match. The covenant, the state, and the destination are fused into a single cryptographic object.

A PRECOP vault is indistinguishable from a standard P2TR single-key transfer until a state transition reveals the script path. The contract’s complexity never bloats the global UTXO set.

---

## 4. Simplicity Introspection: How the Covenant Sees the Transaction

The Astrolabe Pattern (Section 3) encodes *what state a UTXO represents*. But encoding state is only half the problem. The covenant must also **verify the spending transaction itself**  confirming that the correct recipients receive the correct amounts at the correct output indices. This is **transaction introspection**, and it is the core capability that makes TUSM enforcement possible.

In traditional Bitcoin Script, a spending script can verify signatures and timelocks, but it cannot examine the outputs of its own transaction. It is blind to where the funds are going. Simplicity removes this limitation. The Bit Machine provides native jets that read the spending transaction’s outputs at evaluation time:

* `jet::output_script_pubkey_hash(n)` returns the SHA-256 hash of the `scriptPubKey` at output index `n`.

* `jet::output_amount(n)` returns the satoshi value at output index `n`.

These two primitives give the covenant **direct vision** into the transaction that is attempting to spend the UTXO. Combined with `jet::eq_256` (constant-time equality) and `jet::ge_128` (128-bit comparison), the covenant can enforce arbitrary routing and value constraints in a single evaluation pass.

**Concrete example , a CDP Repay transaction.** When a borrower repays debt to close a vault, the covenant simultaneously verifies:

```
jet::output_script_pubkey_hash(0) == BORROWER_SPK_HASH   // collateral returns to borrower
jet::output_amount(0)             >= collateral_sats      // full collateral released
jet::output_script_pubkey_hash(2) == TREASURY_SPK_HASH    // stability fee goes to treasury
jet::output_amount(2)             >= computed_fee          // fee amount is correct (128-bit cross-mult)
jet::output_script_pubkey_hash(3) == EXPECTED_JSON_HASH   // OP_RETURN metadata matches intent

```

All five checks execute within a single Simplicity program evaluation. If any condition fails, the Bit Machine returns `FALSE` and the transaction is rejected (by the TEE today; by consensus after activation).

**Why this matters.** Without introspection, enforcing “output 2 must pay the treasury exactly 3%” would require pre-signing every possible fee amount  the same combinatorial explosion that makes DLC-style CET enumeration impractical for continuous-state machines. Simplicity introspection collapses this into a single algebraic verification: `jet::ge_128(jet::multiply_64(treasury_out, 100), jet::multiply_64(total, 3))`.

The covenant computes the constraint at spend time rather than pre-committing to every possible outcome.

This is the mechanism that connects the Astrolabe state encoding (Section 3) to the MAST covenant topology (Section 5): the state tells the covenant *what should happen*, and introspection lets the covenant *verify that it did*.

**The covenant as the sole arbiter.** The combination of Astrolabe state encoding, Simplicity introspection, and curried SPK hashes produces an important architectural property: the Simplicity program inside the Taproot leaf is the sole and final authority over whether a state transition is valid. Neither the oracle, nor the user’s signature, nor any off-chain indexer can override a covenant that evaluates to `FALSE`. The user authenticates the transaction (Schnorr signature), the oracle provides exogenous data (price attestation), but only the covenant *enforces* ; it verifies the math, checks the routing, and either permits or rejects the spend. Under Soft Covenant enforcement (TEE), this authority is local. Under Hard Covenant enforcement (consensus activation), it becomes global. The logic is identical in both regimes.

---

## 5. Covenant Architecture: 5-Leaf MAST

The validation logic is decomposed into a 5-leaf MAST. Each leaf is a compiled Simplicity program:

| Leaf | Contract | Role |
|----|----|----|
| 0 | `brc20_cdp` | CDP Vault: MINT / REPAY / LIQUIDATE |
| 1 | `universal_dex` | Ask Engine: seller locks asset, BTC fills |
| 2 | `bid_dex` | Bid Mirror: buyer locks BTC, asset fills |
| 3 | `brc20_std` | Sentinel: liveness guardrail, FREEZE |
| 4 | `stake_enf` | Slasher: oracle equivocation burn (0x0C) |

The complete jet surface area uses seven Simplicity jets:

* `jet::output_script_pubkey_hash(n)` — routing enforcement (borrower, seller, treasury)

* `jet::output_amount(n)` — dust invariant (== 546) and fee ratio verification

* `jet::multiply_64(a, b)` — 128-bit cross-multiplication (no division anywhere)

* `jet::eq_256(a, b)` — constant-time 256-bit SPK hash assertion

* `jet::ge_128(a, b)` — unsigned 128-bit comparison for ratio enforcement

* `jet::bip340_verify(pk, msg, sig)` — bootstrap oracle attestation

* `jet::sha256_ctx_8_add_8(ctx, d)` — anti-collision oracle binding (bitwise concatenation)

### 5.1 Division-Free Ratio Enforcement

All financial invariants are enforced via 128-bit cross-multiplication. The minting threshold (150%) becomes:

```
C · P ≥ D · 150 · 10^8

```

evaluated with `jet::multiply_64` producing a 128-bit result verified by `jet::ge_128`. No integer division, no truncation error, no division-by-zero. Liquidation triggers at `C · P < D · 120 · 10^8`.

The stability fee (2.5% APR) is discretized identically:

```
stability_fee_paid · 52,560,000 ≥ D · 25 · Δblocks

```

where `52,560,000 = 144 × 365 × 1000` is the Block Year Factor.

### 5.2 Atomic Swap Symmetry

The DEX implements perfect bid/ask symmetry across two MAST leaves with identical output topology:

| **Output** | **Content** | **Verification Jet** |
|----|----|----|
| **0** | **OP_RETURN metadata** | `jet::output_script_pubkey_hash(0)` |
| **1** | **Asset Receiver (Buyer)** (546 sats) | `jet::output_script_pubkey_hash(1)` |
| **2** | **BTC Receiver (Seller)** (97%) | `jet::output_amount(2)` |
| **3** | **Treasury fee** (3%) | `jet::output_script_pubkey_hash(3)` |

Both engines enforce the 97/3 split via identical 128-bit cross-multiplication. The treasury SPK hash is curried into the TapLeaf at MAST construction time output hijacking requires inverting tagged SHA-256.

### 5.3 Dynamic Indexing (Multi-Standard Support)

A single `asset_protocol` witness byte routes verification across BRC-20 (JSON envelope), Runes (binary edict), and Ordinals (envelope-less), eliminating per-standard contract deployments.

### 5.4 Sovereign Routing and Output Hijacking Prevention

Both recipient hashes (borrower, treasury) are curried into the 72-byte TapLeaf script at MAST construction time:

```
OP_2DROP ×4       (discard 8 witness padding items)
PUSH32 <borrower_spk_hash>
OP_EQUALVERIFY
PUSH32 <treasury_spk_hash>
OP_EQUAL

```

These hashes are committed into the Merkle root M_t and therefore into the Taproot output key Q_t. Substituting a different destination requires forging a TapLeaf with a different Merkle path inverting tagged SHA-256. Funds are address-locked before any Simplicity program evaluates.

---

## 6. Oracle Model: Separating Price Discovery from Attestation Security

We distinguish two orthogonal problems:

1. **Price discovery** (source of truth): Where does the price come from?

2. **Attestation security** (cost of fraud): How do we prevent the oracle from lying cheaply?

### 6.1 Price Discovery: UTXOracle (Heuristic)

The price signal derives from UTXOracle v9.1 a statistical extraction method that analyzes UTXO value clustering around round fiat denominations. **We acknowledge this is heuristic, not consensus-grade.** It is subject to noise, manipulation (via sustained artificial transaction volume), and latency.

The Openclaw extension adds mitigations:

* **Entropy Guard:** Minimum 10,000 transactions in the scanning window before any price is published. Manipulating the signal requires sustained L1-level interference — expensive but not impossible.

* **Asset Oracle:** Heuristic fingerprinting of marketplace witnesses (UniSat, Magic Eden) for BRC-20/Runes VWAP zero API reliance, but dependent on marketplace volume.

**This is the weakest layer of the protocol, and we are explicit about it.** It provides a “good enough” signal for testing the state machine. We are actively investigating alternatives (commitment schemes, median-of-medians across multiple extraction windows).

### 6.2 Attestation Security: Binohash PoW Sealing

Binohash addresses the cost of fraud:

```
SHA256(domain || price_u64 || height_u64 || nonce) ≤ Target

```

**PoW does not prove the price is correct.** It proves that someone expended at least O(2^W) SHA-256 evaluations to commit to a specific (price, height) pair. This is a cost floor, not a truth guarantee.

The security argument is economic:

```
Attack is irrational when:
  Energy_Cost(2^W / R) > MEV(P_deviated) - S_btc

```

where `S_btc` is the oracle’s slashable BTC stake (enforced by `stake_enf`, leaf 4, opcode 0x0C).

The oracle message uses bitwise concatenation via `sha256_ctx_8_add_8`, not arithmetic addition  injective encoding eliminates temporal replay.

### 6.3 Two Security Epochs

| Phase | Mechanism | Trust Assumption |
|----|----|----|
| **Bootstrap (current)** | BIP-340 Schnorr attestation + BTC stake | Oracle identity + economic deterrent |
| **Thermodynamic (target)** | Binohash PoW | Energy expenditure (physics-bound) |

The price discovery layer (UTXOracle) remains heuristic in both phases. We consider this an open research problem.

### 6.4 Difficulty Calibration

| Scenario | W (bits) | R (H/s) | E\[time\] | Context |
|----|----|----|----|----|
| Mutinynet dev oracle | 20 | 10^10 | ≈ 105 μs | Rapid testing |
| Bootstrap mainnet | 42 | 10^10 | ≈ 7.3 min | 8× RTX 5090 cluster |
| High-value CDP | 60 | 10^10 | ≈ 1,332 days | GPU infeasible |
| ASIC-secured vault | 60 | 10^15 | ≈ 13.3 min | Nation-state level |

These numbers depend on hardware assumptions and will require periodic recalibration.

---

## 7. Parallelization: Creator Dust Pools

To avoid the “Hot UTXO” bottleneck, genesis transactions create N identical UTXOs locked under Q_0, each provisioned with exactly 330 sats (TOKEN_DUST_SATS). This enables parallel state mutations within a single block epoch. Empirical testing on Mutinynet demonstrated 20× trade parallelization.

Recursive state settlement (δ^k) aggregates k intents into a single atomic transaction. The Simplicity evaluator performs a recursive fold  if any δ\_j fails its invariant, the entire sequence is invalidated.

---

## 8. State Discovery: Precopscan

Precopscan (v40.35, Next.js 14 / TypeScript) is an ephemeral, stateless observer that reconstructs the complete protocol state from raw Esplora API data at every page load. No database, no cache.

![image|690x412](upload://ibUYp51GELy8tBqgEoRAzIXYDo8.jpeg)

The discovery model anchors on **six fixed Taproot P2TR probe addresses** that every PRECOP settlement transaction is cryptographically forced to touch (because the `vault_covenant` TapLeaf commits the treasury SPK_HASH into the Merkle root).

**We recognize this introduces an indexer assumption** the completeness of the scan depends on these six probes capturing all protocol activity. We believe the covenant structure guarantees this (any spend bypassing these addresses is invalidated by the Bit Machine), but we have not produced a formal proof of completeness and welcome scrutiny on this claim.

The TUSM byte decoder discriminates between witness data types via domain-invariant plausibility guards (economic bounds on field values), distinguishing CDP state blobs from Schnorr signatures.

---

## 9. Empirical Validation (Mutinynet, Height 104,200)

The protocol has been validated on the Mutinynet Signet test network:

* **Reorg Survival:** Successfully weathered continuous 3-block reorganizations.

* **Witness Size:** 358 vBytes (standard CDP Repay).

* **Verification Time:** < 12 ms on standard LSVM.

* **Parallelism:** N=20 concurrent trades tested via Creator Dust Pools.

* **AMM Invariant (Test-AMM-K):** 1,000 randomized swaps, k monotonically increasing in 100% of cases. Mean k drift: +0.000000081113% per swap (truncation favors pool longevity).

* **Margin Call (Test-CDP-MARGIN):** Bit-perfect phase transition at the 120.0% boundary.

* **DEX Symmetry (Test-DEX-Symmetry):** 100% success rate across 1,000 randomized swaps; 128-bit cross-multiplication enforces fees to the last satoshi on both engines.

* **Witness Stack Alignment (Test-STACK-AUDIT):** Bit-perfect alignment between Rust orchestration and Simplicity consumption for the 12-item LIFO stack.

---

## 10. Technology Composition

PRECOP does not invent cryptographic primitives. It is a vertical composition of independently proven L1 technologies:

```
Layer 5: TUSM Covenant (Simplicity Bit Machine — simplicity-unchained, Blockstream)
Layer 4: Taproot MAST & Astrolabe State Encoding (BIP-341, Wuille et al.)
Layer 3: Oracle Sealing (Binohash PoW — Linus, 2024)
Layer 2: Hash-As-Signature (sha2-ecdsa — Robin Linus, 2024)
Layer 1: Thermodynamic Price Extraction (UTXOracle v9.1, Zkao + Openclaw)
Layer 0: Bitcoin L1 UTXO Set (Nakamoto, 2008)

```

Each layer provides a strictly isolated guarantee. No layer’s correctness depends on the internal implementation of any other.

---

## 11. Known Limitations

1. **TEE trust anchor:** The covenant enforcement depends on AWS Nitro Enclave physical security. If the TEE is compromised, the covenant exists only as a Taproot commitment  the network will not reject invalid transitions until Simplicity activation. Mitigation: 144-block CSV fallback to multisig quorum.

2. **Bootstrap Phase oracle:** During the current phase, the oracle relies on Schnorr attestations and a BTC stake. A malicious oracle can cause liveness failure (not fund theft). Mitigation: CSV refund path (≈ 24h unilateral exit).

3. **Price discovery quality:** UTXOracle is heuristic. It can be manipulated by sustained artificial transaction volume. The Entropy Guard raises the cost but does not eliminate the vector. This remains an open research problem.

4. **Head-UTXO discovery:** State transitions require knowledge of the current UTXO. High mempool congestion can cause transaction collisions. Fund safety remains cryptographically guaranteed.

5. **Fee spike griefing:** High L1 fees may render micro-CDP liquidations economically unviable, requiring higher over-collateralization margins.

6. **Simplicity activation:** Until Simplicity is activated on mainnet, enforcement is client-side only. We have designed the contracts so the same CMR hashes work under both regimes, but we do not assume any activation timeline.

---

## 12. Request for Review

We are publishing this architecture primarily as a **covenant research platform** and a security model for agent-managed capital. We seek feedback on:

1. **Simplicity VM divergence:** Are there known edge cases where running a local `simplicity-unchained` evaluator for PSBT introspection would diverge from how a future consensus-level evaluator would behave? This is critical for our Soft→Hard upgrade path.

2. **Sighash isolation:** Our TEE overrides the `witness_utxo` context provided by the agent before signing, recalculating the sighash internally. Does this completely close the vector for malicious PSBT blinding, or are there additional PSBT fields (e.g., `PSBT_IN_SIGHASH_TYPE`) that should also be overridden?

3. **Astrolabe Pattern security:** Is the domain tag `"PRECOP/State"` sufficient for isolation from BIP-341 taptweak and BIP-86 keypath? Are there edge cases in the tweak arithmetic we are missing?

4. **Oracle price correctness:** Binohash secures ordering and cost of attestation, but price discovery remains heuristic. What alternative on-chain price derivation methods or commitment schemes could improve fidelity without trusted APIs?

5. **DLC comparison:** We argue that continuous-state CDPs require algebraic transitions rather than CET enumeration. Is this a genuine design space distinction, or could DLC adaptor signatures be extended to cover the CDP use case?

6. **Precopscan completeness:** Could batching, CoinJoin, or non-standard change output patterns produce valid covenant transactions that bypass all six probe addresses?

7. **Covenant emulation value:** As a testing ground for future L1 covenants, what specific financial primitives (beyond CDPs and AMMs) would be most valuable to simulate and stress-test within this TEE/Simplicity hybrid environment?

8. **Utreexo integration for TEE UTXO validation:** Our Enclave currently trusts the external bridge for `witness_utxo` data (mitigated by internal sighash recalculation). With the recent release of `utreexod` v0.5 and Floresta 0.9, we are investigating feeding Utreexo inclusion proofs alongside the PSBT, allowing the Enclave to verify UTXO existence against internally maintained accumulator roots achieving full-node-equivalent validation within the TEE’s memory constraints (\~50 MB). We would welcome feedback on this integration path from anyone familiar with the Utreexo proof format.

We look forward to your technical critiques.

---

*PRECOP v0.0.2 — Covenant-Enforced Financial State Machines on Bitcoin*
*[laz1m0v](https://x.com/laz1m0v) — April 2026*

-------------------------

