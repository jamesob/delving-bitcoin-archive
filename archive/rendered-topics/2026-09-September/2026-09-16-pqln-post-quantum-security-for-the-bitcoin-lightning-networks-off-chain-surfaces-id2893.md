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

roasbeef | 2026-09-29 01:24:47 UTC | #2

> This topic has come up here before, in @roasbeef’s layer-by-layer post from
> May, which maps these same layers and estimates their PQ message sizes

Happy to see others are working on this topic!

For the sake of conservative defense in depth, my opinion is that for each
layer a hybrid scheme is used, as the space overhead of classical compared to
PQ is negligible.

> Lightning has no certificate infrastructure, so we distribute the
> post-quantum keys through gossip itself.

Yeah the gossip layer is effectively our own form of decentralized cert
management infra. The existing gossip protocol is fairly fragile in that it
wasn't designed with the requisite amount of extension footholds for newer
channel types like Taproot Channels. [There's an on going
effort](https://github.com/lightning/bolts/pull/1059) to revamp this, fixing
many issues with the old protocol, while paving the way for new ways to
advertise channels.

>  The node_announcement messages carry the node’s ML-DSA and ML-KEM public
>  keys along with an ML-DSA signature, and the channel_update messages carry
>  just the signature.

One could imagine a transitionary period in which the new revamped gossip
messages maintain the existing classical signatures while the new PQ signatures
and keys piggy back onto the existing messages. Clients that don't know about
the PQ signatures would still be able to verify the integrity and authenticity
of the payload.

Even without the classic signatures, as brought up in my post, one issues is
the expanded bandwidth overhead. The current gossip protocol is particularly
inefficient, in that there's zero active reconciliation of inventory between
nodes. Instead a node is likely to get the same updates and announcements over
and over again from its peers. Today this is wasteful, but note egregious
bandwidth wise. However if the params in my original post are selected, the
naive extrapolation points towards a nearly 19x increase in steady state gossip
bandwidth. Initial graph download becomes heftier too, but nodes only really do
that once, so it isn't as big of a deal.

> Swap in FN-DSA-512 and everything gets easier.

If this is layered on, then that further expands the amt of active
cryptosystems we'd be relying on from ~1 to 4 (ML-KEM, ML-DSA, FN-DSA-512,
SLH-DSA - assumed for base layer). If FN-DSA-512 is a serious candidate, then
why not also drop ML-DSA in favor of that instead?

Also no reason to be tied to BOLT-11 and some of its design defects. BOLT 12
includes a currently-unexposed serialization format that is much more sane. The
new format doesn't have limits on the size of a tagged field, etc, etc. 

Y'all mention just the signature field, but what about the public key as well?
IIUC, FN-DSA-512 supports a pubkey recovery mode, but it only saves a few
hundred bytes compared to just including the public key in plain site.

> Offers (BOLT 12): The gossip pin doesn’t help much here. The node behind an
> offer is often unannounced and sitting behind a blinded path, so you never
> saw its keys in gossip to begin with

I don't go into this in my original post, but the Offers portion of BOLT 12
relies on blinded paths. Therefore any PQ upgrade across all layers would also
need to affect blinded paths, which invariably leads to a larger QR code size
due to the ML-KEM addition.

As an example, if you want a blinded path with just 2 hops (the introduction
node plus the receiver), the final packet is ~4.5 KB (~1 KB for the
encapsulation, and another 1 KB ish for the public key). Assuming conservative
QR code packing, then it also can no longer fit inside of a single QR code.

> We couldn’t just put ML-KEM inside the onion, because one ciphertext on its
> own would nearly fill the whole 1300-byte packet.

Sure, but there's no reason to be stuck with the old packet format. It's
already a backwards incompatible change if the old nodes can't derive the new
hybrid shared secret. AFAICT, you also aren't saving any space by doing this
either.

> The ciphertexts ride next to the onion in update_add_htlc, in a fixed list of
> 20 slots

Meaning this information is now _outside_ the onion? Is the associated data MAC
check expanded to also cover this extra blob? Again, why not just expand the
packet format as in the KEM Sphinx paper I mentioned in my post?

Note that KEM Sphinx drops the group element blinding step at each hop, so the
performance may end up roughly being on par (compared to classical Sphinx).

