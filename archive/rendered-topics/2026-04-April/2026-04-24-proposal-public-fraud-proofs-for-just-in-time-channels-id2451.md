# [Proposal] Public Fraud Proofs for Just-in-Time Channels

ThomasV | 2026-04-24 13:24:30 UTC | #1

Hello everyone,

I have long been unsatisfied by the trust model of just-in-time channels.

Here is a new proposal, that attempts to improve the situation using fraud proofs and game theory. The main idea is that clients should publish the payment preimage on-chain if their LSP is misbehaving.

https://gist.githubusercontent.com/ecdsa/dfa2d76a5fe845fd283c01b5ed12d274/raw/b690e4abb092318c794547f3f29f63fd58ea2bea/lsp_fraudproofs.pdf

Any feedback is appreciated. While the paper recommends serving op_return data using Electrum servers, it might also be possible to delegate all the indexing and serving work to nostr relays.

Thanks to tnull, f321x and SomberNight for reviewing early versions of this paper.

Thomas

-------------------------

