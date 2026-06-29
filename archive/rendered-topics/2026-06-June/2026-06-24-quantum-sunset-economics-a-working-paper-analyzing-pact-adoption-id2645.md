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

AdamISZ | 2026-06-24 12:43:07 UTC | #2

Re:

> 2. P2PKH-exposed: pay-to-public-key-hash where the corresponding public key
has been revealed by a prior outgoing transaction from the same address.
Quantum-exposed.
> 3. P2PKH-unexposed: P2PKH addresses that have received funds but never pro-
duced an outgoing transaction. Currently quantum-safe (only the hash is on-
chain).
> 4. P2SH / P2WSH: script-hash variants. Treated as quantum-safe at rest in this
analysis; per-script analysis would be required to identify exposed inner public
keys, which we leave to future work.

I don't understand your logic in 4. Basically every redeem script includes a pubkey "lock" requiring a signature, somewhere; otherwise it's trivially claimable at broadcast time by third parties. Not saying 100% but you can safely assume it's near that. So 4 should be in the same bucket as p2pkh and p2wpkh. (a complicating factor is a large number of them will be multisig which has a different risk profile because of (usually) needing the cracking of more than 1 key).

Do I understand correctly that this is why in Table 1 there is no "exposed" variant for p2sh? As per the above it should be very safe to assume that reused is exposed in this case, also, even before inspecting scripts. But then below table 1 you write:

> Percentages computed against a total circulating supply of approximately 19.83 mil-
lion BTC. The sum of all classified UTXOs in the table is approximately 20.04 million
BTC, exceeding the circulating supply by roughly 210,000 BTC **due to multi-address
output expansion in the multisig (P2SH and P2WSH) categories**; this inflation affects
unexposed totals only and does not bias the headline exposed figure of 5.018 million
BTC.

(emphasis mine) . It wasn't obvious to me what you meant by this, but the only thing I can deduce is that you mean: the software you're using erroneously assigns a redeem script like say a 3 of 3 multisig to *3* different addresses, one for each key. Is that correct? It seems a bit bizarre as there's a completely standard way to assign addresses to p2-sh sPKs and their corresponding redeem scripts. But I couldn't make sense of the text another way.

So, very tentatively assuming what I've said so far is correct, I think the data should treat used p2-sh as exposed-when-used, seems like the correct classification at the high level.

-------------------------

AdamISZ | 2026-06-24 13:31:27 UTC | #3

Another point that I think has been mentioned elsewhere: xpubs are another vector of attack. Unfortunately this one is very difficult (impossible?) to measure, so it's hardly a critique of your quantitative analysis other than, you should probably mention this additional risk. Many services have existed that actually require users to share xpubs; in some cases it's even been public data, but of course usually not: usually it's a relationship between a client and a service. Such as, tax reporting software, certain blockchain monitoring services, security type services etc. Any databases can be leaked, so it's hard to know what xpubs are in the public or quasi-public domain. But to state the obvious, a publically exposed xpub means that coins held at an address A that was never used before is not unexposed, but exposed (to CRQC threat). It's probably a huge number of addresses in this category (that is, BIP32 on an unhardened branch, holding funds and never before used), but nearly impossible to know what fraction (probably low) of that total is exposed by the aforementioned xpub threat.

(Sorry that my comments do not address any of the questions you wanted feedback for; this was just the part that captured my interest).

-------------------------

ahmedraza | 2026-06-27 18:53:55 UTC | #4

Hi @AdamISZ, 

Thanks for the detailed read. I've incorporated your points into a v2 of the paper. The work surfaced a couple of empirical results worth sharing back.

The P2SH and P2WSH classification you flagged got rewritten in Section 3.1. The exposed-when-used binary that the paper already applies to P2PKH and P2WPKH now also applies to script-hash address types: a P2SH or P2WSH address is exposed once it has produced an outgoing spend that reveals the redeem or witness script on-chain. Re-running the cohort analysis with this classification gives roughly 1.3 million BTC in newly classified exposed P2SH and 685,000 BTC in newly classified exposed P2WSH, for an additional 1.99 million BTC of exposed supply. The v1 headline of 25.30 percent moves to 35.30 percent in v2; the shift was larger than I expected when starting the revision.

The double-counting footnote on Table 1 that you flagged as hard to parse has also been rewritten. The new version states the mechanic directly: the BigQuery output expansion counts multisig outputs once per constituent address rather than once per UTXO, which inflates the script-hash row totals by approximately 210,000 BTC. The reported headline exposed figure is unaffected.