>  The trick is that we fill the unused slots with dummies that look exactly
>  like real ciphertexts, so a routing node can’t count the real ones and work
>  out how long the route is.

An important attribute of the Sphinx packet format is that each packet is
unlinkable, every incoming and going packet should look unrelated. However to
my knowledge an ML-KEM encapsulation cannot be randomized. Further, if each hop
reads their ciphertext from a fixed slot in this appended packet, they'd
trivially be able to determine which position in the over all route they are.

Sphinx achieves unlinkability by re-randomizing the entire mixnet packet at
each hop. Any new format should strive to maintain a similar level of security. 

> A PQLN node can join the network today, without anyone else having to upgrade
> first.

Well, not really. As LN is a p2p network, various degrees of interop are
required for a new to be able to actively participate in the network.

> One of the open questions is the pinning. It’s trust-on-first-use, so it only
> protects two nodes that met before a quantum computer arrives.

That's roughly how things work today. You can bridge the transition by having
nodes sign under both the new and old identity.

------

With all that said, I think the best layer to start is the transport layer. 

If we're all on board with a hybrid model, rather than adopt PQ Noise in
isolation, we can simply layer the two approaches, so we upgrade to PQ inside
the existing classical channel. The alternative is to work out and/or select a
secure hybrid combiner, which is to be used instead.

The immediate barrier most will likely run into is access to secure vetted
libraries for ML-KEM. Lucky for lnd, with Go 1.26.1, Go ships with an ML-KEM
package as part of the standard library. 

-- Laolu

-------------------------

ahmet-kurt | 2026-10-01 01:55:58 UTC | #3

Thank you Laolu @roasbeef for the detailed reply. I feel that my post left out some context about the work, so let me add a few notes first and then I'll reply to each of your points.

1) I spent a lot of time trying to make the implementation work with unmodified (vanilla) nodes. That's why I tried to work with what we have at hand today, without waiting for spec changes. The interoperability section of the paper talks about what we tested. Basically, all main Lightning functionality works between PQ and vanilla nodes. ML-KEM surfaces in PQLN such as PQ transport, PQ payment onions and PQ blinded paths require both nodes to be PQ. ML-DSA surfaces such as PQ gossip, PQ invoices, PQ offers do not need the other side to upgrade. Either side can upgrade first here and nothing breaks, but the protection itself needs both sides to be PQ.
2) The paper is 13 pages because of the page limit of the journal we submitted to, so we had to leave out some explanations and tests. Probably we will be adding more content in the revision cycles.
3) Some of the implementation details are only explained in the codebase and not in the paper. So I'd invite anyone interested to look at the codebase too, and any feedback there is very welcome.

Now my attempt to answer your comments :slightly_smiling_face::

