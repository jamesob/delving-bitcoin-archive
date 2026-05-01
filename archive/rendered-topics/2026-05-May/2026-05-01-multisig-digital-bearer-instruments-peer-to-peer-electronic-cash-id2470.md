# Multisig Digital Bearer Instruments - peer to peer electronic cash

keys | 2026-05-01 11:42:54 UTC | #1

I am posting this to share a new cryptographic protocol for trustless Bitcoin bearer instruments and invite technical feedback.

**The core idea**

A 2-of-3 P2SH multisig where the key distribution is inverted from prior schemes: the bearer holds Keys B and C (the spending threshold), the issuer holds only Key A (permanently sub-threshold). Key A is deleted immediately after address construction, before funding. The result is an instrument that transfers through key handover with no on-chain transaction, no fee, and no network requirement.

spend = f(kB, kC)

The issuer cannot spend, cannot block a spend, and cannot be compelled to enable one. This is enforced by Bitcoin consensus rules, not policy.

**What is novel**

Prior multisig bearer schemes position the issuer at or above the spending threshold. The threshold inversion — placing the spending threshold entirely with the bearer and the issuer permanently below it — is the departure from prior art.

Secondary contributions: a buyer-generated key issuance protocol closing the malicious issuer attack; a Nostr receipt mechanism providing cryptographic proof of receiver possession before sender key deletion; a pending_transfer intermediate state allowing cancellation without fund loss; and an NC1 security gate preventing key deletion from being triggered by a forged receipt for an unfunded instrument.

Keys live in the device's secure hardware enclave - Android Keystore (AES-256-GCM, hardware security chip; on StrongBox devices keys are designed to never leave the Titan M2) and iOS Secure Enclave (keys are generated and stored inside the Secure Enclave and are designed to be non-extractable under normal operating conditions). No cloud. No server. No custody.

**Proof of concept**

A working implementation on Bitcoin Signet demonstrates the full lifecycle. Batch funding confirmed in block 297,032 on 2026-03-24, P2P transfer with Nostr receipt and on-chain sweep confirmed in block 297,198 on 2026-03-26. Both transactions are verifiable on the Signet explorer.

**What is not yet implemented**

The paper distinguishes specified design from current implementation. Not yet implemented: bounded UTXO set and offline verification, multi-node UTXO querying, NFC and near-field transport bindings, P2TR/MuSig2 upgrade path.

**Open questions I would value feedback on**

* The NC1 gate requires both cryptographic receipt verification and on-chain UTXO confirmation before sender key deletion. Are there attack vectors not covered by this combination?

* The buyer-generated key issuance protocol closes the malicious issuer attack for digital instruments. For physical instruments where buyer key generation is impractical, TEE attestation is described as a weaker alternative. Is there a stronger approach?

Full paper and protocol specification: [https://doi.org/10.5281/zenodo.19382670](https://doi.org/10.5281/zenodo.19382670)
Working app at https://kagikai.app

-------------------------

keys | 2026-05-01 07:09:39 UTC | #2

Things have progressed a little since I drafted this post …

The P2TR/MuSig2 upgrade, listed above as “not yet implemented”, is now complete and deployed on Bitcoin mainnet with real payments confirmed. The instrument address is now a standard P2TR output where the internal key is a MuSig2 aggregate of the two bearer keys (KB, KC). The issuer key (KA) appears only in script tree leaves: a 2-of-3 fallback (BIP 342 OP_CHECKSIGADD) and a CLTV recovery path. KA is deleted before funding. The key path spend requires only kB and kC via a single 64-byte Schnorr signature indistinguishable from any ordinary P2TR payment.

**Multi-node UTXO verification.** Dual-node verification against independent APIs (blockstream.info + mempool.space) is implemented with a configurable threshold. Instruments above 50,000 sats require both nodes to agree before the instrument is accepted as valid.

**Buyer-generated key issuance.** The buyer generates kB and kC locally via BIP32 HD derivation on a fully hardened path (m/77’/0’/0’/N’/0’ and N’/1’) and transmits only the public keys KB and KC to the issuer. The issuer never possesses the bearer private keys at any point, closing the malicious issuer vector by the intractability of ECDLP rather than by trust or deletion attestation.

**E-commerce merchant integration.** A headless KMS (Key Management Server) accepts NaCl-sealed payment bundles via Nostr, verifies the UTXO on-chain, enqueues for batched sweep, and fires a webhook to the merchant’s e-commerce platform (WooCommerce plugin currently). The KMS publishes a cryptographic receipt (ECDSA Sig(kB, nonce) sealed to the customer’s X25519 pubkey) back via Nostr so the customer app can confirm transfer and delete its copy of the keys. The full flow, from customer app to checkout page to payment to order confirmation, is working end-to-end on mainnet.

One observation from mainnet hardening that may be relevant to the NC1 discussion in the original post: we found that receipt verification alone (Sig(kB, nonce) verified against KB) is not sufficient for the headless merchant case. The KMS must verify the UTXO on-chain before publishing the receipt. Otherwise a malicious customer could submit a bundle for an unfunded address, receive a receipt for an instrument that has no value, and potentially use that receipt to trick a sender in a P2P chain into deleting keys prematurely. The implemented gate requires both on-chain UTXO confirmation and cryptographic receipt verification before any key deletion occurs.

The paper at the DOI link above has been updated to reflect P2TR/MuSig2 as the primary construction. The P2SH formulation described in my original post above is retained in the paper as a simplified reference model.

-------------------------

AdamISZ | 2026-05-01 15:40:33 UTC | #3

I don't understand why a protocol based on "transfer private key, then delete it" is interesting [1], since the receiver can never know for sure that it has indeed been deleted, and the sender is economically incented to save it somewhere.

Of course with a TEE/HSM etc. you can always argue that arbitrary computation can be trusted (indeed, Mercury for example did that), and in this model you can do basically anything. It seems to fit more client-server than p2p though, as demanding every participant have such an HSM is not only impractical (isn't it?) but also a bad trust model (how do you know they are using it? Attestation I guess?). I'm a bit vague on this because I've always thought these models are dubious just in general, but that's arguable to be fair.

I also don't understand why your paper spends the first several pages repeating in many, many different ways that Key A is deleted; it's a pretty simple concept! Honestly I'm not sure that having a third key adds that much, but, fair enough, it is an extra element you can use in an "Issuance" scenario.

It also makes it quite difficult for a casual reader to reach the actual point of the protocol, which is that key B and key C are deleted, since this is what effects the transfer.

[1] well, I mean basing transfer of actual ownership purely on that; there are fancy, complex protocols that have key deletion as part of their operation: a good example is emulating covenants with large scale setup ceremonies, doing Shamir secret sharing to produce an output, then you only need 1 of n people to honestly delete. And there are even fancier ideas that include key deletion .. but "you have my coins at this private key because I deleted it" is not such a thing.

-------------------------

