# How wallet fingerprints damage Payjoin privacy

arminsdev | 2026-03-26 15:06:13 UTC | #1

I wrote up an analysis of how wallet fingerprints can be used to partition Payjoin transaction inputs and outputs by owner, potentially recovering the payment amount and undermining the privacy benefits. 

The takeaway: Payjoin privacy depends on fingerprint uniformity between sender and receiver. When wallets diverge on any observable dimension, that divergence becomes a partition signal.

Full post: https://payjoin.org/blog/2026/03/25/wallet-fingerprints-payjoin-privacy/

Interested in feedback on the methodology, additional fingerprint dimensions worth exploring, or cases where this analysis would fail.

-------------------------

0xB10C | 2026-03-27 09:52:01 UTC | #2

Great post! 

While reading, I stumbled over this part in [Example 1: Samourai Payjoin](https://payjoin.org/blog/2026/03/25/wallet-fingerprints-payjoin-privacy/#example-1-samourai-payjoin): 

> Input 0's DER-encoded signature is **71 bytes**. The r-value is 32 bytes with no leading zero pad, meaning r < 2^255. This is a grinded, low-R signature. Input 1's signature is **72 bytes** (the r-value requires a leading `0x00` pad, meaning r ≥ 2^255). This is an ungrinded, high-R signature. A single wallet running a consistent signing policy would either grind all signatures or grind none. Observing one of each is a strong signal that two different signing implementations contributed inputs.

I don't think this is correct. Today, around 60% of the ECDSA signatures on-chain are 71 bytes (https://mainnet.observer/charts/bitcoin-script-ecdsa-length/) and 60% have a low-r value (https://mainnet.observer/charts/bitcoin-script-ecdsa-r-value/). If no wallet would be grinding, it would be 50%. This is because a wallet that does not grind still has a 50% chance to produce a signature that is low-r. I think the only thing you can infer from a transaction with a high-r / 72 byte signature is that it's not been produced by the Bitcoin Core wallet (or Electrum, who also grind). This doesn't hold in the payjoin case though: Just because you see a 71 byte and 72 byte signature, it does not tell you that it were two wallets producing these. It could have been one wallet where it randomly used a low-r and a high-r value.

You might find the PR description of https://github.com/bitcoin/bitcoin/pull/13666 interesting too.

-------------------------

