# Post Quantum Lightning: Layer by Layer

roasbeef | 2026-05-05 04:36:48 UTC | #1

Hi y’all, 

Over the past few weeks I’ve received some direct and indirect questions asking how the rise of quantum computers may affect the Lightning Network. I searched and didn’t see anything concrete written on the topic, so I figured that I’d write something up! 

To answer the question of how quantum computers may affect the design of Lightning, a useful starting point is recognizing that any layer that uses cryptography that rests on classical security assumptions requires modifications. 

So what we can do is pick out the BIPs that directly rely on EC crypto, then walk backwards from there to see how we might replace them with post-quantum primitives. 

If we look at the BOLTs, these documents jump out as they each have direct EC cryptography use:

* BOLT 11+12 
  * Signed invoice formats that rely on Schnorr to authenticate invoices. 
* BOLT 8 
  * Encrypted p2p transport that uses static+ephemeral EC keys with ECDH to derive a series of ephemeral transport secrets. 
* BOLT 7 
  * P2P gossip messages signed with a series of Schnorr signatures. 
* BOLT 4
  * Sphinx-derived mixnet packet that leverages the commutative properties of Elliptic Curve Crypto to make a compact mixnet packet. The “one little trick” is the multiplicative blinding factor that’s used to give each hop a fresh ephemeral EC key. Large savings as we avoid needing to place an EC key for each hop directly in the packet. 
* BOLT 2 + 3 + 5
  * The actual LN channel format, where multi-sig uses Schnorr/ECDSA signatures and payment hashes in the script are RIPEMD 160 hashes (we’ll come back to this later). 

The upside here is that while each of these layers need *some* degree of changes, the overall protocol hierarchy and flows are left mostly unchanged.

<div data-theme-toc="true"> </div>


## Signature+Public Key Size Comparison 

Before we jump into things, here's a handy chart that breaks down the signature+public key size differences across the various available offerings: 

| Primitive | Source | pk | sig (or ct) | **pk + sig** | NIST cat | Notes |
|---|---|---:|---:|---:|:---:|---|
| secp256k1 Schnorr/ECDSA | BIP-340 / DER | 33 | 64 | **97** | — | Today's classical baseline |
| ML-KEM-512 | FIPS 203 | 800 | (ct 768) | **1,568** | 1 | Cat-1 KEM |
| ML-KEM-768 | FIPS 203 | 1,184 | (ct 1,088) | **2,272** | 3 | "Hybrid TLS" default |
| ML-DSA-44 | FIPS 204 | 1,312 | 2,420 | **3,732** | 2 | Lattice signature |
| SLH-DSA-128-24 (reduced) | SP 800-230 ipd (Apr 2026) | 32 | 3,856 | **3,888** | 1 | 2²⁴ sig limit, w=4 |
| SLH-DSA-128s (full) | FIPS 205 | 32 | 7,856 | **7,888** | 1 | 2⁶⁴ sig limit |

