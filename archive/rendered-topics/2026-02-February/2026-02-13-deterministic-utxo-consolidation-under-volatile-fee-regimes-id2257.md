# Deterministic UTXO consolidation under volatile fee regimes

babyblueviper1 | 2026-02-13 19:50:01 UTC | #1

Hi all,

I’ve been thinking about UTXO consolidation not purely as a fee optimization problem, but as a potential determinism and correctness surface in wallet transaction construction.

Specifically, I’m interested in cases where:

• Consolidation is triggered dynamically based on current fee environment
• Time-dependent consolidation policy (e.g. consolidate-now vs defer)
• PSBT construction depends on fee-rate comparisons against historical baselines
• Transaction structure may vary depending on mempool conditions

In such systems, two questions arise:

1. At what point does consolidation logic become part of the wallet’s correctness boundary rather than merely a policy layer?

2. Should deterministic guarantees (e.g., invariant input ordering, bounded fee regret, stable change handling) be considered enforceable properties in consolidation flows?

Potential failure modes I’m considering:

* Non-deterministic input selection under identical wallet state but different mempool snapshots

* Over-consolidation during transient fee dips leading to irreversible privacy loss (CIOH exposure)

* PSBT reproducibility issues if fee-estimation sources differ across construction attempts

* Edge cases around dust outputs or change threshold behavior when fee-rate volatility is extreme

I’m curious how others conceptualize this boundary.

Is consolidation strictly a wallet UX policy question, or are there scenarios where its interaction with fee estimation and PSBT construction introduces correctness or safety concerns that warrant stronger invariant guarantees?

Appreciate any thoughts or prior discussions I may have missed.

-------------------------

