# BIP 118 signing from scratch + on-chain rebinding proof

AaronZhang | 2026-04-11 18:51:47 UTC | #1

## BIP 118 signing from scratch + on-chain rebinding proof

Following up on @ajtowns’ survey of [CTV, APO, CAT activity on signet](https://delvingbitcoin.org/t/ctv-apo-cat-activity-on-signet/1257) — here is an additional data point: an independent Python implementation of the BIP 118 sighash, and two confirmed transactions that demonstrate the rebinding property.

### What I did

I implemented `Msg118` / `Ext118` / the full `TaggedHash("TapSighash", ...)` digest from the [BIP 118 spec](https://github.com/bitcoin/bips/blob/eb497966e4a9eb03030a2d3308fab557311185bc/bip-0118.mediawiki) in Python, outside of Bitcoin Core’s test framework. The signing side constructs the digest; Inquisition’s C++ consensus engine validates it independently. If my implementation disagrees with theirs, the transaction gets rejected.

Source: [`btcaaron/bip118.py`](https://github.com/aaron-recompile/btcaaron/blob/180935dc864c9a36dd725cf8f54a43d183c17b1a/btcaaron/bip118.py)

### What BIP 118 changes in the sighash

The whole point of BIP 118 is what `Msg118` *omits*. Here’s what goes into the digest vs standard BIP 342:

| Field | BIP 342 (`SigMsg`) | BIP 118 (`Msg118`, 0x41) |
|----|----|----|
| nVersion, nLocktime | yes | yes |
| sha_prevouts | **yes** | **no** |
| sha_sequences | **yes** | **no** |
| input amount | yes | yes |
| input scriptPubKey | yes | yes |
| sha_outputs | yes | yes |
| tapleaf_hash | yes | yes |
| key_version | 0x00 | **0x01** |

The outpoint (`txid:vout`) drops out of the digest. As long as the spent output has the same amount and scriptPubKey, and the transaction produces the same outputs, the signature verifies.

In code, the APO branch of `msg118`:

```python
apo_mode = hash_type & 0xC0

if apo_mode == SIGHASH_ANYPREVOUT:      # 0x40
    msg += amounts[txin_index].to_bytes(8, "little")
    msg += serialize_spk(script_pubkeys[txin_index])
    # no sha_prevouts, no sha_prevoutscripts, no sha_sequences
elif apo_mode == SIGHASH_ANYPREVOUTANYSCRIPT:  # 0xC0
    pass  # omit amount and scriptPubKey too
```

And `ext118` uses `key_version = 0x01` instead of BIP 342’s `0x00` — this separates the sighash domains so a BIP 118 signature can’t be replayed against a standard BIP 342 key even if the pubkey bytes match:

```python
ext += bytes([0x01])           # BIP118 key_version
ext += (0xFFFFFFFF).to_bytes(4, "little")  # codesep_pos
```

Final digest: `TaggedHash("TapSighash", 0x00 || Msg118 || Ext118)`.

### On-chain proof: rebinding

I funded two UTXOs with the same amount (50,000 sats) to the same P2TR address. The single-leaf tapscript is `<0x01||xonly> OP_CHECKSIG`. I signed a spend for the first UTXO, then copied the entire witness — all three stack items — onto a second transaction spending the other UTXO. No re-signing. Both broadcast, both confirmed in the same block.

|  | TxID | Prevout |
|----|----|----|
| Spend A (signed) | [`03c0…f3a4`](https://mempool.space/signet/tx/03c0577c1d47da32804d098187644d0eee18b448aded2f427cd02193c070f3a4) | `4b64…344a:0` |
| Spend B (reused witness) | [`4609…1a43`](https://mempool.space/signet/tx/46091190c74d8fd4b39be67a2e945a19b021850e7f8d9e378f5eb11722ae1a43) | `543c…a5eb:0` |

Both in block 298,280 on Inquisition signet. Decode both and compare — the witness stacks (65-byte signature ending in `0x41`, 35-byte leaf script, 33-byte control block) are byte-for-byte identical. Only `vin[0].prevout` differs.

The signature still commits to outputs (`sha_outputs` is in `Msg118` for `SIGHASH_ALL`). I verified that changing the output address or amount in Spend B causes script verification failure — rebinding lets you swap the input, not the destination.

### Cross-validation

The fact that these transactions confirmed means two independent implementations agree on the digest:

* **Signing side** (Python): constructs `Msg118 || Ext118`, hashes, signs with BIP 340
* **Validation side** (Inquisition C++): reconstructs the same digest from the transaction, verifies the Schnorr signature

Bitcoin Core’s wallet and CLI do not support constructing BIP 118 signatures — there is no `signrawtransaction --sighash=anyprevout`. The signing had to be done externally.

### Where this goes next

I used this signing code to build an Eltoo-style three-round state chain (APO updates + CTV settlement) for a Braidpool covenant demo — six transactions, all confirmed on the same signet. Details and txids are in my post on the Braidpool covenant challenge: [https://delvingbitcoin.org/t/challenge-covenants-for-braidpool/1370/2](https://delvingbitcoin.org/t/challenge-covenants-for-braidpool/1370/2).

### Code

* BIP 118 sighash: [`btcaaron/bip118.py`](https://github.com/aaron-recompile/btcaaron/blob/180935dc864c9a36dd725cf8f54a43d183c17b1a/btcaaron/bip118.py)
* Tests (digest invariance, signing): [`tests/test_bip118.py`](https://github.com/aaron-recompile/btcaaron/blob/180935dc864c9a36dd725cf8f54a43d183c17b1a/tests/test_bip118.py)
* Rebinding demo: [`examples/braidpool/rca_taptree_smoke.py`](https://github.com/aaron-recompile/btcaaron/blob/180935dc864c9a36dd725cf8f54a43d183c17b1a/examples/braidpool/rca_taptree_smoke.py)
* Eltoo chain demo: [`examples/braidpool/rca_eltoo_chain.py`](https://github.com/aaron-recompile/btcaaron/blob/180935dc864c9a36dd725cf8f54a43d183c17b1a/examples/braidpool/rca_eltoo_chain.py)

Happy to answer questions about the sighash construction. If anyone spots a divergence from the spec I’d like to know.

-------------------------

