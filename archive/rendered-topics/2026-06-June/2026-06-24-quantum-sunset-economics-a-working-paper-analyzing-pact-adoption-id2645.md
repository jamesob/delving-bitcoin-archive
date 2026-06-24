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

