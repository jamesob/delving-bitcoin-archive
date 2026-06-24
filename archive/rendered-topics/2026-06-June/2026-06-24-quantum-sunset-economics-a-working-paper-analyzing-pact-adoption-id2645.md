# Quantum Sunset Economics: a working paper analyzing PACT adoption

ahmedraza | 2026-06-24 02:27:47 UTC | #1

Hi all,

I'm Ahmed Raza, an independent researcher. I posted a working paper analyzing the economics and governance of Dan Robinson's PACTs proposal:

* [The Paper](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=6901220)
* [Reference implementation](https://github.com/ahmedrazaumer1/pact-commit)

Headline findings:

* **The exposed-key UTXO surface is approximately 25% of circulating supply** (around five million coins, including the 1.1M Patoshi cluster), measured via a BigQuery cohort analysis included in the supplementary materials.

* **Under any convex privacy-cost specification, the largest dormant holders are the population least likely to commit voluntarily.** This is a feature of any voluntary mechanism, not a defect in Robinson's design.

* **The implication: early standardization paired with a pooling service that evens out privacy costs across holder sizes** is the highest-leverage action available now. A solo holder of 100k BTC faces a privacy cost ten thousand times larger than a holder of 10 BTC; only a pooling layer makes protective coverage reach the largest tail.

The paper also contains a five-criterion comparative framework across six policy options (do nothing, BIP-360 only, BIP-361 sunset variants, Rubin's proposal, PACTs), a regulatory analysis of three CRQC-arrival scenarios with VASP and OFAC obligations, and a Python reference implementation of the commitment side of the protocol.

Feedback I would value:

* The convexity assumption in §5. Reasonable or too aggressive?

* The pooling service in §5.4. Is anyone already thinking about this? What would a viable implementation look like (custodial, non-custodial, multisig-based)?

* Whether P2TR should be treated as exposed in the same sense as P2PK (§3.6).

* The PACT vs. Rubin synthesis (§4). What features should win?

Thanks!

-------------------------

