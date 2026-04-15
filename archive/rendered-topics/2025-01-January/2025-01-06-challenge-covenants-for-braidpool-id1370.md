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

Laz1m0v | 2026-04-03 22:00:41 UTC | #3

This discussion around RCA / UHPO state aggregation is very interesting.

In our research group we have been exploring a related approach for modeling continuous-state UTXO machines without consensus covenants, by moving the enforcement to the signing layer.

The architecture we are experimenting with encodes the evolving state algebraically into a Taproot key tweak (what we call the Astrolabe pattern), while a local Simplicity VM evaluates the covenant logic before a Schnorr signature is produced.

In other words, the covenant is enforced client-side at signing time rather than by the network.

Obviously this does not provide the same trust model as consensus covenants or ANYPREVOUT, but it allowed us to experiment with continuous-state machines (CDPs, AMMs, etc.) on signet/mainnet without pre-signing large transaction trees.

I’m curious whether something like this could be useful as a research testbed for Braidpool-style payout orchestration before consensus primitives are available.

Would a “soft covenant” approach at the signing layer provide any useful insights for RCA/UHPO design, or do you see fundamental limitations that would make it irrelevant for this class of protocol?

-------------------------

AaronZhang | 2026-04-15 19:38:51 UTC | #4

**Challenge Follow up：Dynamic coinbase aggregation on Inquisition signet — multi-input APO with growing amounts (Braidpool RCA)**

