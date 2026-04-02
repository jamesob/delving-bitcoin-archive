# Challenge: Covenants for Braidpool

mcelrath | 2025-01-06 20:20:08 UTC | #1

I present a challenge for all the covenant aficionados out there, and your favorite opcode. I'm looking for specific covenant proposals which accomplish the following:

## Background

[Braidpool](https://github.com/braidpool/braidpool) (A decentralized mining pool) will need to custody coinbase rewards across multiple blocks. For this I have proposed using a FROST federation which signs two chained transactions RCA and UHPO for this:

Coinbase -> RCA -> UHPO

[Rolling Coinbase Aggregation (RCA)](https://github.com/braidpool/braidpool/blob/6bc7785c7ee61ea1379ae971ecf8ebca1f976332/docs/braidpool_spec.md#payout-update): This is an Eltoo transaction on-chain that aggregates the most recent coinbase with the previous (aggregated) coinbase(es).

[Unspent Hasher Payout Object (UHPO)](https://github.com/braidpool/braidpool/blob/6bc7785c7ee61ea1379ae971ecf8ebca1f976332/docs/braidpool_spec.md#unspent-hasher-payment-output): One or more transactions that spends the RCA and pays all hashers in proportion to the work contributed. This transaction is timelocked and cannot be broadcast until the end of a difficulty adjustment window (the "settlement period"), at which time the most recently created UHPO is broadcast (with all previous UHPOs invalidated by the Eltoo mechanism of its parent RCA) and pays all miners for the preceding settlement period.

I have proposed using a [FROST federation](https://github.com/pool2win/frost-federation) in development by @jungly for this. Our construction has the following properties:

1. The UHPO transaction is constructed and signed or committed to first. All federation signing nodes can independently construct and verify the payouts in the UHPO transaction because they each have a copy of the Braidpool DAG with share information.
2. The RCA transaction is signed or committed to after the UHPO transaction is signed by a quorum of federation nodes.
3. The Byzantine fault tolerant signing quorum is 3f+1 or 67% of the signing nodes.

The FROST Federation has a number of drawbacks, not the least of which is a 51% attack (or 67% attack) on the pool, if federation membership is decided by PoW, which could steal all funds. If federation membership is decided by some other manner this results in a highly political process at odds with the decentralized nature of Bitcoin, and theft is still a possibility. I wish to consider a covenant-based alternative which eschews the need for custody entirely, or achieves a "can't-be-evil" philosophy where only the correct payouts can happen.

## Requirements

1. The RCA transaction must either: be aggregated with another coinbase to create a new RCA transaction (Eltoo) OR (after a timelock) be spent in a UHPO transaction. It must not be spendable in any other way.
2. A block with an incorrect payout not satisfying the requirements of the RCA or UHPO must be spendable (after a possible timelock) as solo-mined block, and will not be considered a "share" by the pool.
3. Coinbase outputs must have a timelock that falls back to solo mining, so that in the event of a pool failure (RCA not constructed/mined), all miners with blocks not aggregated in the RCA can claim their blocks as solo mined blocks.
4. Each new block found by the pool modifies the set of payouts in the next UHPO, so the UHPO from the previous block can't know the correct payouts in the next block and can't commit to it.
5. In the event of an incorrect RCA or UHPO transaction or 51%/67% attack on the FROST federation signing, and a theft attempt is broadcast or otherwise detected against the RCA or UHPO, the "last-known-good" UHPO transaction must become immediately broadcastable, evading its usual timelock, and ensuring that the RCA Eltoo mechanism is invalidated for future blocks. The pool becomes entirely solo-mining or shuts down in this case.
6. Assume that information necessary to construct the UHPO is known at the time a block template is constructed, and can be committed to in a block. (It's the INPUT to this UHPO tx that requires custody across multiple blocks)
7. You must take into account the 100 block coinbase maturity rule.

A couple suggestions:
1. You MAY choose that the UHPO has all the same outputs as the previous UHPO, with amounts that are greater than or equal to the previous UHPO, with additional outputs added.
2. You MAY spend the entire RCA to fees if it is included in a pool's block, in order to get rid of the RCA and have the UHPO directly committed to by the coinbase output. (As long as this can't be stolen by another miner)
3. You MAY eschew the RCA mechanism in favor of taking all coinbase outputs from the previous settlement period as inputs.

Something deviating from the above outlined structure is welcome as well, if it achieves the goal of ensuring that everyone gets paid correctly and uses covenants.

A proof of impossibility would be welcome as well.

P.S. I have avoided describing other approaches that we might take here to keep it concise. (e.g. the P2Pool/Eligus/OCEAN mechanism of paying PPLNS in coinbases -- we can always fall back on that but it's been done has well known drawbacks) If that info would be helpful I can write it up in a separate post or doc on the Braidpool github. (msg me privately if that is of interest)

-------------------------

AaronZhang | 2026-04-02 18:28:40 UTC | #2

Hi @mcelrath,

This suggests that Braidpool-style rolling aggregation can be implemented as a sequence of per-round covenant updates, rather than requiring a single persistent commitment across blocks.

## The key insight on Req 4

Req 4 looks like a contradiction — CTV commits to a static hash, but UHPO payouts change every block. It dissolves when you treat each APO replacement as atomic with its CTV update:

* **APO handles the input side**: the update signature doesn't commit to the previous RCA's txid, so you can pre-sign the next state before the current one hits the chain.

* **CTV handles the output side**: each new RCA commits to the current period's UHPO template. When the old RCA gets spent, its UHPO loses its input and can never be broadcast. Only the latest RCA survives, so only the latest UHPO can settle.

Dynamic payouts through a sequence of per-round static commits. APO and CTV fire together on every update — no single commitment needs to span multiple blocks.

## The RCA script shape

Each RCA output is a three-leaf Taproot tree:

```
leaf 1 — ctv_uhpo:   OP_CHECKTEMPLATEVERIFY
                      locks spend to the committed UHPO template

leaf 2 — apo_update: <0x01||xonly_pubkey> OP_CHECKSIG  (BIP118)
                      next RCA can replace this one without knowing the txid

leaf 3 — csv_escape: <N> OP_CSV OP_DROP <pubkey> OP_CHECKSIG
                      timeout fallback for solo mining recovery

```

## Demo 1 — APO rebinding

Two UTXOs, same script, same amount, different txids. One APO signature spends both — witness bytes are byte-for-byte identical across the two spend transactions, only the prevout differs. This is what makes pre-signing the next RCA state possible before the current one hits the chain.

|  | TxID |
|----|----|
| Spend UTXO₁ (signature produced here) | [03c0…f3a4](https://mempool.space/signet/tx/03c0577c1d47da32804d098187644d0eee18b448aded2f427cd02193c070f3a4) |
| Spend UTXO₂ (same witness, different prevout) | [4609…1a43](https://mempool.space/signet/tx/46091190c74d8fd4b39be67a2e945a19b021850e7f8d9e378f5eb11722ae1a43) |

The two spend txs look identical on the surface (same script, same amount, same output address) — that's by design. The proof is in the input: each spends a different prevout, but the witness bytes are identical. SIGHASH_ANYPREVOUT does not commit to the outpoint, so the same signature validates against either UTXO.

**Witness:** three stack items (BIP118 signature + `0x01‖xonly` `OP_CHECKSIG` leaf + control block). **Stacks are byte-for-byte identical** on both spends; only `vin` outpoints differ.

## Demo 2 — RCA Eltoo chain (three rounds + settlement)

```
Fund → RCA v1 (50/30/20) → RCA v2 (45/35/20) → RCA v3 (40/35/25) → UHPO settlement

```

| Role | TxID |
|----|----|
| Fund → RCA v1 | [386d…9c41](https://mempool.space/signet/tx/386dbb6aa23fcc35a69d34e3c0f760b185482467abc936196d3def19d54a9c41) |
| APO: v1 → v2 | [096e…5601](https://mempool.space/signet/tx/096e31ccbd8f5460b2730ec4f757ee1b01acf9dde3e3b8cb55fbb534fe195601) |
| APO: v2 → v3 | [0913…d7f6](https://mempool.space/signet/tx/091309b73e299436ff12fceda7e54e28c6f1817fdf3405bca3343ad15198d7f6) |
| CTV settlement (UHPO v3) | [1395…f9b7](https://mempool.space/signet/tx/13957f49d01aa21ed2aa28df7aa78f357d51a2a2fbfb434af05fdc75ad0ff9b7) |

**Witness — APO updates (v1→v2, v2→v3):** same three-part shape each time (BIP118 signature + leaf + control block), but each step uses a new signature (new state).

**Witness — CTV settlement:** two elements — 32-byte template hash push + control block. No Schnorr signature. `nSequence` is `0xffffffff`. Explorers show the opcode as `OP_NOP4`.

## Requirements mapping

| Req | Status | Note |
|----|----|----|
| 1 | ✓ | Three leaves are the only spending paths |
| 2 | partial | csv_escape leaf exists; "incorrect block → solo claim" not fully demoed |
| 3 | partial | CSV timeout present; demo uses normal UTXOs not real coinbase |
| 4 | ✓ | APO + CTV atomic update dissolves the dynamic commitment problem |
| 5 | open | See below |
| 6 | ✓ | UHPO template committed into RCA at block template construction time |
| 7 | partial | Real coinbase maturity needs extra timelock layering |

## Trade-offs

* Each pool aggregation round that advances the UHPO snapshot uses its own on-chain RCA update (one APO spend) — one covenant step, one update tx.

* On-chain footprint scales with how many rounds you run.

* Complexity shifts from script expressiveness to transaction orchestration outside the script.

## On Req 5

No solution here. "Detect an attack and immediately unlock" requires the script to observe external chain state — none of the current opcodes can do this. CSFS can verify an authorization signature, but whoever holds that key becomes a new trusted party, reintroducing the centralization you're trying to remove. A minimal additional primitive enabling conditional unlock based on external chain state (something like OP_VAULT's trigger mechanism) would close this gap.

## Note on FROST

Demo uses a single key where production Braidpool would use a FROST aggregate — on-chain they look identical (64-byte Schnorr signature either way). The covenant layer limits FROST's role: whatever key signs the APO update, the CTV commitment means they can only produce a valid UHPO. FROST goes from "decides payouts" to "triggers the update."

---

Curious whether others see a path to Req 5 without introducing new trust assumptions.

Code: [github.com/aaron-recompile/btcaaron](https://github.com/aaron-recompile/btcaaron) (`examples/braidpool/`)

-------------------------

