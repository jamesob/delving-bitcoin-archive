# [Rust] descriptor-encrypt: Encrypt any descriptor such that only authorized spenders can decrypt

josh | 2025-06-04 01:43:19 UTC | #1

## Tldr

[descriptor-encrypt](https://github.com/joshdoman/descriptor-encrypt) is a rust library that deterministically encrypts wallet descriptors so that they can only be decrypted by a set of keys that can spend the funds. This enables secure public backups where sensitive wallet information needs to remain hidden from unauthorized parties.

## Intro

For the B25 hackathon, I built a rust library, [descriptor-encrypt](https://github.com/joshdoman/descriptor-encrypt), which can deterministically encrypt any wallet descriptor such that it can only be decrypted by a set of keys that can spend the funds.

It supports all descriptors types and miniscript, and it can encrypt the descriptor such that no information is revealed about key inclusion unless enough keys are present to fully decrypt. It also uses a tag-based variable-length encoding scheme to minimize the amount of data that needs to be stored.

This project is a follow-up to a proof-of-concept I shared [earlier this year](https://delvingbitcoin.org/t/multisigbackup-com-backup-and-recover-a-k-of-n-descriptor-using-only-n-seeds/1430), which only supports standard non-taproot multisigs. I'd like to thank @notmandatory, who suggested I create a rust library to make it easier for wallets to support. 

## How it works

This library encrypts any Bitcoin wallet descriptor in a way that mirrors the descriptor's spending policy:

- If your wallet requires 2-of-3 keys to spend, it will require exactly 2-of-3 keys to decrypt.
- If your wallet uses a complex miniscript policy like "Either 2 keys OR (a timelock AND another key)", encryption follows the same structure, as if all timelocks and hash-locks are satisfied.

To do this, the descriptor's spending policy is analyzed and transformed into a tree-like structure, where each node is a threshold and each leaf is either a key or a keyless condition. Keyless leaves are then pruned and each threshold is updated to reflect a tree where all satisfiable keyless conditions are satisfied.

Next, a master encryption key (derived deterministically from the descriptor) is sharded using recursive Shamir secret sharing into an identical tree-like structure, and each leaf is encrypted using the corresponding key. As a result, decryption requires access to enough keys to spend the funds.

In the default mode, shares are encrypted using Chacha20-Poly1305 and the payload is encrypted using ChaCha20. This enables fast decryption for arbitrarily large descriptors, but it leaks information about key inclusion, which is not ideal for setups where encrypted backups are stored in public.

For maximum privacy, `full-secrecy` mode can be used, which encrypts shares using ChaCha20 and encrypts the payload using ChaCha20-Poly1305. This reveals no information about key inclusion unless the payload can be decrypted, but it's slower to decrypt, as we must try all possible combination of shares and keys. This has a running time of $O((N+1)^K)$, where $N$ is the number of decryption keys and $K$ is the number of shares in the descriptor. This takes milliseconds for typical descriptors but can be computationally slow for extremely large ones.

## Encoding

Prior to encryption, the descriptor is encoded using a tag-based encoding scheme (see [tag.rs](https://github.com/joshdoman/descriptor-encrypt/blob/269e8eeb91362900fb5c4f8ccd521a54eb2dd3b7/src/template/tag.rs)). Variable-length encoding is used for integers in derivation paths, timelocks, thresholds, etc.

This encoding splits the descriptor into two byte arrays, a "template" and a "payload." The template contains the structure of the descriptor and the derivation paths, and the payload contains the sensitive data, such as the public keys, xpubs, master fingerprints, hash-locks, and time-locks. Only the payload is encrypted, while the template remains visible in plaintext, so that users know how to derive the necessary keys and recover the descriptor.

The final scheme has the following format:
```
[version][template][encrypted shares][encrypted payload]
```

## Alternatives

In [this post](https://delvingbitcoin.org/t/a-simple-backup-scheme-for-wallet-accounts/1607/20), @salvatoshi proposed an alternative scheme, which only requires a single key in order to decrypt. The simplicity of this scheme is its best property, and it could be suitable for users that want to backup their descriptor via email or the cloud without a hacker or cloud provider being able to see their balance.

This library has a different objective, as it aims to provide maximum privacy to wallet backups, so that no information is revealed about balances, transaction history, or key inclusion except to users authorized to spend the funds. This degree of privacy is advisable if we want to use a public blockchain or another public forum (i.e. social media) to store encrypted backups.

There are several benefits to storing encrypted backups in public instead of via email or in the cloud:
1) **Simplified inheritance**: Heirs can scan the public database for descriptors they can decrypt and recover the funds.
2) **Better privacy**: A hacker that gains access to Gmail or cloud storage learns nothing about the existence of a Bitcoin wallet, making users less susceptible to wrench attacks.
3) **Decoy support**: With full secrecy mode, an attacker who steals one seed cannot determine whether that seed is part of a larger descriptor.

The encoding scheme is designed to support public backups, by minimizing the amount of data that needs to be stored. For example, an encrypted standard 2-of-3 descriptor (with fingerprints and derivation paths) is only 402 bytes. This is ~100vb if stored as witness data on Bitcoin, which becomes fairly cost effective if the taproot annex becomes standard.

## Demo

This [GitHub repo](https://github.com/joshdoman/descriptor-encrypt/tree/main) contains a command-line tool, where the library can be run locally. In addition, I have used Web Assembly to port it to the browser, where it can be demo'd at descriptorencrypt.org.

![](upload://bLgDNRLaUBqAl1s6no3XhqdzmpT.jpeg)


## Further work

This library aims to provide a robust foundation for encoding and encrypting Bitcoin descriptors, with maximum privacy guarantees. While more complex than other schemes, it's built as a rust library to make it easy to port to other languages and integrate into wallets.

The library is intentionally un-opinionated about *where* descriptors are stored. If stored on a public blockchain or another public database, it's advisable to append a short hash of each combination of fingerprints, corresponding to seeds that can be used to decrypt. This would facilitate indexing and rapid lookup during the recovery process.

I hope the community finds this interesting. If there's interest, I can flesh out the implementation in a formal specification.

### Links

Github: [https://github.com/joshdoman/descriptor-encrypt](https://github.com/joshdoman/descriptor-encrypt)

Docs: [https://docs.rs/descriptor-encrypt/latest/descriptor_encrypt/](https://docs.rs/descriptor-encrypt/latest/descriptor_encrypt/)

Demo: https://descriptorencrypt.org

-------------------------

alex | 2025-06-04 19:46:04 UTC | #2

So modest. 😮‍💨 You mentioned the hackathon but didn't mention you won 🥈.

-------------------------

mr21free | 2026-03-25 11:13:10 UTC | #3

Really interesting work. Makes sense to leave the storage question out of scope for the library itself. I am curious what your vision is for the storage layer though. Where do you see these encrypted blobs actually living long term, especially in an inheritance scenario?

-------------------------

josh | 2026-03-25 18:32:24 UTC | #4

> Where do you see these encrypted blobs actually living long term, especially in an inheritance scenario?

Conceivably, descriptors encrypted with this library could be stored alongside each seed phrase, as you might today. The primary benefit is that the activity and balance of the wallet would remain hidden from a thief or an heir who cannot unilaterally spend the funds.

Alternatively, I expect encrypted descriptors to be stored online, ideally in a public well-recognized location. The latter is important so that heirs can find descriptors using only seed phrases, without needing additional details for how to look them up.

In my opinion, a standard protocol that allows a user to locally import master extended public keys and recover all funds controlled by those keys is the ideal user experience (it requires no technical knowledge beyond seed phrases and hardware devices). This would require a data availability layer and an open source indexer, which allows users to find their encrypted descriptors. While I tend to think that the Bitcoin itself is the only data availability layer that users would accept as an industry standard, I could be wrong, and I'd be interested in seeing other solutions.

-------------------------

mr21free | 2026-03-26 08:14:30 UTC | #5

Actually, I tried your solution already. I encoded my descriptor using your library and put together a small script that bundles a local copy of the web interface. An heir just runs it on Linux or Mac, a local web server spins up, they paste in the public keys and get the descriptor back. Bundling locally removes the dependency on your domain or GitHub still being there in 10 years.

Works well as a practical interim solution. The ideal end state in my opinion is wallet software handling this natively - import the encoded descriptor, connect hardware devices, recover the wallet in a few clicks. No command line, no technical knowledge required beyond the seed phrases.

On the data availability layer, the property one might actually need is findability from the keys, durability over decades, and no single company dependency. Bitcoin provides that but at the cost of chain bloat and permanent data exposure. An off-chain network with Bitcoin-anchored economics, e.g. where nodes stake BTC to guarantee availability and lose it if they don't, could satisfy the same requirements without touching the chain. Curious whether you'd consider that a viable alternative or whether Bitcoin's finality is load-bearing for you.

-------------------------

