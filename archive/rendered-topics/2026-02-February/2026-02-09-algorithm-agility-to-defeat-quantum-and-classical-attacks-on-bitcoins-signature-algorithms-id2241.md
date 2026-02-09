# Algorithm agility to defeat quantum and classical attacks on Bitcoin's signature algorithms

EthanHeilman | 2026-02-09 19:56:17 UTC | #1

I want to share my thoughts on increasing the security of Bitcoin to long term threats such as quantum and classical breaks in Bitcoin’s signature algorithms by adding algorithm agility mechanisms to Bitcoin. To quote [RFC 7696: Guidelines for Cryptographic Algorithm Agility](https://datatracker.ietf.org/doc/html/rfc7696):

“Protocol designers need to assume that advances in computing power or advances in cryptoanalytic techniques will eventually make any algorithm obsolete.  For this reason, protocols need mechanisms  to migrate from one algorithm suite to another over time.

Algorithm agility is achieved when a protocol can easily migrate from one algorithm suite to another more desirable one, over time.”

I propose introducing a very secure post-quantum signature algorithm to be used alongside the current algorithms. This signature algorithm can be very expensive in txn fees and block space since it is for emergency migrations only. This enables Bitcoin holders to cheaply and easily create outputs that “failsafe” even against  an unexpected break in a signature algorithm.

## **Motivation**

Bitcoin should enable a person to self-custody coins for at least one human lifetime, \~75 years. Someone should be able to bury an HD Seed in a coffee can and then dig it up in 75 years and spend those coins. No store of value can promise complete safety on long timescales, but the trust we build by demonstrating that Bitcoin is serious about mitigating long-term low probability risks. Trust and credibility is built even when the risk we defended against never happens.

The main risk I will be considering here is the loss of the ability to authenticate ownership of coins resulting from a break in a digital signature algorithm used by Bitcoin. Such risks are extremely unlikely in the short term (1 to 5 years), but become more likely on 5 to 75 year timescales. Most of the focus in the wider cryptocurrency world has been on mitigating the quantum threat, but I take a less narrow view of the problem. We should consider not just Quantum attacks on Bitcoin’s signature algorithms and also classical breaks that do not require a quantum computer. One particular area of concern to me is an unexpected breakthrough in Mathematics driven by AI approaches.

To address these risks we propose the following design for protecting holders even against an unexpected break in Bitcoin’s signature algorithms quantum or otherwise.

## **Design**

Assume that Bitcoin supported two digital signature algorithms: DSA1 and DSA2. Each signature scheme would have its own CHECKSIG opcode, CHECKSIG_DSA1 and CHECKSIG_DSA2.

Using BIP 360, we could have a leaf script for CHECKSIG_DSA1 and a leaf script for CHECKSIG_DSA2.

* Leaf1: DSA1_PUBKEY, CHECKSIG_DSA1
* Leaf2: DSA2_PUBKEY, CHECKSIG_DSA2

If DSA1 was broken, users could simply switch to spending the output with DSA2 signatures by using leaf2. An attacker that could break DSA1, wouldn’t be able to learn the public key for DSA1, and thus wouldn’t be able to spend DSA1, despite DSA1 being vulnerable.

This approach makes the assumption that the user has not leaked their public key to an attacker or reused their public keys. As a user wishing to hold Bitcoin in an output over long periods of time generate a fresh set of public keys for that output.

Our approach does not defend against the case where DSA1 and DSA2 are broken at the same time. For this reason, DSA1 and DSA2 should use different cryptographic assumptions. Additionally DSA2 should use a signature scheme that trades off efficiency for additional security and robustness. This way, we can get the best of both worlds, DSA1 can be used for everyday signatures and if DSA1 is broken, DSA2 can be used to migrate to a new signature scheme, say DSA3. Even if DSA3, chosen in haste to replace DSA1, is also found to be weak, holders are still protected. This is because DSA2 is unbroken, allowing us to replace DSA3 with DSA4.

* DSA1 - Efficient, low cost to use, should support nice-to-have features such as aggregation.
* DSA2 - Expensive, extreme levels of security, only used to transition to new signature schemes.

A person with an HD seed buried in a coffee can for 75 years is still safe even if they don’t transition from DSA1 to DSA3 and since DSA2 is still safe. When they dig it up, they can use DSA2 to move the output to DSA3+DSA2.

Given this framework, let’s think about DSA1 and DSA2 with concrete algorithms:

* DSA1 is Schnorr, the currently supported Schnorr signature algorithmin Bitcoin.
* DSA2 is SLH-DSA (SPHINCS+ SLH-DSA-SHA2-128s). SLH-DSA is a hash based signature algorithm. Hash based signatures are the most likely secure long term.

If we merged BIP 360 and support for a SLH-DSA CHECKSIG opcode, holders could hedge against a classical or quantum attack against Schnorr, while still using Schnorr.

Our approach mitigates the main drawback of the size of SLH-DSA signatures, their size. SLH-DSA signatures are 8 KB in size, while \[0\] explores methods for reducing the size of these signatures to 3 KB, 3 KB is still huge. Because SLH-DSA signatures would not have any additional discount, they would be very expensive in transaction fees, and only economically worthwhile to migrate from Schnorr to a new signature scheme. In all other cases the addition of SLH-DSA would exist as unused leaf scripts in the output, which increases the witness by 32 bytes.

An additional benefit to this approach of using BIP 360 and SLH-DSA would be to buy time for non-hash based PQ signature schemes to mature. We are seeing rapid advances in research on post-quantum signature schemes and waiting a little long might buy us a lot. SLH-DSA would provide an effective hedge against this risk, while delaying the decision of what PQ signature scheme should replace Schnorr in the event of Schnorr’s security being weakened.

## **Q & A**

**Q:** Could these signatures be abused to store JPEGs on the blockchain?

**A:** No because they would have no additional discount. This means they would have no advantage for JPEGs over what is currently possible.

–

**Q:** Why not use XMSS or Lamport signatures instead of SLH_DSA?

**A:** I prefer SLH_DSA because it is likely to be well supported outside of Bitcoin and Bitcoin can benefit from this ecosystem of support in the form of HSMs, hardware acceleration and software liberties. That said, it is reasonable to consider stateful hash based signature schemes like XMSS, Winternitz, or Lamport signatures as the fallback signature, especially if size becomes a concern.

–

**Q:** What non-consensus critical changes would be needed to support this?

**A:** We’d need to create new wallet standards to provide alternatives to BIP32 xpubs. Wallet would have to write code to generate SLH-DSA keys and create a script tree per signature alg. Wallets would also have to put into place mechanisms to warn for and prevent public key reuse.

–

**Q:** What consensus critical changes would be needed to support this?

**A:** We’d need to merge something like BIP 360 and then a new CHECKSIG opcode for SLH_DSA.

–

**Q:** Couldn’t you do this without BIP 360 by using Taproot instead and then disabling the taproot key spend path?

**A:** Yes, however this would be confiscatory, since Taproot allows key spend path only outputs. People holding key spend path-only Taproot outputs would have the coins in those outputs destroyed. BIP 360, in essence is Taproot, without the key spend path. BIP 360 provides the same functionality as disabling Taproot key spend paths, but rather than being confiscatory, it is opt in.

–

**Q:** Are you proposing this now because you think that the Bitcoin signature algorithms are under threat?

**A:** No, I am confident in the Bitcoin signature algorithms and I know of no immediate threats. This effort is motivated by longtermism and thinking about how to enable Bitcoin to be safe on timescales of decades or centuries.

\[0\]: Mikhail Kudinov, Jonas Nick, Hash-based Signature Schemes for Bitcoin (2025) [https://eprint.iacr.org/2025/2203](https://eprint.iacr.org/2025/2203)

-------------------------