> [There’s an on going effort](https://github.com/lightning/bolts/pull/1059) to revamp this, fixing many issues with the old protocol, while paving the way for new ways to advertise channels.

Oh, great to hear. I'll be looking at this more closely.

> the new PQ signatures and keys piggy back onto the existing messages.

Exactly, that is pretty much what we did for gossip messages.

> If FN-DSA-512 is a serious candidate, then why not also drop ML-DSA in favor of that instead?

I might have worded that part badly. By "swap in" I meant replacing ML-DSA, not adding FN-DSA on top of it. We kept ML-DSA-44 as the default only because FIPS 206 isn't final yet. Once it is, I agree it makes sense to drop ML-DSA for FN-DSA, since size is our biggest cost.

> Also no reason to be tied to BOLT-11 and some of its design defects... Y’all mention just the signature field, but what about the public key as well?

Right, BOLT 12 was made PQ as well, and it was one of the harder ones. 
The key is there too, split across three optional tagged fields. With ML-DSA-44 the invoice is 4286 characters without the key and 6396 with it, and FN-DSA-512 drops these to 1473 and 2915. The key only helps a payer that already trusts it, though. An announced payee's key comes from the gossip pin anyway. For an unannounced payee, a key in the same invoice as the signature proves nothing, because a quantum attacker can replace both. So I agree on moving past BOLT 11, since an offer can give the payer an anchor and a BOLT 11 invoice can't.

> any PQ upgrade across all layers would also need to affect blinded paths.

Oh, you just made me realize I never talked about PQ blinded paths in the writeup. The recipient builds the path, so it encapsulates to every hop's ML-KEM key and folds each secret into that hop's route-blinding secret. The payer only delivers the ciphertexts, so it doesn't need the hops' public keys. On message paths, each later ciphertext rides inside the previous hop's encrypted data. On payment paths, the ciphertexts ride next to the onion in a second list of 10 slots, which shares the gaps below.

You're right about the QR code too. Our offer with a one-hop PQ path already comes out at 4332 characters, just past the 4296 that fit in the largest QR code. Your two-hop case adds another ciphertext and should land around 6000. Two changes should help, though I haven't tried them yet. The offer could commit to a hash of the per-offer key and let the invoice carry the key itself. Also, the recipient can re-derive the secret for its own hop like the per-offer key, so that hop needs no ciphertext. A two-hop PQ offer would then come to roughly 2400 characters.

> Sure, but there’s no reason to be stuck with the old packet format.

We tried to extend existing messages with optional TLVs wherever we could, rather than define new formats. That's why the onion kept its format. But you're right that this buys nothing for the onion, since a hybrid hop needs the upgrade anyway. The side list isn't smaller than a KEM Sphinx header either.

> Is the associated data MAC check expanded to also cover this extra blob?

No, and that's a real gap. Each hop checks its own entry implicitly. A changed entry decapsulates to a different secret, so that hop's HMAC fails. However, nothing checks the entries for later hops. So a routing node could tamper with a later entry and learn from the failure whether the route reaches that far. In Sphinx the very next hop would catch the change.

> Further, if each hop reads their ciphertext from a fixed slot in this appended packet, they’d trivially be able to determine which position in the over all route they are.

This part should be fine. Every hop reads the front entry and rotates it to the back, and the dummies are encoded exactly like real ciphertexts. A hop therefore sees its own entry at the front, followed by 19 indistinguishable entries, so its position stays hidden. Your unlinkability point stands though. ML-KEM ciphertexts can't be re-randomized and the list only rotates, so two colluding hops can match it. Today every hop of a payment already sees the same payment hash, so we don't lose anything yet. However, the list would become a real leak as soon as hops stop sharing a payment hash, for example with PTLCs. KEM Sphinx fixes both problems, since each later ciphertext hides under the earlier layers and the MAC covers the whole header. So I'd rather put the ciphertexts inside the onion layers, like KEM Sphinx does, than patch the side list. And your idea from May of lowering the max hop count helps either way, since 8 slots would cut our 21.8 kB list to about 8.7 kB.

> As LN is a p2p network, various degrees of interop are required.

Right, that line is too strong. I meant that a PQLN node keeps working with vanilla peers, but PQ protection only kicks in when the other node is also a PQLN node. For a fully PQ payment, that means every node on the route.

> You can bridge the transition by having nodes sign under both the new and old identity.

Agreed, and the `node_announcement` already does this. The ECDSA signature covers the PQ keys, and the ML-DSA signature covers the announcement. That bridges the transition for any node that sees the announcement before a quantum computer shows up. I'm less sure about a node that first sees it afterwards, because by then the old identity's signature can be forged.

> With all that said, I think the best layer to start is the transport layer.

Agreed. Our handshake follows the second option from your May post, and it turned out simpler than I expected. Noise's chaining key already works as the combiner. We call MixKey on each ML-KEM secret right after the ECDH secret of the same act, so the session keys stay secure as long as either secret does. The Noise HFS extension does the same thing.

> The immediate barrier most will likely run into is access to secure vetted libraries for ML-KEM.

Agreed, and I should be upfront that our prototype uses the fips203 and fips204 crates, which are pure Rust but still marked experimental. For rust-lightning, libcrux looks like a good candidate. Its ML-KEM is formally verified, and OpenSSH uses it too.

By the way, Bitcoin Optech covered the post in [Newsletter #424](https://bitcoinops.org/en/newsletters/2026/09/25/). I couldn't join their live podcast and instead prepared a recording. It should be up on their [podcast page](https://bitcoinops.org/en/podcast/) soon.

Thanks again for the careful read!

Ahmet

-------------------------

