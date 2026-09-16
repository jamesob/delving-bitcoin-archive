# PQLN: Post-Quantum Security for the Bitcoin Lightning Network's Off-Chain Surfaces

ahmet-kurt | 2026-09-16 07:30:49 UTC | #1

Hi everyone,

This is my first post here. I'm Ahmet Kurt from East Texas A&M University doing research mainly on Lightning and its various applications.

We built **PQLN**, a hybrid post-quantum (PQ) extension of Lightning with a complete implementation in rust-lightning. To the best of our knowledge, it is the first post-quantum design for Lightning that comes with an implementation and measurements on real nodes. This topic has come up here before, in @roasbeef's [layer-by-layer post from May](https://delvingbitcoin.org/t/post-quantum-lightning-layer-by-layer/2479), which maps these same layers and estimates their PQ message sizes. My coauthors and I have been building PQLN since around December 2025. A full implementation took time, and we wanted enough testing behind it to be confident the design was sound. We posted the paper to arXiv a few days ago, and the code is on GitHub.

**Paper**: https://arxiv.org/abs/2609.13781

**Code**: [pq-rust-lightning](https://github.com/ahmet-kurt/pq-rust-lightning) adds about 11,000 LoC behind a `post-quantum` cargo feature, and [pq-ldk-sample](https://github.com/ahmet-kurt/pq-ldk-sample) is LDK's sample node modified to work with our fork.

**Scope:** The keys inside the funding, commitment, and penalty transactions can't move to post-quantum without a consensus change, so we leave those alone. Everything else in Lightning lives in messages between nodes, and all of that can move with a software update. It also has to, since a post-quantum output type on Bitcoin does nothing for Lightning's own off-chain surfaces. And the clock is already running. An adversary can record Lightning traffic today and decrypt it the day a big enough quantum computer shows up. So in this work, we protect all off-chain surfaces of Lightning, namely BOLTs 4, 7, 8, 11, and 12. Details for each surface are below.

**Gossip (BOLT 7):** Lightning has no certificate infrastructure, so we distribute the post-quantum keys through gossip itself. The `node_announcement` messages carry the node's ML-DSA and ML-KEM public keys along with an ML-DSA signature, and the `channel_update` messages carry just the signature. Every node pins those keys the first time it sees them, and after that it rejects any announcement that shows up with substituted or missing keys. We left the `channel_announcement` message classical, because two of its four signatures are rooted in Bitcoin. Making only the other two post-quantum would leave the message half forgeable and buy no real PQ benefit.

**Transport (BOLT 8):** We make the Noise_XK handshake hybrid with two ML-KEM encapsulations. One goes to the responder's pinned static key, which authenticates it, and one goes to a fresh ephemeral key, which gives forward secrecy. Both shared secrets get folded into the Noise chaining key alongside the classical ones. There's no in-band negotiation of any of this. The hybrid handshake runs on its own port instead, because a negotiation message is exactly the kind of thing a quantum adversary could rewrite to force a downgrade.

**Invoices (BOLT 11):** The problem with BOLT 11 is size. A tagged field holds at most 639 bytes, an ML-DSA-44 signature is 2420 bytes, and there's no way to fit one into the other. So we split the signature across four fields and the optional public key across three. A signature-only PQLN invoice comes out to 4286 characters, which still fits inside the largest alphanumeric QR code, with 10 characters to spare. Swap in FN-DSA-512 and everything gets easier. The signature drops to two fields and the invoice falls to around 1500 characters, well clear of the QR limit.

**Offers (BOLT 12):** The gossip pin doesn't help much here. The node behind an offer is often unannounced and sitting behind a blinded path, so you never saw its keys in gossip to begin with. The offer carries its own anchor instead. It commits a fresh ML-DSA key, one per offer, and when the invoice comes back the payer checks it against that exact key before any HTLC goes out.

**Payment onion (BOLT 4):** We couldn't just put ML-KEM inside the onion, because one ciphertext on its own would nearly fill the whole 1300-byte packet. So the onion keeps its format untouched, and the per-hop Sphinx secrets become hybrid instead. The ciphertexts ride next to the onion in `update_add_htlc`, in a fixed list of 20 slots. The trick is that we fill the unused slots with dummies that look exactly like real ciphertexts, so a routing node can't count the real ones and work out how long the route is.

**Cost:** Computation turned out not to be the problem. The slowest thing we do is ML-DSA signing, at 0.33 ms. Over an emulated Internet link with 50 ms of round-trip latency and 10 Mbit/s of bandwidth, a payment takes somewhere between 19 and 53 ms longer per hop, and most of that is just moving the 21.8 kB ciphertext list around, not the crypto.

The real cost is bandwidth. On regtest networks of real nodes, a PQLN node pulls down about 10x the gossip of a vanilla node and stores about 9x as much. Put that on today's network of 33,000 public channels, and a node joining from scratch downloads roughly 270 MB instead of 26 MB. For comparison, roasbeef's estimate for turning every gossip signature into ML-DSA-44 was a 29x blowup. We stay at 10x because we left the biggest gossip message, `channel_announcement`, classical. Move to FN-DSA-512 and the growth drops to about 4x.

**Interoperability:** We ran 12 scenarios mixing PQ and vanilla nodes, from opening a channel all the way to BOLT 12 refunds and async payments. The short version is that nothing ever got stuck. When both ends spoke PQ, the payment went through with full protection. When a vanilla node sat somewhere on the path, the payment quietly fell back to classical and still completed. And if you turn on the require-PQ flag, a node refuses to be downgraded, so instead of going classical it fails closed before any HTLC moves. That last part is really the point. A PQLN node can join the network today, without anyone else having to upgrade first.

**Open problems:** One of the open questions is the pinning. It's trust-on-first-use, so it only protects two nodes that met before a quantum computer arrives. I have a couple of ideas for closing that gap without a consensus change, both written up in the paper, but I'd rather hear how people here would approach it first.

The TLV types, invoice tags, and feature bits still need assignment through the BOLT process. A rigorous interoperability study against the other implementations (CLN, LND, and Eclair) is also still on the future-work list.

This last one is really a question for the rust-lightning maintainers. For PQ gossip to spread across the network, vanilla nodes have to relay it too. But right now that can't happen, because `MAX_EXCESS_BYTES_FOR_RELAY` in rust-lightning is hardcoded at 1024 bytes and the PQ records go well past it. So PQ gossip only propagates among PQ nodes. If that limit were raised, vanilla nodes would carry it as well. Are there any discussions to raise `MAX_EXCESS_BYTES_FOR_RELAY`?

Thanks for reading and any feedback is appreciated!

Ahmet

-------------------------

