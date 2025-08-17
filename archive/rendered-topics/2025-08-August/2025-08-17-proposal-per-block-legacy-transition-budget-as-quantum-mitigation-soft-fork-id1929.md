# Proposal: Per-Block Legacy Transition Budget as Quantum Mitigation (Soft Fork)

MartinS84 | 2025-08-17 02:03:40 UTC | #1

Hi,

with ongoing discussions about post-quantum (PQ) migration strategies (see recent BIPs around freezing legacy outputs), I’d like to outline an alternative approach that aims for a *softer* migration path.

### Problem

If large-scale quantum attacks on ECDSA/Schnorr become practical, adversaries could rapidly drain exposed legacy UTXOs (public keys revealed, reused addresses, etc.). Proposals to outright freeze legacy coins after a deadline are simple, but they risk orphaning honest users who migrate late, and may cause political controversy over “lost” coins, censorship etc.

### Idea: Transition Budget

Instead of freezing, introduce a **per-block budget** that limits how much value can move **out of legacy cryptography** per block:

* Define **Legacy inputs** = any input validated with ECDSA/Schnorr.

* Define **Transition(tx)** =
  `sum(value of legacy inputs) − sum(value of legacy outputs)`
  (i.e., the amount leaving Legacy: going to PQ outputs, OP_RETURN/burn, or fees).

* Require that for every block:
  `Σ Transition(tx) ≤ LEGACY_TRANSITION_LIMIT(height)`

* Additionally, for each tx with Legacy inputs, require at least one **PQ output ≥ 1 sat**.

* Fees count as “leaving Legacy”, so miners cannot bypass the limit by self-paying fees.

* Legacy change outputs remain allowed; only the *net exit* from Legacy consumes budget.

### Rationale

* **Soft Fork**: This is a tightening rule (forbids blocks that spend too much Legacy at once).

* **Gradual migration**: Users can still move funds, but the system rate-limits overall Legacy outflow.

* **Miner loophole closed**: Fees are included in the budget.

* **Minimal fairness**: Every Legacy spend contributes at least 1 satoshi to PQ migration.

* **Analogy**: Similar to block-wide sigop/weight limits, but applied to Legacy-to-PQ transition value.

### Example

* Spend 10 BTC Legacy → 9.99999999 BTC Legacy + 1 sat PQ, fees 0 → Transition = 1 sat.

* Spend 50 BTC Legacy → no outputs (all fees) → Transition = 50 BTC.

* Spend 2 BTC Legacy → 2 BTC PQ → Transition = 2 BTC.

### Open Questions

* What parameters for `LEGACY_TRANSITION_LIMIT(h)`? (1 BTC/block floor? Decay schedule?)

* Should a minimum `Transition(tx)` (dust-like threshold) be enforced by consensus or left to policy?

* Which PQ scheme(s) to standardize first (SPHINCS+, Dilithium, LMS/HSS, etc.)?

* How to coordinate deployment with the PQ-BIP introducing new verification rules?

### Goal

The intention is to discuss whether a **rate-limited migration** offers a reasonable middle ground between “freeze all legacy coins” and “do nothing until quantum arrives.”

Feedback on feasibility, risks, and whether this approach could fit within Bitcoin’s consensus philosophy would be very welcome.

Thanks!

-------------------------

