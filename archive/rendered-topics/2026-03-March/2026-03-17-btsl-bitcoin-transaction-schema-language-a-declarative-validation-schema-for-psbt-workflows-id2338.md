# BTSL (Bitcoin Transaction Schema Language): A Declarative Validation Schema for PSBT Workflows

Tsua00021 | 2026-03-17 14:38:00 UTC | #1

Dear Community,

PSBT standardizes how unsigned transactions are exchanged between participants, but it does not define a way for signers to independently verify that a transaction satisfies the economic rules expected by the protocol or workflow they agreed on.

In practice, multi-party PSBT workflows (e.g., marketplaces, batch payouts, shared expenses) often rely on imperative coordinator logic: calculating change amounts, ensuring output equilibrium, and managing fee bumping across multiple inputs. This logic is usually implemented in host software that signers must implicitly trust.

This post explores an experimental declarative schema for PSBT validation that I have been experimenting with: **BTSL (Bitcoin Transaction Schema Language)**. It shifts from *constructing* transactions to *specifying* them.

> **BTSL introduces no new consensus rules and does not modify the PSBT format. It is purely a validation layer operating above BIP174/BIP370.**

---

## The “Tri-Count” Problem (A concrete example)

Consider three people who went on a trip. They want to settle debts in a single transaction rather than multiple txs. They have different UTXO types (P2WPKH, P2TR), they need to pay a platform maker fee, and they need to balance the settlement.

Instead of writing custom imperative code, the settlement is defined as a declarative schema:

```
; ==============================================================================
; BTSL v1.0 - Reference Implementation: TRICOUNT
; This version accepts both P2WPKH and P2TR inputs indifferently.
; The wallet inspects the scriptPubKey of the UTXO at injection time.
; ==============================================================================

CONST:
    DUST_LIMIT = 546

PSBT_SCHEMA TRICOUNT:
    PARAMS:
        @BOB_UTXO:UTXO
        @CARO_UTXO:UTXO
        @ALICE_ADDRESS:ADDRESS
        @MAKER_ADDRESS:ADDRESS
        @FEE_RATE:FEERATE
        @MAKER_FEE:SATOSHI
        @MEAN:SATOSHI
        @A2:SATOSHI  ; Initial amount Bob paid
        @A3:SATOSHI  ; Initial amount Caro paid

    INPUTS:
        0:
            utxo: @BOB_UTXO
        1:
            utxo: @CARO_UTXO

    OUTPUTS:
        0: @ALICE_ADDRESS payment sats
        1: CHANGE @BOB_UTXO.address c_bob sats
        2: CHANGE @CARO_UTXO.address c_caro sats
        3: @MAKER_ADDRESS maker_fee_val sats

    calc:
        payment       = (2 * @MEAN) - @A2 - @A3
        fees_sats     = vSize(CURRENT_PSBT) * @FEE_RATE
        total_fees    = @MAKER_FEE + fees_sats
        maker_fee_val = @MAKER_FEE
        c_bob         = REF(@BOB_UTXO.amount) - (@MEAN - @A2) - (total_fees / 2)
        c_caro        = REF(@CARO_UTXO.amount) - (@MEAN - @A3) - (total_fees / 2)

    ASSERT:
        0: c_bob >= DUST_LIMIT
        1: c_caro >= DUST_LIMIT
        2: payment > 0
        3: REF(@BOB_UTXO.amount) >= (@MEAN - @A2) + (total_fees / 2)
        4: REF(@CARO_UTXO.amount) >= (@MEAN - @A3) + (total_fees / 2)
```

---

## Why this matters

**1. Separation of Concerns.** The validation logic (`calc` and `ASSERT` blocks) is decoupled from the PSBT construction. The schema acts as a specification describing the expected transaction invariants, while the PSBT remains the transaction artifact.

**2. Independent Verification.** The schema acts as a formal implementation guide. Any signer wallet — or external auditor — can re-run the `calc` logic against actual PSBT data and referenced UTXOs to verify that output values satisfy the declared invariants. The `ASSERT` block acts as a deterministic validator: if the PSBT diverges from these invariants, signing can be refused before any key material is used.

**3. Structural Role Binding.** Inputs are deterministically associated with declared roles in the schema, reducing the risk of unintended input substitution or coordinator-side manipulation in multi-party workflows.

**4. Hardware wallet applicability.** One possible application is enabling hardware wallets to verify transaction invariants independently of the host software before signing — without having to trust the coordinator’s construction logic.

---

## Relationship with existing tools

This is the question I expect first, so I will address it directly.

| Tool | Layer | Purpose |
|:---|:---|:---|
| Script / Miniscript | Consensus | Defines spending conditions |
| Descriptors | Wallet | Defines output templates |
| PSBT (BIP174/370) | Signing workflow | Standardizes unsigned transaction exchange |
| **BTSL** | Validation | Specifies expected transaction structure and economic invariants |

BTSL does not replace or compete with any of these. It operates above them. A BTSL schema is not a spending condition — it is a description of what a valid PSBT *should look like* before anyone signs it.

---

## Security & Limitations

The specification addresses the independent verification flow and formalizes structural role binding.

Note that, absent `BIP118` (SIGHASH_ANYPREVOUT), this approach relies on structural dependencies (`DEPENDS_ON`) which are subject to txid mutation if the parent transaction is replaced via RBF. This is treated as an accepted structural risk for v1.0, to be mitigated by child-anchoring and CPFP.

This is an **experimental prototype**, not a finished standard. The goal of publishing the specification, grammar (EBNF), and test vectors is to explore whether a declarative validation layer for PSBT workflows could be useful in practice.

Full details: [tsua0002/btsl-standard: Bitcoin Transaction Schema Language (BTSL) Standard](https://github.com/tsua0002/btsl-standard)

---

## Questions for discussion

* Would a declarative transaction validation schema be useful for hardware wallet implementations?
* Are there existing tools attempting something similar at the validation layer?
* Is there prior work addressing transaction invariant verification at the PSBT layer?
* Could `PSBT_GLOBAL_UNKNOWN` fields be a viable transport for attaching schemas to PSBTs?
* Would wallet developers find value in deterministic, schema-driven fee and change validation for multi-party workflows?

-------------------------

Tsua00021 | 2026-03-19 13:56:33 UTC | #2

**Update: Interactive Playground Available**

I’ve published a client-side playground that implements the **Maker pipeline** from the BTSL v1.0 specification:

**Live demo:** https://btsl-playground.vercel.app/

**Source:** [github.com/tsua0002/btsl-playground](https://github.com/tsua0002/btsl-playground)

The playground walks through the full construction pipeline: 

Schema Input → Parameter Binding → Code Generation → PSBT Output. It runs entirely in the browser — no server, no private keys.

Built-in examples cover the five normative use cases from the spec: simple payment via `From(@PUBKEY)`, TRI-COUNT shared settlement, P2WSH multisig (1-of-2, 2-of-2), OP_RETURN embedding, and the timelocked Taproot vault workflow with `DEPENDS_ON`.

You can paste any BTSL schema, bind parameters against live mainnet UTXOs (Blockstream API) and fee rates (Mempool.space), then export the unsigned PSBT for signing in Sparrow, Coldcard, bitcoin-cli, or any BIP174-compatible tool.

This is a **preview release** covering the Maker (construction) side. The **Validator pipeline** — with Zero-Trust UTXO restoration and independent ASSERT replay — is under active development and will be added shortly.

Feedback welcome, particularly on the developer experience and any edge cases you encounter with the examples.

-------------------------