Picking up from **[my reply in the Braidpool covenants challenge thread](https://delvingbitcoin.org/t/challenge-covenants-for-braidpool/1370/2)** (where I walked through how the covenant stack could meet the pool’s constraints), I later posted an on-chain **[Eltoo state chain demo](https://delvingbitcoin.org/t/eltoo-state-chain-on-signet-three-rounds-six-transactions-apo-ctv/2413)** as a *closed* system: one input per round, amounts only moving downward. After the Eltoo demo, the next question that kept bothering me was what happens once new coinbase keeps arriving and the pool actually grows? This is the motivating shape of Braidpool’s rolling coinbase aggregation.

I tried to answer that by actually constructing the transactions on-chain.

## What changed

Each aggregation round is now a **two-input** transaction:

* Input 0: RCA state UTXO — APO script-path (BIP 118, 3-leaf TapTree)
* Input 1: New coinbase — bare P2TR key-path (different address)

## The chain

Braidpool-realistic parameters (scale: 10,000 signet sats = 1 BTC; block subsidy 3.125 BTC):

```
Round 1: Miner A finds Block #1
  coinbase 32,750 sats (3.275 BTC) -> RCA v1
  UHPO v1: [A: 100%]

Round 2: Miner B finds Block #2
  coinbase 33,750 sats (3.375 BTC) + RCA v1 -> RCA v2   (2 inputs)
  Pool: 63,500 sats (6.350 BTC)
  UHPO v2: [A: 50%, B: 50%]

Round 3: Miner C finds Block #3
  coinbase 32,250 sats (3.225 BTC) + RCA v2 -> RCA v3   (2 inputs)
  Pool: 92,750 sats (9.275 BTC)
  UHPO v3: [A: 45%, B: 35%, C: 20%]

Settlement: CTV spend RCA v3 -> 3 outputs
  Miner A: 40,387 sats (4.039 BTC)
  Miner B: 31,412 sats (3.141 BTC)
  Miner C: 17,951 sats (1.795 BTC)
```

Same 31,250 sats subsidy every block; the three coinbase sizes above differ only by assumed per-block fees (1,500 / 2,500 / 1,000 sats).

All transactions confirmed on Inquisition 28.2 signet:

| Role | TxID | mempool.space |
|----|----|----|
| Block #1 coinbase fund | `7557c229...ea7f5f7c` | [link](https://mempool.space/signet/tx/7557c229f29771410307055fabf907e4263899c140ff1963a81868c2ea7f5f7c) |
| Block #2 coinbase fund | `600b9c7f...dc5bf063` | [link](https://mempool.space/signet/tx/600b9c7f990ce09b7bfc06406d9f9e8d75696f44b86069974abb81b5dc5bf063) |
| **Aggregation (R2)** | `094a8aa1...07d5ea9f` | [link](https://mempool.space/signet/tx/094a8aa1d572ac76d5f012ce6cb31f9cb9f4e982e9e90604e96b551c07d5ea9f) |
| Block #3 coinbase fund | `1bac7966...a3e1ea0c` | [link](https://mempool.space/signet/tx/1bac796632961eb18f8e50db0b643862cf7c8f6923fe27068a29b760a3e1ea0c) |
| **Aggregation (R3)** | `01a00ecb...b4193206` | [link](https://mempool.space/signet/tx/01a00ecba14dac6e60181f925bf2612dc9061d5791196a03270ccacfb4193206) |
| **CTV settlement** | `7669e251...0800de5a` | [link](https://mempool.space/signet/tx/7669e2519776a6af8ac3317ba64324282d3c14188256c0ef64dac5500800de5a) |

## Aggregation witness — byte-level analysis

The star transaction is **R2** (`094a8aa1...`). Two inputs, one output:

```
Input 0: RCA v1 (32,750 sats)  —  APO script-path spend
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Witness stack (3 elements):

  [0] Signature (65 bytes):
      0d3cbaf6d96431f4b8c90bfc5bb02580e2a293468366
      b8afe3de5bfa70b8b1d821ffe9a603f6b03dfd6ebe18
      8ecc546028d31f60041709cdd54c35da1fec6854 41
                                                 ^^
                                    SIGHASH_ALL | SIGHASH_ANYPREVOUT (0x41)
                                    This is BIP 118.

  [1] Tapleaf script (35 bytes):
      01 ff1f9fa326a9438227e6aa25030ccf89bcb8ce53db
         4f78dbce6146499d9986b8 ac
      ^^                        ^^
      BIP118 key prefix         OP_CHECKSIG
      (makes key APO-capable)

      Decoded: OP_PUSHBYTES_33
               01 ff1f9fa326a9438227...9d9986b8
               OP_CHECKSIG

  [2] Control block (97 bytes):
      c1                                                   ← version 1 | odd parity
      ff1f9fa326a9438227e6aa25030ccf89bcb8ce53db
      4f78dbce6146499d9986b8                               ← internal key (32B)
      b1a19aa6ef48350a32d697569b4a9dd05e17b76fb2
      e23282ce9726ff83788c9d                               ← sibling: CTV leaf hash
      2e97bfb37eb4707bb8c82a6afa241b4840b5dc5382
      596fed89e1590ec3bc3eec                               ← sibling: CSV leaf hash

      97 bytes = 1 (version) + 32 (key) + 32 + 32 = 3-leaf TapTree proof


Input 1: Block #2 coinbase (33,750 sats)  —  bare P2TR key-path
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Witness stack (1 element):

  [0] Signature (64 bytes):
      adaadef0832af80374c5f2e8796d0f6bd6342a2d1227
      eefdd57826f0322ed6e5558c6107d7e145dda9488c0b
      f0e52bb6e1662e88f97a83647501867fcd13ce68
      (no sighash suffix → default SIGHASH_ALL, standard Schnorr)

  Previous output: V1_P2TR (bare — no script tree)
  Address: tb1pamm9wlachp9pdgp0t6wlshnl...mqcv7wfw


Output: RCA v2 (63,500 sats)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Address: tb1pf86ekneawwfdq8u7e4tcjkw8qjp6qk8uudwcxcfw4n4uxawj3ppser0n8m
  New 3-leaf TapTree → next round's state
```

## Cross-round control block evolution

Between R2 and R3, what surprised me a bit is that the control block makes the state evolution very explicit — you can literally see which part changed across rounds:

```
R2: c1 ff1f9f...86b8  b1a19aa6...8c9d  2e97bfb3...3eec
R3: c1 ff1f9f...86b8  fc07e219...f751  2e97bfb3...3eec
    ^^                 ^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^
    same parity        CTV leaf CHANGED  CSV leaf SAME
```

| Element | R2 | R3 | Why |
|----|----|----|----|
| Version/parity | `c1` | `c1` | Both odd parity (coincidence of merkle root) |
| Internal key | `ff1f9f...86b8` | same | Same signer across rounds |
| Sibling 1 (CTV) | `b1a19a...8c9d` | `fc07e2...f751` | **Changed** — UHPO ratios evolved (50/50 → 45/35/20) |
| Sibling 2 (CSV) | `2e97bf...3eec` | same | **Unchanged** — escape hatch is a protocol constant |

As expected in hindsight, the CTV leaf hash changes every round as the payout template evolves. The CSV escape hatch, on the other hand, stays completely unchanged — which is exactly what you’d want from a protocol constant.

## CTV settlement

The final transaction (`7669e251...`) spends RCA v3 via the CTV leaf:

```
Input 0: RCA v3 (92,750 sats)  —  CTV script-path spend
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  P2TR tapscript:
      OP_PUSHBYTES_32 ee9a9088aecc7bd5df75a7dbe80d2031
                      c1f02e87a84fa330d9d696ccb6d2b866
      OP_NOP4
      ^^^^^^^^^^^^
      = OP_CHECKTEMPLATEVERIFY (opcode 0xb3)

  nSequence: 0xffffffff  (CTV requires this)

  The template hash commits to exactly 3 outputs at exactly these amounts.
  Any deviation → script fails → transaction invalid.

Output 0: 40,387 sats → Miner A  (45% of pool - fees)
Output 1: 31,412 sats → Miner B  (35% of pool - fees)
Output 2: 17,951 sats → Miner C  (20% of pool - fees)
```

## The sighash detail that matters

BIP 118 Msg118 for input 0 includes `sha_amounts` and `sha_scriptpubkeys` covering **all** inputs — including the coinbase from a completely different address. If the coinbase input’s scriptPubKey is missing from the array, the sighash digest is wrong and the signature is invalid.

I didn’t expect this at first, but dynamic aggregation turns out to be trickier than the single-input Eltoo case: every additional input changes the sighash context for the APO-signed input.

## What this proves beyond the previous post

| Property | Previous (Eltoo chain) | This experiment |
|----|----|----|
| Inputs per round | 1 (same address) | 2 (different addresses) |
| Amount trend | Decreasing | Increasing |
| Input scriptPubKeys | Homogeneous | Heterogeneous |
| Pool pot | Static | Grows: 32k → 63k → 92k |
| UHPO allocation | Fixed ratios | Evolving: 100% → 50/50 → 45/35/20 |
| Braidpool applicability | LN-Symmetry analog | Dynamic coinbase aggregation |

## What it doesn’t prove (yet)

* 100-block coinbase maturity (wallet-funded proxies — addressed in follow-up)
* Solo-mining fallback (no CSV leaf on coinbase — addressed in follow-up)
* FROST federation threshold signing (single key in demo)
* Pre-signing (APO allows it; demo signs sequentially)

## Combined coverage for Braidpool’s covenant layer

With the CSFS equivocation penalty from our previous experiment:

| Building block | Opcode | On-chain |
|----|----|----|
| RCA Eltoo state chain | APO | yes |
| Dynamic coinbase aggregation | APO | yes (this post) |
| UHPO deterministic payout | CTV | yes |
| Signer accountability bond | CSFS | yes |

The unsolved parts of mcelrath’s [Braidpool challenge](https://delvingbitcoin.org/t/challenge-covenants-for-braidpool/1370/2) (maturity handling, emergency broadcast, solo-mining fallback) are protocol-layer constraints, not signing-primitive gaps. The covenant building blocks are individually validated.

## Links

* Previous: [BIP 118 signing from scratch — on-chain rebinding proof](https://delvingbitcoin.org/t/bip-118-signing-from-scratch-on-chain-rebinding-proof/2411)
* Eltoo state chain: [Three rounds, six transactions (APO + CTV)](https://delvingbitcoin.org/t/eltoo-state-chain-on-signet-three-rounds-six-transactions-apo-ctv/1407)
* Braidpool challenge: [Challenge: Covenants for Braidpool](https://delvingbitcoin.org/t/challenge-covenants-for-braidpool/1370/2)
* Code: [btcaaron](https://github.com/aaron-recompile/btcaaron) | Experiment: `experiments/Braidpool/rca_dynamic_chain.py`

-------------------------

