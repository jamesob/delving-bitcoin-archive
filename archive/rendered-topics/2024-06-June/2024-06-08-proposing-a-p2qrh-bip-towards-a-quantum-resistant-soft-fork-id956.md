# Proposing a P2QRH BIP towards a quantum resistant soft fork

cryptoquick | 2024-06-08 21:11:13 UTC | #1

The motivation for this BIP is to provide a concrete proposal for adding quantum resistance to Bitcoin. We will need to pick a signature algorithm, implement it, and have it ready in event of quantum emergency. There will be time to adopt it. Importantly, this first step is a more substantive answer to those with concerns beyond, "quantum computers may pose a threat, but we likely don't have to worry about that for a long time". Bitcoin development and activation is slow, so it's important that those with low time preference start discussing this as a serious possibility sooner rather than later.

This is meant to be the first in a series of BIPs regarding a hypothetical "QuBit" soft fork. The BIP is intended to propose concrete solutions, even if they're early and incomplete, so that Bitcoin developers are aware of the existence of these solutions and their potential.

This is just a rough draft and not the finished BIP. I'd like to validate the approach and hear if I should continue working on it, whether serious changes are needed, or if this truly isn't a worthwhile endeavor right now.

The BIP can be found here:

https://github.com/cryptoquick/bips/blob/61aac1a91a596cdeed75ed7f5c5c19b0b4b923ef/bip-p2qrh.mediawiki

(Cross-posting from the Bitcoin Mailing list, for visibility)

-------------------------

cryptoquick | 2024-06-13 12:56:10 UTC | #2

This document has gone through several additional iterations since it was first posted. I'm not sure I can edit my original post, but I'll link to the latest version:
https://github.com/cryptoquick/bips/blob/e1fa27cb25deee8d58bdb9a64c112dfdf09e216d/bip-p2qrh.mediawiki

Here is a diff for those who read it before:
https://github.com/cryptoquick/bips/compare/61aac1a91a596cdeed75ed7f5c5c19b0b4b923ef...e1fa27cb25deee8d58bdb9a64c112dfdf09e216d

-------------------------

conduition | 2024-10-22 19:51:57 UTC | #3

Hi @cryptoquick. Thank you for pushing for quantum-resistance! This is an extremely important long-term problem and we need smart people thinking about it today.

However, we would be jumping the gun if we designed a soft-fork for quantum resistant addresses using today's post-quantum signing algorithms. It seems likely to me that when a large-scale quantum computer eventually comes about, we will likely know much more about post-quantum signature schemes than we do today, even if that day comes as soon as 5 years from now (unlikely).

I have an idea which I hope might help bitcoin users transition to post-quantum secure keys today, without any near-term consensus changes.

