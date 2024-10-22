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