On the k-of-n weighting question, I went after this empirically and ran into a constraint worth flagging. The `bigquery-public-data.crypto_bitcoin` dataset does not expose witness data anywhere; it is not in the flat inputs table or in the nested inputs struct on the transactions table. The closest available signal is `inputs.required_signatures`, which the bitcoin-etl pipeline that builds this dataset populates as 1 for spends with single-key redeem or witness scripts (P2WPKH-in-P2SH wraps and analogous P2WSH single-key constructions) and as NULL for multisig or non-standard scripts. The exact k threshold for k-of-n multisig is not exposed. So what I can deliver is a binary classification, not a full k-by-k decomposition.

That said, the binary result is itself informative. Across all exposed P2SH addresses (approximately 1.3 million BTC), 73.4 percent (about 955,000 BTC) falls into the multisig-or-other bucket and 26.6 percent (about 345,000 BTC) is single-key; the single-key fraction reads as P2WPKH-in-P2SH wrapping, consistent with how widespread that pattern became post-SegWit. Across all exposed P2WSH addresses (approximately 687,000 BTC), 99.5 percent is multisig-or-other and 0.5 percent is single-key; the lopsided split reads as Lightning channel funding outputs and institutional multisig wallets dominating the P2WSH cohort.

Combined across both script-hash types, approximately 82.4 percent of the newly classified cohort (about 1.64 million BTC) is multisig-or-other and 17.6 percent (about 350,000 BTC) is single-key. Under a budget-constrained quantum-attack model where the marginal cost is dominated by the number of ECDLP-solves required, the single-key fraction would be drained before the multisig fraction. About 8.3 percent of total Bitcoin supply, the multisig portion of the newly classified cohort, costs the attacker at least two key compromises per UTXO rather than one.

Precise k-by-k extraction for the multisig bucket is genuinely future work. It needs either witness data via a different ingest of the chain or a scriptSig push-walker that locates the embedded redeem script for each spend. Both are tractable but neither is in this revision.

While I was thinking through what the multisig classification implies for the policy argument, I added a new Section 6.5 framing the multisig-protection horizon as regime-dependent. The short version: in an early-CRQC regime where compute is scarce and each ECDLP solve takes hours-to-days, the second solve required by 2-of-3 multisig is a meaningful cost and the budget-constrained attacker drains single-key targets first. In an industrialised-CRQC regime with parallel solves and cheap compute, the multisig advantage erodes. Multisig is therefore a useful interim defence but not a durable one, which actually strengthens rather than weakens the institutional-pooling argument: pooling converts a temporary multisig defence into a durable post-quantum one without requiring each holder to migrate their architecture individually.

On the xpub-disclosure point, I added a new Section 3.8. The discussion argues that the on-chain headline is a lower bound on actual quantum-exposed value because publicly leaked or shared xpubs expose all derived addresses on the disclosed branch, including addresses that have never produced an outgoing spend. The new paragraph also connects the xpub-disclosure risk to the paper's central argument about institutional pooling: the cohort most exposed to xpub disclosure is precisely the cohort that would benefit most from an institutional commitment service, because the disclosure typically originates from the same custodial and reporting relationships that a pooling service would centralise.

While I was in the revision, I also re-ran the P2TR alternative classification (the sensitivity that treats P2TR as effectively hashed rather than inherently exposed). The v2 alternative, with P2SH and P2WSH treated as exposed-when-used and P2TR treated as exposed-only-after-spend, is 6,854,421 BTC, or 34.57 percent of supply. This replaces a rough estimate I had been carrying with the chain-tip-precise figure.

Thanks again for the engagement. Both points materially improved the analysis.

-------------------------

AdamISZ | 2026-06-29 12:35:47 UTC | #5

[quote="ahmedraza, post:4, topic:2645"]
On the xpub-disclosure point, I added a new Section 3.8. The discussion argues that the on-chain headline is a lower bound on actual quantum-exposed value because publicly leaked or shared xpubs expose all derived addresses on the disclosed branch, including addresses that have never produced an outgoing spend. The new paragraph also connects the xpub-disclosure risk to the paper’s central argument about institutional pooling: the cohort most exposed to xpub disclosure is precisely the cohort that would benefit most from an institutional commitment service, because the disclosure typically originates from the same custodial and reporting relationships that a pooling service would centralise.
[/quote]

Yes that is an interesting/scary aspect, isn't it. At a deep level, the whole concept of "xpub" could be argued to have been flawed from the start: an xpub does indeed count as being on the "pub" side in the classic public vs private distinction from asymmetric cryptography, but it's a hybrid, and makes a convenience-safety tradeoff that was probably never acceptable (probably! that's very debatable). I remember many years ago, even before anyone mentioned quantum computers, wallet software (good software anyway!) printing warnings to users to not share xpubs. It was a bad privacy violation, as well as a security threat (because you can not reasonably expect users to understand the cross-privkey-leakage).

Glad I could help with the analysis.

-------------------------

