# How wallet fingerprints damage Payjoin privacy

arminsdev | 2026-03-26 15:06:13 UTC | #1

I wrote up an analysis of how wallet fingerprints can be used to partition Payjoin transaction inputs and outputs by owner, potentially recovering the payment amount and undermining the privacy benefits. 

The takeaway: Payjoin privacy depends on fingerprint uniformity between sender and receiver. When wallets diverge on any observable dimension, that divergence becomes a partition signal.

Full post: https://payjoin.org/blog/2026/03/25/wallet-fingerprints-payjoin-privacy/

Interested in feedback on the methodology, additional fingerprint dimensions worth exploring, or cases where this analysis would fail.

-------------------------

