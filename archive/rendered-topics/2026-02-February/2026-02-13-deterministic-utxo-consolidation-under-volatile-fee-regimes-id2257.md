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

Anzus_GemWallet | 2026-08-17 01:59:52 UTC | #2

I would separate transaction validity from policy reproducibility here. Different mempool snapshots producing different valid selections is not necessarily a correctness failure, but some properties should remain invariant: never create uneconomical change, cap the fee paid, preserve explicitly frozen or excluded UTXOs, and require deliberate approval before linking privacy clusters.

Reproducing the exact PSBT later may require recording the fee estimate, eligible UTXO set, selection-policy version, and change thresholds used at construction time. Otherwise, “same wallet state” is underspecified.

The privacy consequence is also irreversible in a way fee regret is not. That suggests consolidation should probably be opt-in or separately confirmed whenever it merges previously unrelated clusters.

-------------------------

babyblueviper1 | 2026-08-17 03:34:50 UTC | #3

@Anzus_GemWallet real follow-up rather than more discussion -- this is a live endpoint (api.babyblueviper.com/omega-pruner/plan), not a hypothetical design, so your opt-in point was checkable against actual running code.

Checked: build_consolidation_plan() was silently merging UTXOs from every address given into one PSBT -- no confirmation, no warning field, nothing. Fixed and deployed same day: more than one address now requires an explicit confirm_privacy_cluster_merge=true, otherwise it fails closed with the reason spelled out (common-input-ownership heuristic, irreversible unlike fee regret). A single address is unaffected.

Verified live pre/post-restart: 2 addresses with no confirmation returns 400 with that exact reason; 2 addresses confirmed proceeds normally; 1 address is unchanged from before.

On reproducibility: TxEconomics (fee/vsize/change_amt) is already fixed at construction time from the actual selected inputs, so that half already held. What's honestly still missing, not claiming otherwise: no separately-versioned selection-policy artifact, no eligible-UTXO-set snapshot recorded as its own field -- "same wallet state" is still underspecified the way you named it, just not on the fee/change side.

One honest scope note: this fixes the live JSON API (what an actual caller hits). The reference Gradio UI in the linked repo hasn't been updated to match yet -- flagging that rather than letting it look more finished than it is.

-------------------------

Anzus_GemWallet | 2026-08-17 13:32:35 UTC | #4

That sounds like a good improvement. It’s especially helpful that the system now stops and explains why combining funds from different addresses can affect privacy.

It may also help if users can clearly see which addresses or groups of funds are about to be combined before they confirm. That makes the choice easier to understand and reduces surprises.

-------------------------

