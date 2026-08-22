# Seedroller and keyderiver: cli tools to generate BIP39 seed words and derive BIP380 keys

bubb1es | 2026-08-22 05:13:29 UTC | #1

In light of recent events I became interested in how to safely generate a strong random seed. The closest tool I found in rust is [seedtool-cli-rust](https://github.com/BlockchainCommons/seedtool-cli-rust) but it has a lot of dependencies I don't recognize and I prefer building on the rust-bitcoin orgs crates. 

I humbly submit my own cli tool `seedroller` for generating BIP39 seed words from dice rolls hashed with `getrandom()` system entropy. I also created a companion cli tool `keyderiver` that takes your seed words and derives BIP380 descriptor key expressions. Both are based on rust-bitcoin org rust crates and have minimal other dependencies. For all the fun details please checkout the [bitcoin-key-tools](https://github.com/bubb1es71/bitcoin-key-tools) repo. 

Feedback is much appreciated, this is still early work and I'm happy to implement suggestions that improve these tools soundness and security.

-------------------------

