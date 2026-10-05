# Descriptor syntax for hash preimages

jaonoctus | 2026-10-05 12:28:49 UTC | #1

## Summary

Descriptors can carry private keys (`pk(WIF/xprv)`) and return them with `listdescriptors private=true`, but there is no equivalent for the secrets of miniscript hash fragments. I'd like to discuss adding a private form, e.g. `sha256(preimage(HEX))`, and the same for `hash256`, `ripemd160` and `hash160`, possibly as a new BIP extending BIP 379.

## Motivation

I ran into this while solving the [TABConf](https://github.com/TABConf) / [Michael Tidwell](https://github.com/miketwenty1) CTB challenge, where spending required providing a hash preimage. I expected it to work like `pk(WIF/xprv)`: put the secret in the descriptor, import it, and let the wallet build and sign the transaction.

Instead, there is currently no way to give a preimage to Bitcoin Core's wallet:

* the descriptor only accepts the digest;
* no RPC adds preimages to a PSBT;
* the only path is inserting `PSBT_IN_SHA256` (or the other hash fields) with an external tool before `walletprocesspsbt`/`finalizepsbt` can complete the witness.

## Why preimages are closer to keys than to signatures

#24114 suggests descriptors could eventually replace most of `SignatureData`, "everything except signatures/preimages". Signatures commit to a specific transaction, so they can never be part of a descriptor. A preimage does not: once generated (or given to you ahead of time), it is a long-lived secret that depends only on the script. That is exactly what a private key is.

|  | Keys | Hashes |
|----|----|----|
| Public | `pk(PUBKEY)` | `sha256(DIGEST)` |
| Private | `pk(WIF/xprv)` | *(none right now)* |

In @sipa's words from the same issue:

> if we keep adding exceptions, perhaps the philosophy is wrong.

"Descriptors may carry private keys but not preimages" looks like one of those exceptions.

## Sketch/Draft:

* **Syntax.** `sha256(preimage(HEX))`, `hash256(preimage(HEX))`, `ripemd160(preimage(HEX))`, `hash160(preimage(HEX))`.
* **Exactly 32 bytes.** All four fragments compile to `OP_SIZE 32 OP_EQUALVERIFY <HASHOP> <digest> OP_EQUAL`, so a valid preimage is always 32 bytes, even for `ripemd160`/`hash160`. Any other length would be unsatisfiable and should be rejected by the parser.
* **Explicit marker required.** For `sha256`/`hash256`, preimage and digest are both 32 bytes, so raw hex is ambiguous by construction. Keys avoid this because WIF/xprv have their own encodings.
* **Serialization.** The public string prints the digest; the private string (`private=true`) prints the preimage form. Descriptors inferred from scripts produce the digest form, just as they never contain private keys.
* **Wallet/signing (Bitcoin Core).** Today the miniscript satisfier looks up preimages only in `SignatureData`, populated from PSBT fields. Preimages from a descriptor would be stored and encrypted alongside keys, exposed through the `SigningProvider`, and used by the satisfier in addition to `SignatureData`.
* **Safety.** A revealed preimage is public, but sane miniscript already requires a signature on every satisfaction path (the `s` property), so a preimage alone never spends. The preimage form is a secret like a WIF and should be handled the same way (backups, encryption, never in the public string).

This does not replace the PSBT path: preimages learned at spend time (e.g. from a counterparty in an HTLC) still belong in PSBT fields. It only covers preimages known when the descriptor is created.

## Open questions

1. **Marker syntax:** `preimage(HEX)` nested inside the fragment, a prefix, or something else?
2. **Scope:** an amendment to BIP 379, or a separate BIP like other descriptor extensions?
3. **Complementary RPC:** independently of descriptors, should Core get an RPC to add hash preimages to a PSBT, for the learned-at-spend-time case?
4. **Signers/hardware wallets:** would devices that accept miniscript descriptors want to handle preimage secrets, or should this be software-wallet only?
5. **Deterministic preimages:** is there interest in deriving preimages from key material (similar to how Lightning derives secrets), or is a literal preimage enough for a first step?

Related threads: https://github.com/bitcoin/bitcoin/issues/24114#issuecomment-5978130575 , [Tr(): rawnode() and rawleaf() support](https://delvingbitcoin.org/t/tr-rawnode-and-rawleaf-support/901), [Unspendable keys in descriptors](https://delvingbitcoin.org/t/unspendable-keys-in-descriptors/304), [BIP352 private key formats](https://delvingbitcoin.org/t/bip352-private-key-formats/2080).

-------------------------

