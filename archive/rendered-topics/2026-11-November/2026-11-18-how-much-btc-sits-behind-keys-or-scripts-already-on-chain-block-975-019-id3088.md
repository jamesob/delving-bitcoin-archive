# How much BTC sits behind keys or scripts already on chain (block 975,019)

duncan0k | 2026-11-18 03:32:12 UTC | #1

This follows up on https://delvingbitcoin.org/t/standardizing-an-exposure-classification-for-existing-outputs-pre-bip/2866 and adds the reuse counts asked for in the BIP 362 review at https://github.com/bitcoin/bips/pull/2294#discussion_r4469613254.

By the rule below, 7,164,754.12 BTC of the 20,109,207.36 BTC in unspent outputs at block 975,019, 35.6%, sits in outputs whose public key or script is already public.

Each output type has its own rule:

- P2PKH and P2WPKH need their program in `pubkey_hash_index.bin`.
- P2SH counts if its program is in `pubkey_hash_index.bin` or `script_hash_index_20.bin`.
- A P2WSH program has to be in `script_hash_index_32.bin`.
- P2TR, P2PK and bare multisig always count, because the key is in the output script.
- Other forms never count.

The tables hold hashes derived from public keys published on chain and from revealed scripts. P2PK, P2TR and bare multisig, which count by their form alone, hold 1,937,232.46 of the 7,164,754.12 BTC. The other 5,227,521.66 BTC, about 73%, is in P2PKH, P2WPKH, P2SH and P2WSH outputs that count only because their program was found in the tables.

| Output type | Unspent BTC | Key or script public, BTC | Share of type |
| --- | ---: | ---: | ---: |
| P2PKH | 4,502,293.81 | 1,162,510.91 | 25.8% |
| P2WPKH | 8,310,066.71 | 2,137,831.72 | 25.7% |
| P2SH | 3,678,167.10 | 1,177,704.69 | 32.0% |
| P2WSH | 1,678,829.15 | 749,474.34 | 44.6% |
| P2TR | 222,545.28 | 222,545.28 | 100% |
| P2PK | 1,714,618.81 | 1,714,618.81 | 100% |
| Bare multisig | 68.36 | 68.36 | 100% |
| Other | 2,618.15 | 0 | 0% |
| Total | 20,109,207.36 | 7,164,754.12 | 35.6% |

Unspent outputs at block 975,019, rounded to 0.01 BTC; exact amounts and output counts are on the method page. The rule for each type is in the list above the table. The P2SH row includes nested P2WPKH.

In the BIP 362 review, conduition asked "How common is this in the chain history? (reusing a pubkey between P2PK and P2PKH)", and about P2PKH/P2WPKH and "compressed vs uncompressed ECDSA pubkeys". 40,385 keys were paid to as both P2PK and P2PKH. The largest pair, P2PKH and P2WPKH, has 189,857 keys in any encoding on either side. 5,011 keys were paid to as P2PKH or P2WPKH both compressed and uncompressed. These counts cover public keys the scan found on chain by block 975,019, one per x-coordinate and matched in all six encodings; a key counts for a form if any output, spent or not, ever paid to it in that form.

Same-hash reuse was only measured on unspent outputs: 6,927 distinct 20-byte hashes, one serialized key each barring collisions, had unspent outputs under both P2PKH and P2WPKH, 93,972.89 BTC in all, which the P2PKH and P2WPKH rows above already include. For 3,301 of them, holding 49,154.61 BTC, the tables have no match, and a spend from either form would put the key on chain for both.

I built the tables and the UTXO census tool for the BIP 362 draft, but what they count differs from its `EXPOSED_AT_REST`. The differences run both ways, some come down to ambiguities in my own draft and several are unmeasured, so I can't say whether the draft's definition would count more or less. Section 11 of the extraction rules lists them, including cases the draft may not count (some table matches, outputs with no valid key): https://github.com/duncan0k/btc-exposure-index/blob/2a0f4b053cad2ae3dc101a4e4c154ac2b4706f25/docs/extraction-rules.md#11-relation-to-the-bip-362-draft

Some uncounted outputs are `UNDETERMINED` in the draft, so the uncounted set is not its `EXPOSED_ON_SPEND` set either. A key or script behind an uncounted output may still have been disclosed, on chain in a position the rules don't read or off chain. In a random sample of 250 uncounted outputs of each of P2PKH, P2WPKH, P2SH and P2WSH, none carried a scriptPubKey spent from at or before block 975,019 (one-sided 95% upper bound 1.19% for each type). It's a sample, not a full pass, checked through a public Esplora API, independent of the tables.

Code, extraction rules and result files are at https://github.com/duncan0k/btc-exposure-index (commit 2a0f4b0), definitions and verification steps at https://snapshots.ark0.si/method.html, and the tables in one archive of 33,654,024,835 bytes at https://snapshots.ark0.si/btc-exposure-index-975019.tar.zst. A full rescan with the repository's scanner reproduced the tables byte for byte in about 3.9 hours, reading only the blocks on a Bitcoin Core node's best-chain list. The UTXO figures need your own `dumptxoutset` at block 975,019, in the snapshot format version 2 that Bitcoin Core v28.1.0 writes.

If anything in the rules or counts looks wrong, that's useful to hear.

-------------------------

