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

