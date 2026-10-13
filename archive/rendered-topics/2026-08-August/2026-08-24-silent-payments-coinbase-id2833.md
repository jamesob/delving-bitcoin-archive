# Silent Payments coinbase

marathon-gary | 2026-08-24 19:47:45 UTC | #1

I've been wondering how a mining pool might pay out miners to different addresses directly in the coinbase (and generally about "fun" things to do with a coinbase tx). Ocean pool currently pays miners in the coinbase^[Miners with enough hashrate to be above a threshold get paid in the coinbase.] but only to a static address set as the stratum username.

After [experimenting](https://average-gary.github.io/sv2-apps/) with miniscript wildcard descriptors to rotate the pool's coinbase address, I initially thought that having a miner provide an xpub or similar to the pool would work. However, any compromise of the pool's database means that all transactions from that xpub would be revealed. Not ideal if your aim is privacy.

It recently occurred to me that [BIP352](https://bips.dev/352/) Silent Payments (SP) could be used, since a receiver (miner in this case) has a static public SP address. The rest of this post outlines how I believe this could be done within a coinbase transaction for paying hash providers (aka "miners" in this text). I will refer to this as "Silent Payments coinbase" (SPc). 

For SP receivers, a silent payment address is composed of two keys, the scan key and the spend key. For SPc, this remains unchanged. The receiver would send their SP address via the Stratum v2 protocol^[Stratum v1 is intentionally left out as a consideration because it lacks native encryption to the spec and has no formal definition. Sv2 uses BIP324 style noise for the wire protocol, which I deemed necessary for private communication between miner and pool. Full bias disclosure: I have contributed to the Sv2 reference implementation and protocol.] upon opening a [Mining Protocol](https://stratumprotocol.org/specification/05-mining-protocol/) Channel to the pool. The pool would register this in its accounting database to begin crediting hashrate to the SP address. 

Pool accounting methods are orthogonal, but I believe the pool should scrub share accounting logs after they are no longer relevant. It is an unavoidable necessity for pools to track correlation between SP address and amount owed. There is a threat vector to users privacy if the pool's database is compromised so any implementation should retain share accounting data only so long as it is needed to make payments. I see this akin to the same privacy trust trade-off as VPNs. The VPN provider (pool) is a trusted party in your privacy, which means operational best practices should be required, as deletion is not provable.

Now to the fun part!

When a normal SP sender creates a transaction, they need to reveal a pubkey and nonce. These allow receivers to do [DHKE](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange) maths to discover which address is theirs and spend it.  This is somewhat of an oversimplification that I encourage the reader to digest in the ["Sender" section](https://github.com/bitcoin/bips/blob/7fe0b034ec967b52a5a28276419117326df93263/bip-0352.mediawiki#sender) of the BIP352 spec. 

In the original specification, the pubkey/nonce combination is derived from deterministic data within the input(s), including the spending key of the sender. A coinbase transaction has no inputs^[TECHNICALLY, there is 1 coinbase input, which we still utilize, just differently. This input doesn't have a pubkey either.] so standard SP won't work. In SPc, the nonce is the block height, and is used in a commitment with the pool's pubkey (`hash(block_height ‖ A_send)`). By committing to A_send with the height, it prevents the pool from grinding a malicious `a_send` (see [footnote 3 of BIP352 why_include_A](https://github.com/bitcoin/bips/blob/7fe0b034ec967b52a5a28276419117326df93263/bip-0352.mediawiki#rationale-and-references)).

The sending pubkey is then published in the coinbase pool tag (aka pool signature). There are 100 bytes for the coinbase script sig which include some required bytes ([BIP34](https://bips.dev/34/)) but plenty of headroom for a 34 byte `A_send` to replace the ASCII pool tag, provided they don't overlap the miner rollable `extranonce` (which is pool defined in Sv2). The `A_send` key the pool uses for the DHKE ideally is ephemeral so compromise of the key cannot leak secrecy data of those paid. In ordinary SP that key is the sender's spending key, so it can never be discarded. In SPc it controls no money at all, which is a different posture when considering compromise.

This simple rethinking of the specification for coinbase payouts now introduces some dynamic requirements for a pool looking to adopt or use SPc. Luckily we now have a well defined and specified mining protocol in Stratum v2 so defining/coding this becomes easy (famous last words!).

As a SPc pool receives Mining Protocol Channel establishment and has a SP address shared for a given hashrate provider, it will use that SP address as the accounting key in its database. As the pool accounts and updates its coinbase payout distributions, it would simultaneously conduct the necessary maths to create the SPc transaction that it would send to all connected hashrate providers. Given the online/interactive requirement of mining, this does not add additional complexity to hashrate providers by allowing the pool to absorb that cost of computation, which is what pools are meant to do. With a proper implementation, a hashrate provider would simply update their username to a SP address and point their hash to a SPc enabled endpoint.

As with normal SP, a receiver uses the publicized (in pool tag) `A_send` in combination with their scan key to find which address pays them. Luckily, because SPc transaction is always the `0`th transaction in every SPc block, the scanning burden is reduced. A modest win, but worth noting since I believe it addresses an [open question for light clients (5)](https://github.com/bitcoin/bips/blob/7fe0b034ec967b52a5a28276419117326df93263/bip-0352.mediawiki#rationale-and-references) in BIP352.

Some considerations/threats:

* SPc does not mitigate amount fingerprinting. Steady/consistent hashrate for a given pool could make miners identifiable across blocks the pool finds. There are likely amount mitigations but I'll leave those for the comments and future ideation.
* Compromise of the mining pool database is likely the largest threat vector to privacy for those paid. Ideally, the pool only retains share accounting data for a duration necessary to calculate and payout hashrate providers then nukes the accounting data, helping secure the privacy of past blocks.
* As Ocean has demonstrated recently, there are limits to the number of payouts in a given coinbase tx to avoid dust amounts. A SPc pool would likely want to adopt [Ocean-style BOLT12 LN payments](https://ocean.xyz/docs/lightning) to allow for smaller hashrate providers to exit anonymously. For SPc, we assume that not all hashrate providers will be paid out in a given coinbase due to these limitations.
* There is a [Sv2 spec extension proposed for non-custodial payouts](https://github.com/stratum-mining/sv2-spec/pull/203) which would be a needed for SPc to support JobDeclaration, where hashrate providers are full miners crafting block templates, but deferring to the pool for what the coinbase transaction looks like. This spec proposal leaves out the pool tag requirements for SPc but otherwise seems to be useful. 

If you'd like to review the AI (claude) generated webpage that I prompted while researching this idea, you can find it [hosted on GitHub](https://average-gary.github.io/sp-coinbase-vectors/). It is within a repository containing test vectors for SPc and other iterations of the idea that the AI helped me create.

Comments/questions/critiques welcome and encouraged to improve the idea so it can mature into a BIP/specification. There is definitely some due diligence and more thought needed for this proposal but I wanted to share to help with that process. Please share with someone who may be inspired or intrigued. It is my belief that Stratum v2 maturation is leading to a robust and vibrant mining software ecosystem, and I hope this idea brings more brain juice to the ecosystem.

Thank you. 

Blessings and Peace be upon you.

-------------------------

Anzus_GemWallet | 2026-08-28 14:11:36 UTC | #2

From a wallet user’s point of view, would a wallet that already supports Silent Payments find these payouts automatically, or would it need a separate update? A short compatibility section could help miners understand whether they need a special wallet before using this.

-------------------------

Seldenthuis | 2026-08-29 18:24:49 UTC | #3

One number worth double-checking: the BIP34 height push isn't a small fixed cost.

I've been implementing coinbase scriptSig construction recently (BIP34 height encoding, the 2-100 byte consensus limit) and at current real-world heights (both mainnet and testnet4 are well past 2^16, under 2^24).
It already needs 4 bytes: a 1-byte push length plus 3 data bytes.

The minimal encoding can also need a 5th padding byte in some cases, whenever the top byte of the height's little-endian encoding happens to have its high bit set (avoids it being misread as a sign bit).

So the actual budget right now is closer to 100 - 4 (BIP34) - 34 (A_send) = 62 bytes for the extranonce, not "100 minus 34 with plenty of headroom left over.".
Still plenty of room today, but the BIP34 cost only grows from here (5 bytes once height passes 16,777,215), and it's not obvious to me whether that's a real constraint on extranonce sizing or just noise.

-------------------------

marathon-gary | 2026-08-31 13:49:06 UTC | #4

This would require a new scanning profile for wallets. The data that indicates ownership of your wallet is in a different location than standard silent payments.

-------------------------

marathon-gary | 2026-08-31 13:55:47 UTC | #5

I'm not sure the extranonce constraint is too problematic even with more bytes needed for height in the future, especially with [BIP323](https://github.com/bitcoin/bips/blob/2a783793f45d81558c2cecc7749f94226660ed54/bip-0323.mediawiki)

-------------------------

optout | 2026-09-24 12:45:17 UTC | #6

Very interesting idea!

Most miners use a custodial pool, where the pool controls the coinbase output, and they pay to themselves.

With the rise of Ocean, however, a non-negligible share of miners (~2%) receive their payout directly from the coinbase transaction. Here the pool still controls the payout amounts, it controls the coinbase outputs (both when controlling the whole block, or when allowing the miners to build their own block through Datum). This is implemented by using a bitcoin address as the single user idenitfier, and using that as a payout address.

The drawbacks are clear: address reuse, privacy leak by connecting subsequent payments to the same miner, ability to observe their spending. Silent payment would make a lot of sense.

Could you clarify which relevant components would be affected by this change? Anyone doing receive -- libraries, scanner services, wallets. As sending would be done exclusively by special mining-related software (regular wallets don't create coinbase transactions), there only the relevant mining software, or general-purpose silent payment libraries, if used.

How do you envision this change, as an amendment to the existing BIP-352 silent payment spec, or as a new BIP? Can it be a straighforward amendment of the spec, or are there some parts that need to be changed?

A note on solo pools. A very small amount of hash rate uses so called "solo pools" (small hash rate, but a relatively high number of individulal miners). Solo pools typically use the same payout-bitcoin-address-as-user-ID scheme, so the idea could be applied there as well. However, it would put some extra requirements on the solo miners, while address reuse is not a typical issue in this use case. Hiding the spending of an eventual block reward may be a worthy privacy benefit though.

(Note: I haven't ventured into the details of the proposed input/nonce solution, but it looks plausible at first glance.)

-------------------------

marathon-gary | 2026-10-13 13:18:21 UTC | #7

Pondered some more on this and found, perhaps, that you can use the same `sp1q…` address for two payouts. One above a threshold for inclusion in the coinbase and the other below that threshold using an Ark. 

The Ark destination is derived from the same per-block ECDH SPc already uses:

```plaintext
ecdh = H(height ‖ A_send) · a_send · B_scan   (pool)     = H(height ‖ A_send) · b_scan · A_send   (miner)
```

Domain-separated tweaks (BIP340 tagged hashes, BIP352 serialization):

* `t_onchain = H_SPc/TierSpc(ser_P(ecdh) ‖ ser_32(k))` → coinbase output `P = B_spend + t_onchain·G`
* `t_ark = H_SPc/TierArk(ser_P(ecdh) ‖ ser_32(k))` → Ark user pubkey `U = B_ark + t_ark·G`

Above threshold: SPc coinbase output. Below: pool accrues off-chain and arkoors to `U` once proceeds mature. Never reuse the same tweak across both (would leak `B_ark − B_spend`).

Stock Ark server needs no changes. Pool builds a payable bark `ark1` address from public `U` alone (blinded mailbox id is derived from the VTXO pubkey). One-time blinding hint the miner registers per server:

```plaintext
D = b_ark · S_mb
```

Pool then computes `blinded = M + D + t_ark·S_mb`. Server unblinds to exactly `M`.

Validated on unmodified captaind/regtest: server co-signed the arkoor to `U`, delivered to mailbox `M`. Modified miner client derived `u = b_ark + t_ark`, claimed the VTXO, proved ownership, and spent it back through the same server. Stock bark wallet rejects the VTXO (not owned by its seed); only the keystore needs extension. Pool wallet unchanged.

Privacy: the pool already sees both tiers. On-chain observers see only unlinkable SPc outputs (arkoors invisible). Static mailbox `M` lets the server link all arkoor payouts; fix is rotating `M_i` per pool block height (epoch) with on-demand one-shot reads (first at `i+100`, ≤5 reads/epoch). Variants: independent keys `m/352'/0'/0'/3'/i'` (per-epoch registration) or `M_i = M + H(ecdh_i)·G` (no registration, related keys). Residual: same IP re-links epochs → Tor stream isolation + jitter.

Client side: \~150–300 lines in bark so `pubkey_keypair` / `find_vtxo_keypairs` / etc. consult a pluggable extra-keys source and recognize tweaked keys. Bark currently ties every VTXO key to a seed index, so tweaked keys need their own type + DB storage; recovery re-scans pool blocks. After that, receive/spend/refresh/exit work unchanged.

Code, vectors, notes: https://github.com/average-gary/silent-pool (`spc_ark`, `spc_ark_mailbox`, `spc_ark_client`).

\*\*The text above was generated by a LLM but reviewed and edited by a human.

-------------------------

