# Eltoo State Chain on Signet: Three Rounds, Six Transactions (APO + CTV)

AaronZhang | 2026-04-13 05:49:35 UTC | #1

In a previous post ([BIP 118 signing from scratch + on-chain rebinding proof](https://delvingbitcoin.org/t/bip-118-signing-from-scratch-on-chain-rebinding-proof/2411)) I demonstrated an independent Python implementation of the BIP 118 sighash, and two confirmed transactions that demonstrate the rebinding property. That post covered the signing primitive. This one covers what we can build with it.

I ran a three-round Eltoo-style state chain on Inquisition 29.2 signet. 

**Construction**

Each state UTXO is a three-leaf Taproot tree:

> leaf 1 — ctv_settle:  <template_hash> OP_CHECKTEMPLATEVERIFY
>
> leaf 2 — apo_update:  <0x01 || xonly_pubkey> OP_CHECKSIG   (BIP 118)
>
> leaf 3 — csv_escape:  <N> OP_CSV OP_DROP <pubkey> OP_CHECKSIG

APO leaf — update path. The signature doesn't commit to the prevout outpoint (sha_prevouts is absent from Msg118). It still commits to outputs — you can rebind the input, not the destination.

CTV leaf — settlement path. Each state's payout distribution is locked as a template hash. When an APO update spends state vN, vN's CTV leaf loses its only viable input and can never settle. Only the latest surviving state produces a valid settlement. This is the Eltoo invariant.

**The chain**

> Fund → State v1 (50k / 30k / 20k)
>
>      → State v2 (45k / 35k / 20k)    ← APO update
>
>      → State v3 (40k / 35k / 25k)   ← APO update
>
>      → Settlement                          ← CTV spend, empty witness

| Role             |   TxID            |

|------------------|-------------------|

| Fund → State v1 | \`[386d…9c41](https://mempool.space/signet/tx/386dbb6aa23fcc35a69d34e3c0f760b185482467abc936196d3def19d54a9c41)\` |

| APO: v1 → v2 | \`[096e…5601](https://mempool.space/signet/tx/096e31ccbd8f5460b2730ec4f757ee1b01acf9dde3e3b8cb55fbb534fe195601)\` |

| APO: v2 → v3 | \`[0913…d7f6](https://mempool.space/signet/tx/091309b73e299436ff12fceda7e54e28c6f1817fdf3405bca3343ad15198d7f6)\` |

| CTV settlement | \`[1395…f9b7](https://mempool.space/signet/tx/13957f49d01aa21ed2aa28df7aa78f357d51a2a2fbfb434af05fdc75ad0ff9b7)\` |

Plus two APO rebinding transactions — same script, same amount, different txids, witness bytes identical:

| Role                 | TxID                        |

|---------------------|----------------------------|

| Spend A (signed) | \`[03c0…f3a4](https://mempool.space/signet/tx/03c0577c1d47da32804d098187644d0eee18b448aded2f427cd02193c070f3a4)\`   |

| Spend B (witness copied) | \`[4609…1a43](https://mempool.space/signet/tx/46091190c74d8fd4b39be67a2e945a19b021850e7f8d9e378f5eb11722ae1a43)\` |

Note on pre-signing: the demo runs sequentially — each round waits for the previous state to confirm before constructing the next signature. APO's sighash *allows* pre-signing (the signature is prevout-independent), and the rebinding proof confirms this directly: one signature spent two different UTXOs without re-signing. Pre-signing is a strict subset of that property.


 **What this demonstrates**

\- APO signature rebinding works as specified on Inquisition

\- The Eltoo state update sequence executes end-to-end with current tooling

\- CTV settlement integrates cleanly as the terminal path — empty witness, nSequence = 0xffffffff

\- An independent Python Msg118/Ext118 passes Inquisition's consensus validation (cross-validation between two codebases)


**What it doesn't**

\- Full LN protocol stack (BOLT messaging, two-node communication, HTLC routing)

\- Penalty path for old-state broadcast (csv_escape leaf exists but wasn't exercised)

\- FROST aggregation for the signing key

Sanders's LN-Symmetry CLN proof-of-concept covers the protocol layer. This covers the execution layer underneath.


 **The CTV+CSFS gap**

This construction uses APO. The CTV+CSFS path to the same Eltoo primitive requires the spender to carry the CTV hash in witness data and commit to it via CSFS — more bytes, more moving parts, and no on-chain demonstration exists yet.

The opcodes are activated on Inquisition. The tooling exists. If CTV+CSFS is being proposed as sufficient for LN-Symmetry, the execution-layer evidence should exist alongside the claims.

 **Links**

\- Braidpool application: \[Challenge: Covenants for Braidpool\]([challenge-covenants-for-braidpool](https://delvingbitcoin.org/t/challenge-covenants-for-braidpool/1370/2))

\- Code: \[btcaaron\]([https://github.com/aaron-recompile/btcaaron](https://btcaaron)) (\`examples/braidpool/\`)

-------------------------

cdecker | 2026-04-13 10:57:41 UTC | #2

I don’t have anything smart to say, but I wanted to thank you for pushing forward with this research. I had mostly given up on APO or any other combination of opcodes landing, so seeing this moving again is amazing. Thank you so much :folded_hands:

-------------------------

AaronZhang | 2026-04-13 15:47:57 UTC | #3

Thanks, Christian — and for the eltoo work as well. This line of experiments was very much inspired by it. The CSV unilateral path is included but not exercised yet. Happy to extend the demo if you have thoughts on the timeout side.

-------------------------