`SLH-DSA-128-24` is a [newly proposed variant](https://csrc.nist.gov/pubs/sp/800/230/ipd) that achieves a smaller signature size in exchange for a hard cap on the number of signatures per signing key (2²⁴ instead of 2⁶⁴ for the standard `SLH-DSA-128s`).

## One Key No Longer Does the Job

One thing that will be sorely missed about ECC is how universal it is. The very same ECC group parameters can be used for both digital signatures and non-interactive key exchange, along with a whole suite of other interesting cryptography protocols (eg: musig2, adapter signatures, PTLCs, silent payments, etc, etc). 

Unfortunately, in Post Quantum land, the same truth doesn’t hold, as primitives like hash chains/trees and noisy lattices don’t inherit the mathematical symmetry and structure that EC groups give us. 

One implication of this is that there isn’t any one post-quantum cryptosystem that can give us everything we need in terms of functionality. As a result, it’s likely the case that we’ll end up shipping not one, not two, but likely three different cryptosystems to just achieve even the *base line* functionality that we have today with ECC (ML-KEM for transport, ML-DSA for off-chain signatures, and SLH-DSA for on-chain signatures). 

We’ll get more into this later, but let’s say we wanted to go with purely lattice based crypto. We need to be able to do some sort of key/secret exchange (a la ECDH), and also generate digital signatures over data to be authenticated. In ECC land, a single key is able to satisfy both these requirements: we use the same key for the static DH in the Noise protocol, as we do to sign our gossip announcements. 

In NIST PQ land none of the accepted schemes support such generality. Instead for a DH-like construct, the only option is ML-KEM (the Module Lattice Key Encapsulation Mechanism). ECDH is a very elegant KEM, as a secret can be derived in a purely non-interactive manner. ML-KEM on the other hand is effectively a hybrid encryption protocol, where an initiator encrypts a random secret to the public key of a responder. If we’re sticking with lattices, then for signatures the main choice is ML-DSA  (Module-Lattice-Based Digital Signature Algorithm). 

You might think that since they’re both lattice-based algorithms, the public/private keys would be the same…nope! Both schemes are based on module lattices over polynomial rings, but aside from that nearly all the other parameters/primitives differ. 


## To Hybridize or Not to Hybridize? 

Instead of switching over whole sale to PQC, we can actually hedge a bit instead. Hybrid Post Quantum cryptography combines classical and post quantum cryptography in a way that if either of the schemes are secure the final scheme is still secure. As after all, maybe the PQ schemes are the ones that are broken in the future. Alternatively perhaps the classical schemes fall not due to a quantum computer, but some other cryptanalysis innovation.

As we’ll see below, other than for onion routing, the other areas can be hybridized in a gradual manner with new TLVs or existing version extension fields. We could add these hybrid keys as optional TLVs today, validate them as normal, then only in the future would we *reject* messages that didn’t include the PQ capsules/signatures. 

### Hybrid KEM

For KEMs, there’s extensive [literature](https://eprint.iacr.org/2018/024) that shows how to [safely combine a classic and PQ KEM](https://eprint.iacr.org/2020/1364) to derive a new hybrid shared secret. The technique of composing a classic+PQ KEM is commonly referred to as a Combiner, or KEM Combiner. There are many ways to achieve this in practice, depending on the desired security profile. I won’t go into all the options in this post, but if you’re curious, see my recent post on the bitcoin dev ML about design options for a PQ BIP 324 variant. 

Space/bandwidth wise, the TL;DR here is that you need to transmit both the classic+PQ public keys *and* the capsule (the encrypted shared secret) to establish a shared secret. From the PoV of PQ crypto, classical crypto is very compact, so we don’t really lose much by *also* transmitting the classic keys+capsules. 

### Hybrid Digital Signatures

For hybrid digital signatures, we don’t need any fancy composition techniques. Instead you just include both the classical and PQ signature. Once again, the classical signatures are tiny compared to the PQ signatures so there isn’t a huge hit in terms of space/bandwidth. 

For a PQ KEM, there’s only one real option. However for signatures, we have hash based (SLH-DSA) and lattice based (ML-DSA). Before doing the requisite research to write this up, I was under the impression that we’d just go with SLH-DSA everywhere as hashes are simple. However after doing some analysis, I think ML-DSA is actually better suited to certain layers than SLH-DSA. Hence my comment above that we may actually end up deploying three different schemes… 

With this background out of the way, let’s go layer by layer, analyzing the design requirements+constraints to see how the BOLTs would need to change to become PQ. 


## Presentation Layer: BOLT 11 + 12

At the very “top” of the protocol stack, we have BOLT 11 + 12. One can view this through the OSI lens as the “presentation” layer. I’ll focus on BOLT 11 here for simplicity as we’ll return back to some of the BOLT 12 mechanics (a la Offers + Onion Messaging) when we visit BOLT 4. 

### BOLT 11 Classical Crypto

BOLT 11 invoices are signed using ECDSA, leveraging ECDSA recovery to avoid having to place the public key directly in the invoice. SHA2 is used to commit to a description hash, and also for the payment hash/secrets. 

### Proposed BOLT 11 Post Quantum Crypto

For signature choices, without reaching for anything exotic, we have either ML-DSA (lattice based), or SLH-DSA (hash based, SPHINCS+).

Today we use public key recovery to avoid having to include the public key in the invoice itself. This saves us 33 bytes. For post quantum signatures, in addition to the signatures being much larger, for most schemes, the public keys are also larger on a relative basis. 

This increase in the size of sigs+keys with PQC presents some challenges for the BOLT 11 encoding scheme. As BOLT 11 took bech32 rather literally, the max field length is: 1023 × 5 bits = 5115 bits = 639.375 bytes. Even with the smallest params, the sigs of both ML-DSA and SLH-DSA exceed that limit. As a result, if BOLT 11 is to be kept alive, the format also needs an overhaul. 

#### SLH-DSA vs ML-DSA

**[u]BOLT 11: SLH-DSA[/u]** 

SLH-DSA actually supports a type of public key recovery. As a signature is more or less a series of hash chain and merkle tree openings, if you hash everything together, you arrive at a candidate root. You can then combine that candidate root along with a public key seed to derive the actual public key. For SLH-DSA, if we’re focusing on the recently published [SLH-DSA-128-24 variant](https://csrc.nist.gov/pubs/sp/800/230/ipd) (upper limit of 2^24 signatures, more on that later), then public keys are 32/64 bytes, while signatures are 3.8 KB. With these numbers, including the key outright vs recovering via the signature doesn’t really add much. 

SLH-DSA-128-24 does indeed give us much smaller signatures (competitive with ML-DSA), but it imposes a 2^24 signature limit, which is around 16 million. Compared to the standard SLH-DSA-128s with its ~8 KB signatures, the -24 variant gives some sizable savings. 

Depending on the scale of a user/merchant/node/businesses, 16 million signatures might actually not be that much. If we assume a popular store/site, or even an exchange that generates 1k invoices an hour, then within two years they’ll have entirely run through the 16 million signature limit!

As a result, **if hash based signatures are desired at this level, then we may need to eat the 8 KB signatures and go with vanilla SLH-DSA-128s**. 

**[u]BOLT 11: ML-DSA[/u]**

If we look at ML-DSA, focusing in ML-DSA-44, we get public key sizes of 1.3 KB and signature sizes of 2.4 KB (the private key itself is 2.5KB). We don’t deal with any signature count limits like SLH-DSA-128-24, but the tradeoff is greater implementation complexity. The tradeoff is that we end up having around 4 KB of just signature+key information. 

### Payment Request QR Code Constraints

For QR codes, [the max amount of bytes that can be effectively encoded is 2953 bytes in bytes mode](https://www.qrcode.com/en/about/version.html). Looking at the signature+key sizes mentioned above, then this means that none of the PQ schemes can feasibly be encoded in QR codes the way we do today. Instead another format would need to be selected, or several QRs are stitched together like a mini-movie to transmit the additional data.

## Transport Layer: BOLT 8

Next, let’s check out the transport layer. Today we use the Noise protocol handshake, which leverages a novel triple DH handshake to mix in a series of static and ephemeral keys into a final shared secret. We use Noise_XK which also gives us a degree of information hiding (initiator must know the identity of the responder). 

As mentioned in PQ land we can’t use the same slick series of DH operations, as we only have a KEM, which is effectively a secret encrypted with a static long term key. 

One implication of this is that we now need to advertise an additional KEM key, as the same key can’t be used for verifying signatures/invoices *and* also setting up the encrypted connection. 

### PQ Noise

Thankfully we already have a ready to go scheme for [PQ Noise](https://eprint.iacr.org/2022/539). The gist is that you transform the existing `ee` (ephemeral+ephemeral), `es` (ephemeral+static), and `ss` (static+static) operations with appropriate KEM operations. The one larger shift is that the static exchanges, and the `es` exchange from the responder, now also need to carry along a key confirmation message. Overall the flow is mostly the same, but there are extra messages and we carry more information per handshake. 

If we’re ok with going with a pure PQ noise, then we’re mostly done here as we can use an off the shelf solution. However if we want to hybridize this layer, then we need to make some additional design decisions. 

### Hybrid PQ Noise

For a more detailed break down on Hybrid KEM protocols, see my recent post to the Bitcoin Dev ML. 

Broadly we have two options here:

1. Upgrade to the PQ KEM handshake *inside* of our existing classical Noise handshake.
2. Use a non-Noise Hybrid Combiner Handshake. 

Both options require us to advertise a new PQ KEM key. 

The first option allows us to leverage all of the existing Noise code we have, sprinkling in a PQ KEM like ML-KEM-768 after the initial handshake is set up. We’d then mix in that new PQ derived shared secret into the existing session state, and rekey. We already have clean domain separation between the handshake transport and application data, so we wouldn’t risk transmitting sensitive information with classical security. 

The second option is arguably more elegant, as we’d do the classical+KEM in a single pass. However we’d either need to ditch Noise altogether, or compose the normal and PQ noise handshake patterns into a new combined handshake. 

## Discovery Layer: BOLT 7

Today with the wonders of ECC, all the gossip messages are only a few hundred bytes max. In PQ land, we continue the mantra of: "everything is bigger with PQC"! 

### Gossip Authentication: ML-DSA vs SLH-DSA

Today the node ID key signs: invoices, `node_announcement`, `channel_announcement`, and `channel_update`. NIST advises a hard upper limit of 16 million signed messages. 

A quick napkin sketch above showed that if a node signs 1k invoice an hour, then they run out within 2 years. The situation is a bit more dire if we factor in `channel_update` messages, and also `channel_announcement`. For simplicity, we'll focus on `channel_update` as it's the spammiest message in the network today. 

If a node with 1k channels signs a new channel update for each of them every 10 minutes (eg: gossip v2 may use block height based rate limiting), then they'll exceed the 2^24 limit within **4 months**: 

| N (minutes) | sigs/day | sigs/year | **Time to exhaust 2²⁴** |
|---:|---:|---:|---|
| 1 | 1,440,000 | 525,600,000 | **11.6 days** |
| 5 | 288,000 | 105,120,000 | **58.3 days** |
| 10 | 144,000 | 52,560,000 | **116.5 days** (~3.8 mo) |
| 15 | 96,000 | 35,040,000 | **174.8 days** (~5.7 mo) |
| 30 | 48,000 | 17,520,000 | **349.5 days** (~11.5 mo) |
| 60 | 24,000 | 8,760,000 | **1.92 years** |
| 360 (6h) | 4,000 | 1,460,000 | **11.5 years** |
| 1,440 (24h) | 1,000 | 365,000 | **46 years** |



This is a relatively conservative estimate, as there are nodes on the network with more than 1k channels, and it doesn't factor in the rate of churn of new channels either. Therefore, I don't think the reduced-use SLH-DSA (SLH-DSA-128-24) is a serious candidate here, unfortunately. 

Without `SLH-DSA-128-24`, then ML-DSA signatures are much smaller as mentioned above. Functionality wise, both signature types give us what we need (no fancy musig2 multi-signatures for now). 

For the remaining sections, I'll assume that we're going full PQ for size comparisons, as the classical EC signatures are dwarfed by the size of their PQ counterparts (a 64-byte Schnorr sig is ~3% of a 2,420-byte ML-DSA-44 sig), so it's just a rounding error if we go with a hybrid deployment. 

### `node_announcement`

We'll start with the node announcement, as it contains the initial set of authenticated keys that are required to be able to connect to a node, and also verify any of its announcements. 

Today we're lucky to be able to send out just a single EC pub key in the node announcement, along with some other feature/address information. In PQ land, as mentioned we'll need individual keys for the encrypted p2p transport, _and_ an authentication key for the signed gossip announcements. 

An ML-KEM-768 public key is 1.1 KB, that's the only option for the new version of BOLT 8, so we're stuck with that. This adds a _new_ key that needs to be advertised in the `node_announcement` message. 

However for the node identity key (signs the updates), we once again have the question of which signature algorithm to contend with. Based on the line of reasoning in the earlier section, the 3 KB SLH-DSA signature type may not be viable for most node profiles.

For completeness, the following table includes a head to head size comparison with the various possible combinations (various ML-KEM param configs, then either ML-DSH/SLH-DSA/SLH-DSA-128s): 

| Variant | sig | auth_pk | kem_pk | other | **Total** | vs. 140 B |
|---|---:|---:|---:|---:|---:|---:|
| Classical | 64 | 33 | — | 43 | **140** | 1× |
| ML-KEM-512 + ML-DSA-44 | 2,420 | 1,312 | 800 | 43 | **4,575** | ~33× |
| ML-KEM-768 + ML-DSA-44 | 2,420 | 1,312 | 1,184 | 43 | **4,959** | ~35× |
| ML-KEM-512 + SLH-DSA-128-24 | 3,856 | 32 | 800 | 43 | **4,731** | ~34× |
| ML-KEM-768 + SLH-DSA-128-24 | 3,856 | 32 | 1,184 | 43 | **5,115** | ~37× |
| ML-KEM-512 + SLH-DSA-128s | 7,856 | 32 | 800 | 43 | **8,731** | ~62× |
| ML-KEM-768 + SLH-DSA-128s | 7,856 | 32 | 1,184 | 43 | **9,115** | ~65× |

Each row has an ML-KEM entry as that's required for the new BOLT 8 encrypted transport. The `-24` row for SLH-DSA is the new variant that's limited to 2^24 signatures. 

### `channel_announcement`

Next up, we have the channel announcement. Today this contains 4 public keys, and 4 signatures. With Taproot gossip we can eliminate the 3 signatures and instead only transmit a single aggregated signature. In PQ land, we don't have such nice techniques (yet!). 


Given the amount of public keys and signatures, we really start to see the size blow up for this message compared to the `node_announcement` message: 

| Auth scheme | node_sig×2 | node_pk×2 | btc_sig×2 | btc_pk×2 | other | **Total** | vs. 430 B |
|---|---:|---:|---:|---:|---:|---:|---:|
| Classical | 128 | 66 | 128 | 66 | 42 | **430** | 1× |
| ML-DSA-44 (both sides) | 4,840 | 2,624 | 4,840 | 2,624 | 42 | **14,970** | ~35× |
| SLH-DSA-128-24 (both sides) | 7,712 | 64 | 7,712 | 64 | 42 | **15,594** | ~36× |
| SLH-DSA-128s (both sides) | 15,712 | 64 | 15,712 | 64 | 42 | **31,594** | ~73× |
| **Mixed: LN = ML-DSA-44, BTC = SLH-DSA-128-24** | 4,840 | 2,624 | 7,712 | 64 | 42 | **15,282** | ~36× |
| **Mixed: LN = ML-DSA-44, BTC = SLH-DSA-128s** | 4,840 | 2,624 | 15,712 | 64 | 42 | **23,282** | ~54× |

The bottom two rows assume a future where Bitcoin soft forks to include either SLH-DSA variant, which results in a mixed signature/pubkey pairing for the channel announcement message. 

I'll skip `announcement_signatures` for now as that isn't gossiped to the wider network (doesn't contribute to gossip bandwidth the same way these other messages do). As you can guess from the above table, the new message ends up being 30-90× bigger than the old message. 

#### `zk_channel_announcement` for Signature Aggregation 


In terms of pubkey/signature compression, one option here is a STARK proof in place of plain signatures. The STARK can prove that a given hash is actually a commitment to N valid signatures of a valid `channel_announcement` sighash, etc, etc. Such a proof can even go a step further by including a block hash or block header in the announcement, and asserting that: there exists a transaction, whose `pkScript` is a valid LN funding output using the committed keys, etc, etc. One could even take yet another step and gossip an LN channel state root, alongside an incrementally batched+aggregated STARK proof that all channels under that root are valid. Then a `channel_announcement` just references that state root, and includes a merkle tree inclusion proof for that root. 
 

### `channel_update` 

Wrapping up this section, we repeat the above analysis, but for `channel_update` messages. Even a conservative estimate for signature usage (medium sized node/merchant/exchange) shows that the reduced signature count SLH-DSA variant isn't suitable for this use case. However for completeness, I've still included it in this table. 

| Auth scheme | sig | payload | **Total** | vs. 136 B |
|---|---:|---:|---:|---:|
| Classical | 64 | 72 | **136** | 1× |
| ML-DSA-44 | 2,420 | 72 | **2,492** | ~18× |
| SLH-DSA-128-24 | 3,856 | 72 | **3,928** | ~29× |
| SLH-DSA-128s (full) | 7,856 | 72 | **7,928** | ~58× |


### PQ Aggregate Message Size + P2P Bandwidth

To wrap up this section, here's a table that shows what the aggregate message sizes would be across all announcements with various signature size combos. The "Mix" columns assume a future where Bitcoin soft-forks in SLH-DSA on the consensus side while Lightning uses ML-DSA-44 off-chain. 

| Message | Classical | ML-DSA-44 (both) | SLH-DSA-128-24 (both) | SLH-DSA-128s (both) | **Mix: LN ML-DSA-44 / BTC SLH-DSA-128-24** | **Mix: LN ML-DSA-44 / BTC SLH-DSA-128s** |
|---|---:|---:|---:|---:|---:|---:|
| `node_announcement` (×1) | 140 | 4,959 | 5,115 | 9,115 | 4,959 | 4,959 |
| `channel_announcement` (×1) | 430 | 14,970 | 15,594 | 31,594 | 15,282 | 23,282 |
| `announcement_signatures` (×2) | 336 | 9,760 | 15,504 | 31,504 | 12,632 | 20,632 |
| `channel_update` (×2) | 272 | 4,984 | 7,856 | 15,856 | 4,984 | 4,984 |
| **Total** | **1,178** | **34,673** | **44,069** | **88,069** | **37,857** | **53,857** |
| **vs. classical** | 1× | ~29× | ~37× | ~75× | ~32× | ~46× |


The most "conservative" option (SLH-DSA-128 everywhere) is pretty much double the size of all the other options. If we look at size alone, then ML-DSA-44 is the clear winner. 

#### PQ P2P Bandwidth

Across all the signature types, we now end up transmitting an order of magnitude more bandwidth in aggregate for gossip messages. This has some predictable effects on total bandwidth usage across the entire network. 

The following table assumes we have 20k public channels, that each send `R` channel updates a day:

`bytes/day = 20,000 × R × msg_size`

| R (msgs/channel/day) | Total msgs/day | Classical (136 B) | ML-DSA-44 (2,492 B) | SLH-DSA-128-24 (3,928 B) | SLH-DSA-128s (7,928 B) |
|---:|---:|---:|---:|---:|---:|
| 1 (one dir, daily) | 20,000 | 2.72 MB | 49.84 MB | 78.56 MB | 158.56 MB |
| 2 (both dirs, daily) | 40,000 | 5.44 MB | 99.68 MB | 157.12 MB | 317.12 MB |
| 10 | 200,000 | 27.2 MB | 498.4 MB | 785.6 MB | 1.59 GB |
| 50 | 1,000,000 | 136 MB | 2.49 GB | 3.93 GB | 7.93 GB |
| 100 | 2,000,000 | 272 MB | 4.98 GB | 7.86 GB | 15.86 GB |
| 288 (1/5min/dir) | 5,760,000 | 783 MB | 14.35 GB | 22.62 GB | 45.66 GB |
| 1,440 (1/min/dir) | 28,800,000 | 3.91 GB | 71.78 GB | 113.13 GB | 228.32 GB |

Note that this is ingress, so what a gossip node that processes all gossip would download in a given day. This also assumes they only receive a single set of updates from a given node. In the real network, assuming no better gossiping protocol, this number gets multiplied by the gossip flooding factor (5–10×, depending on peer count and dedup efficiency) since each message is forwarded to peers redundantly. 

We can also extrapolate further and compute the expected bandwidth _per year_
| R | Classical | ML-DSA-44 | SLH-DSA-128-24 | SLH-DSA-128s |
|---:|---:|---:|---:|---:|
| 1 | 0.99 GB | 18.20 GB | 28.69 GB | 57.91 GB |
| 2 | 1.99 GB | 36.40 GB | 57.39 GB | 115.82 GB |
| 10 | 9.93 GB | 182.0 GB | 286.9 GB | 579.1 GB |
| 50 | 49.6 GB | 909.9 GB | 1.43 TB | 2.90 TB |
| 100 | 99.3 GB | 1.82 TB | 2.87 TB | 5.79 TB |
| 288 | 286 GB | 5.24 TB | 8.26 TB | 16.67 TB |
| 1,440 | 1.43 TB | 26.20 TB | 41.30 TB | 83.34 TB |


## Onion Layer: BOLT 4

Today we use the Sphinx mixnet packet format in BOLT 4. It uses a repeated blinding trick to generate a relatively compact final mixnet packet. Rather than include a new ECDH key for each hop to derive the shared onion/session secret, it includes one key in the packet, then instructs each hop to tweak the existing key in a deterministic way to generate a key for the next hop. If such a technique wasn't possible, then the packet would be much larger as you'd need to include a new ECDH for each hop, up to a max of 20 hops (the packet is always fixed sized). 

### PQ Sphinx

Ignoring [Isogenies](https://eprint.iacr.org/2021/633), ML-KEM doesn't support such a commutative operation, so we're forced to go back to the old format that includes the information to derive the shared secret with each hop. For now we'll assume that the same ML-KEM key is used for BOLT 8 encrypted transport creation, and also the onion layer. 

Thankfully, [David Stainton has mostly worked everything out for us](https://eprint.iacr.org/2023/1960). Packet processing+generation is more or less unchanged. The only difference is that rather than sending that nice 33 byte ECDH key, instead we need to send the 1 KB ML-KEM capsule to each hop. They can then decapsulate to derive the shared secret key, and process the packet as normal. 

As you can imagine, this makes the onion packets *much* larger. 

The overhead per hop increases by a factor of 10-20x (per-hop overhead here includes the MAC): 

| Variant | per-hop (legacy 65) | per-hop (TLV ~132) | classical equivalent |
|---|---:|---:|---:|
| Classical Sphinx (today) | 65 | 132 | 1× |
| KEM Sphinx, ML-KEM-512  |   833 |   900 | ~13× / ~7× |
| KEM Sphinx, ML-KEM-768  | 1,153 | 1,220 | ~18× / ~9× |
| KEM Sphinx, ML-KEM-1024 | 1,633 | 1,700 | ~25× / ~13× |


If we assume around 132 bytes of max payload space per hop, then we arrive at the following size comparison: 

| max `N` | Classical | ML-KEM-512 | ML-KEM-768 | ML-KEM-1024 |
|---:|---:|---:|---:|---:|
| 4  | 1,366 |  3,633 |  4,913 |  6,833 |
| 6  | 1,366 |  5,433 |  7,353 | 10,233 |
| 8  | 1,366 |  7,233 |  9,793 | 13,633 |
| 10 | 1,366 |  9,033 | 12,233 | 17,033 |
| 15 | 1,366 | 13,533 | 18,333 | 25,533 |
| 20 | 1,366 | 18,033 | 24,433 | 34,033 |

Notice how for classical, we can stay at the same size no matter the number of hops. This is because we only ever include a _single_ ECDH key as we can use the blinding trick. For ML-KEM however, we need to include a capsule for _each_ hop. So for a realistic view, you can just look at the 20-hop limit. 

One way to mitigate the size impact here is to actually _reduce_ the max number of hops possible with a packet. 20 in practice is extremely large (the network's median path length is 4-6 hops), and I'm not sure anyone has ever created an organic 20-hop route in the wild with vanilla software. 

These sizes also have direct impact on blinded paths and trampoline. Either the max hop size for them is rather small (as they eat into the size of the outer onion), or the broader onion becomes even larger. 

### `update_add_htlc` Impact

As the onion packet goes in the `update_add_htlc` message, the total size of the message also goes up. Thankfully, we're still within the 65 KB message limit set by BOLT 1: 

| Path / KEM | onion (legacy 65) | `update_add_htlc` total |
|---|---:|---:|
| Classical, current | 1,366 | 1,450 |
| 8-hop, ML-KEM-768 | 9,257 | 9,341 (~6.4×) |
| 20-hop, ML-KEM-512 | 16,693 | 16,777 (~11.6×) |
| 20-hop, ML-KEM-768 | 23,093 | 23,177 (~16×) |
| 20-hop, ML-KEM-1024 | 32,693 | 32,777 (~22.6×) |


Thankfully nothing needs to change w.r.t onion errors, as they rely on the existing per-hop shared secret. 

## Channel Layer

On the channel layer, the main determinant is what signature type Bitcoin initially soft forks in. You can plug in any of the signature sizes above to get a feel of how large the new transaction witnesses will be. When it comes to `pkScript` size, SLH-DSA shines, as the public keys are only 32/64 bytes. A new channel type can mechanically replace `OP_CHECKSIG` with `OP_CHECKPQSIG` (w/e the sig type is), then swap out the public key/signatures and be mostly good to go. 

In terms of more advanced channel types, one drawback in PQ land is that there's no analogue for PTLCs without reaching for more heavy weight succinct proofs. 

Depending on what other capabilities are available in Bitcoin Script by the time we actually need PQ signatures, we could even switch to an entirely new more PQ-native channel type. 


### Payment Hash: RIPEMD-160 → SHA-256

A quantum attacker with Grover's algorithm can find the preimage for a RIPEMD-160 hash with 2^80 work vs 2^128 against SHA-256. With the BHT algorithm, a quantum attacker can find a collision with 2^53 work on RIPEMD-160. Today we use RIPEMD-160 (a 160 bit hash) in the script instead of plain sha2 to save a few bytes. However in quantum land everything is massive, so it makes no sense to play script golf for a few bytes any longer. RIPEMD-160 is basically legacy crypto, so we should just use plain sha2 instead for the payment hash in the script.

-------------------------

