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

Tsua00021 | 2026-03-28 12:05:26 UTC | #3

**Update — Validator in the playground + spec revision (BTSL v1.0)**

A short follow-up: the **Validator / Checker path** described in the spec is now implemented in the same client-side app alongside Maker.

**Playground** (`https://btsl-playground.vercel.app/`, repo `github.com/tsua0002/btsl-playground`)

* **Maker** (unchanged in intent): schema → bind **PARAMS** (including `.params`-style paste / file) → generate → export **unsigned PSBT** (base64/hex + QR).
* **Validator**: reuse the **same schema and confirmed PARAMS**, paste a PSBT — chain-backed checks, `calc` replay, and `ASSERT` evaluation (spec §9.3-style workflow).
* **Onboarding / examples**: curated examples stay **always available** (collapsible panel); demo **PARAMS** fixtures are called out explicitly; derived inputs using **`From(@PUBKEY)`** trigger **automatic UTXO resolution** after a valid pubkey is set (debounced), so the “simple payment” path is faster without extra clicks.

**Specification** (normative source in the standard repo, e.g. `btsl-spec-v1.0.md` + aligned `btsl-implementation-guide-v1.0.md`)

For implementers and reviewers, the v1.0 text on `master` is brought up to date with:

* **Lexer — `:`**: Normative split between **`SECTION_OPEN`** (colon that opens an indented block) vs **`COLON`** elsewhere; **exception**: after a numeric **input / output / `ASSERT` index**, the colon is always **`COLON`**, even when an indented line follows — removes ambiguity with `utxo:`, `witness_data:`, etc.
* **Outputs**: Positional rules for `output_type` / `amount` / `address_ref`; bare `snake_case_id` in first address position rejected outside `alias_ref`; lexer note on residual `IDENTIFIER` in `address_ref`.
* **`calc` / expressions**: Explicit arithmetic precedence (`*` `/` before `+` `-`); in **`primary`**, **`alias_ref` vs `value`** disambiguation with **single-token lookahead** on `.`.
* **`CONST`**: **Global (file)** vs **local (`PSBT_SCHEMA`)** scope; shadowing → **`BTSL_WARN_09`**; same-scope redeclaration → **`BTSL_ERR_00`**; §3.1 table and “one block per nesting level” guidance.
* **`witness_data` vs `witness:` (P2TR paths with a `witness:` block)**: **Name, order, and case** must match placeholders → **`BTSL_ERR_10`** on mismatch; **no** nominal `witness_data` check for P2WSH/P2SH paths **without** `witness:` placeholders; example **§6.4** (vault) aligned with that binding model.
* **§8 implementation checklist**: Updated (including P2TR witness rules and full **`BTSL_ERR_00`–`BTSL_ERR_10`** / **`BTSL_WARN_01`–`BTSL_WARN_09`** tables in §5.3–5.4).
* **Examples / fixtures** in the reference tree are tightened so they work as **repro and test vectors** where applicable.

I consider this **my finalized v1.0 spec pass** for public review; further changes would be explicit revisions / errata, not silent drift.

Feedback still welcome on edge cases, especially around Validator vs your own PSBT constructions and any parser ambiguities you hit with real schemas.

---

*Spec: `github.com/tsua0002/btsl-standard` · Playground: `github.com/tsua0002/btsl-playground`*

-------------------------

Tsua00021 | 2026-04-07 12:57:42 UTC | #4

**Update — BTSL v1.0 specification marked FINAL (checker completeness)**

Following the March update, I’ve **closed the remaining normative gaps on the Checker side** so an external verifier has an explicit, testable contract: not only replay of `calc` / `ASSERT`, but also **shape fast-fail**, **strict prevout value verification**, **outpoint binding** (parameters and workflow parent), **output `scriptPubKey` cross-check**, and a **complete `BTSL_ERR_00`…`BTSL_ERR_13`** mapping aligned with **§9.3.1** in `btsl-spec-v1.0.md`.

**Repository (normative source):** https://github.com/tsua0002/btsl-standard
**Anchored revision:** https://github.com/tsua0002/btsl-standard/commit/1e48a0a0fa91b1a18081e2d9df5a24844a8b8593 *(spec README: Reference Specification \[FINAL\])*

**Summary for implementers / reviewers:**

* **§9.3.1** — Predicate set **S-1…A-5** with phase order: parse → **shape (S-1/S-2 → `BTSL_ERR_13`)** → field-level (**I-1…I-4**, **O-1**, **O-2**) → algebraic (**A-1…A-5**).
* **I-3 / zero-trust** — **`BTSL_ERR_11`** if PSBT-declared input amount ≠ independently chain-fetched value for that outpoint.
* **I-2** — **`BTSL_ERR_12`** for bound-parameters outpoint mismatch (**Case A**) and for confirmed workflow parent vs wrong PSBT prevout (**Case C**, **§9.4**); **`BTSL_ERR_05`** when the parent is unavailable.
* **`BTSL_ERR_06`** — Explicitly includes **O-2** (PSBT output amounts vs `calc` / schema), not only ASSERT / implicit balance.
* **`From()`** — **§9.1.1** binding persistence (Level 1) for handoff to an external Checker.
* **`btsl-implementation-guide-v1.0.md`** — Checker steps reordered so **shape precedes** per-input chain work (consistent with §9.3.1).
* **`btsl-checker-predicates-v1.0.md`** — Consolidated predicate/error reference annex (normative text remains **§9.3.1**).

Further changes to v1.0 should be **explicit errata or a numbered revision**, not silent drift.

**Playground:** the public app remains a separate implementation; I am **aligning the client-side Validator** with this FINAL **§9.3.1** predicate set and will note when that release is tagged.

Feedback still welcome from wallet / PSBT implementers—especially real coordinator flows and **`.params` interoperability** across Maker and Checker builds.

-------------------------

Laz1m0v | 2026-04-15 23:33:36 UTC | #5

Hi Tsua. This is a highly relevant initiative. You have accurately identified one of the most significant vulnerabilities in multi-party PSBT workflows: the implicit trust placed in imperative coordinator logic.

To answer your questions regarding prior or similar work at the validation layer: we have been tackling this exact problem space through the PRECOP (Predictive Covenant Oracle Protocol) framework, albeit from a covenant and oracle derivation perspective rather than a pure wallet UX perspective.

While BTSL provides an excellent off-chain declarative schema for wallets, our recent work on PRECOP’s "Tier 1: Structural Enforcement" applies this exact philosophy to signing enclaves. We enforce a "Command-First Topology" (a strict output structure like [0: OP_RETURN, 1: Target, 2: Change, 3: Fee]) combined with exhaustive UTXO context hydration. If the PSBT deviates from this structural invariant, the oracle deterministically fails-closed and refuses to sign.

Your premise of shifting from constructing transactions to specifying them is the only viable path forward for secure, deterministic workflows. BTSL looks like a fantastic standardization for the off-chain/hardware wallet side of this equation, while frameworks like PRECOP/Simplicity enforce those schemas at the L1 execution layer.

We will be reviewing your EBNF grammar closely. Standardizing these schemas is a critical next step for the industry. Excellent work.

 laz1m0v

-------------------------