[I spent the last month surveying the history and current landscape of hash-based signature (HBS) algorithms](https://conduition.io/cryptography/quantum-hbs/). HBS algorithms are the most conservative but also the least practical option for post-quantum signatures. They're conservative because HBS relies on even fewer cryptographic assumptions than Schnorr signatures, but they're impractical because of large signature sizes (several kilobytes in most cases) and statefulness. Some algorithms are one-time signatures, so keys can only be used to safely sign a single message.

[At the end of the above article](https://conduition.io/cryptography/quantum-hbs/#Upgrading-Bitcoin), I describe a post-quantum upgrade plan I call "Digests as Secret Keys" (DASK). Instead of using BIP32 to derive a child key for a Bitcoin address, a DASK-supported wallet will do the following.

1. From a seed value (e.g. BIP39), derive a secret key for a hash-based signature algorithm.
2. Compute the corresponding HBS public key. If needed, hash the HBS public key into a 32-byte value (some HBS keys are already represented by a single hash).
3. Interpret the HBS public key (hash) as an secp256k1 secret key.
4. Compute the secp256k1 public key and use it for standard P2PKH/P2WPKH/P2TR spending

<img src="https://conduition.io/images/quantum-hbs/dask.svg">

When a viable QC is close at hand, the BItcoin network can activate a consensus rule change which disables ECDSA/Schnorr. Instead post-quantum bitcoin nodes would expect the spender to provide a signature using the _inner_ HBS key, rather than the _outer_ secp256k1 key.

For more flexibility, the inner key will probably be used to _endorse_ some other post-quantum signing key, which then signs the transaction using an algorithm which may not yet have been invented. As a result, the inner HBS key can use a one-time signature algorithm like [Winternitz OTS](https://conduition.io/cryptography/quantum-hbs/#Winternitz-One-Time-Signatures-WOTS), which has relatively small (1KB) signatures. 

The nice thing about the DASK approach is that we don't need any consensus or scaling changes today. We kick the can down the road, giving cryptographers more time to design secure and efficient post-quantum algorithms, while retaining fallback authentication in case of sudden quantum advancements.

Eventually yes, Bitcoin will need a first-class quantum-resistant address format for better scaling (HBS won't work at large scale I think). But we shouldn't standardize a PQ address format using one of today's cutting-edge PQC algorithms, because the odds are that it will not age well through decades of attack and optimization. 

Instead we should use a tried and proven algorithm like WOTS as an emergency fallback, and choose a more efficient primary PQ signing scheme later, when it's actually needed. The fallback HBS key format would need to be standardized today, but it would be only a client-side change with no consensus modifications needed until Q-Day.

-------------------------

cryptoquick | 2025-01-02 19:14:16 UTC | #4

I just looked through your article and have been doing some research and thought towards which algorithms we should develop with and include, and I tend to agree that we shouldn't rush into introducing what's available now into, as Adam Back puts it, [high assurance products](https://x.com/adam3us/status/1874877560427004190).

I'm not yet sure how I'll update BIP-360 to reflect this thinking, but I'm definitely flexible and open to different approaches. I like your idea of starting with WOTS, and I think we should choose one other algorithm for testing attestation disambiguation.

As for DASK, that looks really interesting. One modification I might suggest is to not ever deprecate secp256k1 keys, but instead, expect them to be included along with other signature types, an approach known as hybrid cryptography.

One question... In HBS, if signatures are validated in clients, how will nodes validate signatures for inclusion in a block? I would assume the same way clients do, except clients would have their own secrets.

Great work on your article btw, very well-researched.

Also, here's a link to the latest iteration of the BIP:
https://github.com/cryptoquick/bips/blob/0ae69db70a4a28f202d441b7131cd5b2169e7afe/bip-0360.mediawiki

Would you be interested in writing a BIP for DASK, and do you think it might be a good idea to narrow down BIP-360 to work with DASK?

-------------------------

conduition | 2025-01-11 22:13:01 UTC | #5

Thank you! I think some newer and better approaches than DASK have been proposed since i wrote this article and which would allow us to more easily implement these changes as a soft fork without any new address formats needed. Check out @MattCorallo's post [on the mailing list here](https://groups.google.com/g/bitcoindev/c/8O857bRSVV8).

Matt's soft-fork proposal is to disable key-spending on P2TR addresses, and define one of the `OP_SUCCESS` opcodes reserved by BIP342 to validate a post-quantum signature scheme in a taproot script-path spend branch.

This implies some additional fun twists, such as the fact that one must now carefully hide this `OP_SUCCESS` script branch until activation day, otherwise it could be spent by literally anybody. But in principle it seems much more achievable to me than DASK, so i'm wholeheartedly in favor. If we were going to write a BIP, i think Matt's idea would be the way to go. All power to you for writing that draft BIP but I think we should try to reuse P2TR if we can, so that we can get the space saving benefits of taproot for as long as possible.

He suggests using SPHINCS right off the bat for this, but i think a WOTS certification layer would be way better for future-proofing. Surely SPHINCS isn't the best we will ever have. We could also use WOTS or FORS to directly sign a transaction instead, which would be even more efficient but less flexible/safe. So there's still some kinks to iron out. 

The action steps i can see would be:
- Find consensus on a flavor of hash-based signatures (i'm partial to Compact WOTS+C personally)
- Determine whether to use it as a certification layer or to sign transactions directly
- Create a reference implementation of the signature scheme (specifically key-generation)
- Define the ground rules for how validation of the new opcode should work (following past BIPs as examples should help). New rules, such as the specific signature scheme to certify with WOTS, can be added later, since we're not planning to activate this upgrade for a long time.
- Write it all down in a BIP and start collecting feedback



A separate client-side BIP could be written to define deterministic key-gen of the HBS keypair, safe and future-proof taptree structure, etc.

I'm unfortunately tied up helping a full-time client at the moment and can't spare the time to pursue these things myself, but I'd be happy to review if this is something you're interested in.

-------------------------

0xfffffffa | 2025-02-10 15:37:09 UTC | #6

Hello, 

Its my first post here and have come across this very interesting work by @cryptoquick 

I've been working on https://www.bitcoinqs.org - its a L2 solution rather than a soft/hard fork of the L1 Bitcoin network. 

Behind the scenes, I've gone for ML-DSA/Dilithium. 

I've created a bridge between L1 / L2 BitcoinQS that bridges back and forth between L1 Bitcoin and L2 Quantum Wrapped Bitcoin (Quantum Safe) BQS at a 1:1 ratio ie same in same quantum wrapped out and vice versa.

I considered the BIP route as well, but given historically BIPS take an incredibly long time to get implemented and supported I thought this would be the fastest way to get a working solution ready. 

High level, signature algo ML-DSA as previously stated. At present while on testnet the  address generation uses SHA3 256 and base58 encoding. However I'm in the process of upgrading this in time for mainnet launch.

Key generation , signing all happen entirely in the browser. Console tools are being developed to perform same from the command line. The platform is non custodial and at no point do we have access to user keys. 

Once transaction is signed in the browser, it is broadcast as expected to various validators that confirm said signature and update their state.

A block size in testnet is defined as 2 transactions once 2 transactions happen on L2 they are hashed and posted to L1 Bitcoin using OP_RETURN referencing the previous hash (first genesis hash is simply the word GENESIS followed by the hash of the transaction). 

On mainnet, I'm still rather undecided what the optimal block size would be as need to find the golden ratio between security and keeping transaction costs low (possibly 20) - ultimately this will depend on adoption and # of txs per hour. 

Given its L2, can implement all sorts of wonderful extensions such as quantum resistant BRC20 tokens. We can implement quantum safe smart contracts and others. 

there is no limit to what the imagination can conceive and extend the base system to support. 

We are thinking along the same lines and it would be nice to collaborate feel free to reach out on twitter @bitcoinqs

-------------------------

sipa | 2025-02-10 18:01:14 UTC | #7

[quote="0xfffffffa, post:6, topic:956"]
its a L2 solution rather than a soft/hard fork of the L1 Bitcoin network.
[/quote]

What's the point?

In the hypothetical future people are concerned about, L1 UTXOs are vulnerable to theft. The currency cannot maintain value in that setting, regardless of what other technologies are additionally used to represent tokens pegged to it. Instead of adding security, this is weakening it: now there are potentially two protocol stacks which could be broken that threaten security of the system. Remember, PQ algorithms can still be broken, possibly even classically; this has even happened fairly [recently](https://cacm.acm.org/news/nist-post-quantum-cryptography-candidate-cracked/).

-------------------------

0xfffffffa | 2025-02-13 08:39:33 UTC | #8

Thank you for your reply. Its a valid question and we can both theorize at this point and of course use our discussions as fuel to help build more robust solutions. 

The way the solution has been designed is that it starts life as an L2 working as a parallel system (think like lightning network) works as a parallel system on top of Bitcoin. 

As some point when a cryptographically relevant quantum computer comes to fruition and is capable of attacking Bitcoin and this is observed by the community / us / everyone on the blockchain that this has happened the L1 Bitcoin chain as we know it can no longer be trusted. Blocks can be rearranged so on and so forth. 

At this point  the parallel system that was working as backup becomes the new L1. Its capable of it as it has its own consensus, its own independent blockchain. 

When I say "working as a backup" the concept is more akin to a third wheel that continuously is working and supporting but if one of the other wheels gets punctured it can be relied upon to continue functioning as normal and as a replacement to the wheel that was punctured - in other words people can use it now on testnet and mainnet once it launches to quantum wrap their BTC.

I am aware of the article you referenced regarding SIKE. Our system uses FIPS240 ML-DSA which for all intents and purposes at present is considered by NIST approved and went through their rigorous verification process.

-------------------------

vazertuche | 2025-02-21 20:04:43 UTC | #9

I think a push to go straight to a quantum-resistant address type is too premature. 

The best thing to do for people that want to be proactive about this threat is to finalize as quickly as possible the most secure and simple quantum proof signature scheme, likely something akin to Lamport signatures since they are simple and hash based. (Yes, they are massive, but we’ll get to that in a minute) 

Your BIP would lay out exactly how this would be rolled out in SegWit version 2. This version 2 quantum address (bc1z…) type would add to taproot such that it includes ECDSA, Schnorr, and Lamport signatures. 

When wide consensus is gathered around this, wallets could begin to implement it immediately without any soft fork or hard fork needed. They would simply make it standard practice to always include at least one Quantum Lamport signature path within the script tree of every taproot address generated. In the event of a sudden quantum attack, we could immediately roll out the soft fork adding a segwit v2 quantum address type and also enables taproot spending of the hidden Lamport script path. This would allow people to freely move from taproot to the new v2 quantum address without fear of a quantum computer sniping their transaction in the mempool.

Now this scenario is highly unlikely in the near term. Personally, I don’t believe there is any evidence of a quantum computer being able to break our encryption for at least 15 to 20 years. But that’s besides the point.  Using this approach allows us to leverage the time needed to work out the robust math that Bitcoin requires and also allows us to be proactive about doing something in the meantime. In 20 years, we could have easily already moved consensus to a proven lattice based crypto that is more compact and then after that, hopefully to a proven supersingular EC Isogeny crypto also that is compact enough to be acceptable to Bitcoins size constraints AND we can do all this without ever actually implementing any type of fork. 

In this hypothetical scenario, when the quantum attack happens in 20 years then 90% of people will be able to move their taproot addresses via the newest quantum algo, 8% on the older lattice algo and 1% using the oldest Lamport algo. The remaining 1% will either be lost or we can allow ECDSA and Schnorr script path spends ONLY. These people would be able to connect directly with a mining pool via a quantum secure connection and spend that way. 

Yes, trust will be assumed on the miner but it is better than just having those people get burned. In addition, this whole plan requires trust that any of these offline algo’s will ever get implemented via a soft-fork. In addition, what about all the people not using taproot addresses? All things being equal, while it would seem that taproot addresses are the worst option to use in this new quantum world they might actually be our salvation. 

Given the quirky nature taptweak and the two different spending paths, we can use that to our advantage. It can’t be overstated how powerful a taproot address in which the primary path is un-spendable via a NUMS point is. 

It’s powerful since it acts as a honeytrap for a quantum computer. If it was my address and a quantum computer stole the funds, I would be able to raise the alarm to everyone that it WAS in fact a quantum computer that stole the funds. I would be able to prove this by revealing the necessary info to show that the primary path was in fact an un-spendable NUMS point and that I had true ownership over a pubkey in the script path. (In fact, you wouldn’t even need to do the last part. Showing the NUMS point and the merkle root is enough) Everyone would be able to see that the valid signature could have only been created by a quantum computer. (or all our crypto we know now somehow broke)

Given some basic game theory, this actually gives Bitcoin a huge tactical advantage. If a majority of UTXO’s actually move to taproot and if wallets begin to make it common practice to make the primary path un-spendable if necessary and even if not necessary (example: to create deliberate honey pot addresses) then we’ve effectively laid out land mines for a quantum computer. Any adversary who gains the first quantum mover advantage will want to retain that advantage as long as possible without alerting the world at large. The fact that any attack on a Bitcoin address “could” fully reveal their advantage would be incentive enough to not even try to attack Bitcoin. Exchanges and large ETF players would of course be incentivized to use taproot addresses with un-spendable primary paths for all their large holdings. 

If an adversary instead wanted to use all their quantum power to just attack bitcoin directly, then again, we would know fairly immediately that we were being attacked and could shut down all non-quantum algo’s and implement a soft fork immediately. We could implement some form of Tadge’s idea from the mailing list about a pre-emptive quantum soft fork that is triggered by one of these honeypot taproot addresses. Soft-forks are contentious as-is. Having solid proof of a quantum attack should alleviate a lot of obstacles in implementing a soft-fork in an expedited manner. 

All in all I think this method buys us TIME and that is what is needed for this new frontier of quantum math to develop. It buys us TIME and appeases the masses who will remain hyper vigilant about quantum computers suddenly appearing in the near term.

In the end, what we really want is a quantum proof of knowledge that is compact. Add that with some client-side verification and quantum thrustless bridges and we are really in business. 

If the stars align right, we can get that before a quantum threat really does become real (in 25+ yrs) In the meantime we can simply leave a trail of alternate quantum spending paths that are all TOO large to ever be feasibly implemented and thus subsequently abandoned in never revealed script leaves.

-------------------------

