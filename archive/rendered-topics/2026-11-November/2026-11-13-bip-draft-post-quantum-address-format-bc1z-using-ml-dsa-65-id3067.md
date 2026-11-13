# BIP Draft Post-Quantum Address Format (bc1z) using ML-DSA-65

Zaenus | 2026-11-13 02:35:28 UTC | #1

Dear Bitcoin Development Community,

I'm sharing a draft BIP for a quantum-resistant address format for Bitcoin, developed as part of the safe-btc project (https://github.com/Zaenus/btc-safe).

## Summary

This BIP proposes a new `bc1z` address format using ML-DSA-65 (FIPS 204) post-quantum signatures. The format commits to a SHA-256 hash of the ML-DSA public key, revealing the full public key only at spend time in the witness — similar to how P2WPKH works today.

## Motivation

Bitcoin's ECDSA and Schnorr signatures are vulnerable to Shor's algorithm on a cryptographically relevant quantum computer (CRQC). Once a public key appears on-chain, a quantum attacker can derive the private key and spend the funds. Approximately 6.9 million BTC (\~$586B) sits in addresses with exposed public keys.

## Design Highlights

* **Address format:** `bc1z<witness version 0x02><32-byte SHA-256 commitment>`
* **Spend witness:** `<ML-DSA signature (3,309 bytes)> <ML-DSA public key (1,952 bytes)>`
* **Soft fork compatible:** Old nodes see `bc1z` outputs as anyone-can-spend (witness version > 1)
* **Off-chain reference implementation available:** Python library with key generation, signing, verification, migration helper, and quantum canary monitor
* **Signature aggregation prototype:** SNARK-wrapped ML-DSA signatures compress \~33KB of individual signatures to \~2KB for 10 signatures

## Repository

https://github.com/Zaenus/btc-safe

The full BIP draft is in `docs/bip/0001-pq-address-format.md`.

## Request for Comments

I'm looking for feedback on:

1. Address format design (HRP `bc1z`, witness version 0x02, 32-byte commitment)
2. ML-DSA-65 parameter choice (security vs. signature size trade-off)
3. Soft fork deployment considerations
4. Migration path for existing UTXOs
5. Whether the community sees value in pursuing this before a CRQC exists

This is an early draft intended to start discussion. I'm happy to revise based on community feedback.

Best regards,
Zaenus
https://github.com/Zaenus

-------------------------

Zaenus | 2026-11-13 17:28:51 UTC | #2

**Revised draft (v0.2): key type byte, BIP-341 sighash, full spending rules, test vectors**

Thanks for the feedback so far. Here and on the list, it led to a substantial revision. The draft and reference implementation are at https://github.com/Zaenus/btc-safe/blob/4e09003f69ee276f8df1eadcc913ab98a84dbf52/docs/bip/0001-pq-address-format.md , with vectors in `docs/bip/0001-test-vectors.json`.

### Corrections to the previous version

In the interest of transparency:

* The reference code emitted **bech32 rather than bech32m** checksums for witness v3, so every BIP-350 decoder would reject those addresses. Fixed; the official BIP-350 vectors now pass.
* The text claimed a BIP-143 sighash, but nothing implemented it. It is replaced (see below).
* The "SNARK aggregation" prototype is withdrawn. Pairing-based SNARKs aren't post-quantum sound, and consensus can't depend on a trusted setup. What's left is a short research note; aggregation is out of scope for this BIP.

### What changed in the spec

```
key_blob     = key_type || pubkey                      (0x01 = ML-DSA-65)
commitment   = TaggedHash("PQKeyCommit", key_blob)
scriptPubKey = OP_3 OP_PUSHBYTES_32 <commitment>       -> bc1r... (bech32m)
witness      = [signature(||hash_type), key_blob, (annex)]
sighash      = TaggedHash("PQSighash", 0x00 || SigMsg(hash_type, 0))  # BIP-341 SigMsg, verbatim
verify       = ML-DSA.Verify(pk, sighash, sig, ctx="bitcoin:pq:tx")
```

* **Key type byte with an upgrade hook.** The key blob is always the last witness element, and the stack shape is checked per key type. An undefined key type succeeds, like OP_SUCCESS. ML-DSA-44, SLH-DSA, FN-DSA or a hybrid type can then be added later without spending another witness version (only v2–v16 remain) and without locking future types into a two-element witness. `0x50` is rejected explicitly (annex collision).
* **BIP-341 SigMsg under a distinct tag.** It commits to every input's amount and scriptPubKey, which avoids the BIP-143 fee-lying issue for signing devices, and lets implementations share Taproot code. The different tag means a Taproot sighash can never equal a v3 sighash.
* **FIPS 204 context strings** separate transaction signatures from off-chain message signatures (`"bitcoin:pq:msg"`).
* **Also specified:** non-32-byte programs remain unencumbered, P2SH-wrapped v3 is unencumbered and non-standard, plus relay policy (no annex, no undefined key types).

### Two points worth stating explicitly

1. **Unlike P2WPKH, a v3 output stays quantum-resistant after its public key and signatures are revealed.** The hash commitment exists only to keep outputs at 34 bytes. Mempool (short-exposure) attacks don't apply, and address reuse is safe against a quantum attacker, though still bad for privacy. Thanks to Zeeshan Ali Khan for pointing out that the safe-after-reveal property deserved to be stated explicitly.
2. **This has to be in place *before* a CRQC exists.** Activation takes years, and migration is limited by block space. A 1-in/1-out P2WPKH → v3 move is 486 WU, so every 10M UTXOs needs about 1,200 entirely full blocks.

### Costs

|  | Weight | Per block |
|----|----|----|
| ML-DSA-65 input | 5,433 WU (\~1,358 vB) | \~736 |
| ML-DSA-44 input (for comparison) | 3,904 WU (\~976 vB) | \~1,024 |
| Taproot key-path input | 230 WU | — |

A 1-in/2-out v3 transaction is \~1,455 vB.

### Test vectors

These cover keys and addresses, BIP-39 derivation, sighash for all seven hash types with and witho valid, invalid, non-standard and upgrade-hook. Signatures are deterministic, produced by anindependent implementation (dilithium-py) and verified by the OpenSSL-backed reference validator. The SigMsg code also passes the official BIP-341 vectors. Independent implementations checking these vectors would be very
welcome.

### Open questions, where I'd most value input

1. **ML-DSA-44 vs ML-DSA-65** as the first key type. 44 is 28% smaller per input; 65 gives more maryptanalysis. With the key type byte, this is a choice of default rather than a permanentcommitment.
2. **Should a hybrid secp256k1 + ML-DSA key type be defined from day one?** It costs about 100 bytst a classical break of lattices.
3. **Relationship to P2MR (BIP-360).** My view is that they're complementary: P2MR for scripts, v3 for single-key spends, with the `key_blob` encoding reusable by a future PQ checksig opcode in a P2MR leaf. Is there appetite
   for both, or should this become a leaf type instead?
4. **Validation cost.** Each input does exactly one ML-DSA-65 verification, so I haven't proposed a separate sigops budget. Bitcoin Core benchmarks are the next step on my side.
5. **Witness discount.** Should ML-DSA witness data be discounted the same way, given \~5.3 KB per
6. **Allocations:** a BIP-43 purpose number (currently `3'` as a placeholder; derivation is hardenpubs) and PSBT field types (currently proprietary `0xFC`).

Reference implementation: https://github.com/Zaenus/btc-safe . Changelog: `CHANGELOG.md`.

-------------------------

