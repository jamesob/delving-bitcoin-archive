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

