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

tnull | 2026-04-27 09:57:20 UTC | #2

Thanks for this interesting proposal! As mentioned elsewhere I have to admit I’m a bit skeptical regarding how much this would indeed gain us in practice, especially given the added complexity of this scheme.

I also think the scheme as currently outlined in Section 3 has a griefing vector as, if I understand correctly, after step 2 Alice would have everything she needs to publish a fraud proof, but could then maliciously stall out the channel open itself, thereby grief Bob’s reputation? If I’m not mistaken this might however be fixable by reordering the steps so the whole flow (i.e., sending of the UTXO commitment in particular) is only conducted after Bob receives `funding_signed` from Alice, but of course before he broadcasts the funding transaction.

-------------------------

