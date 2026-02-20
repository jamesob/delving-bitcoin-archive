# A quantum resistance script only using op_ctv/op_txhash and no new signatures

simul | 2025-12-21 00:51:45 UTC | #1

# Anchor-gated, UTXO-moving, template-bound spend

## using OP_TXHASH + OP_CTV with an escape hatch

*(prunable-friendly; quantum-resilient to signature forgery)*

---

## Assumptions

* **OP_CHECKTEMPLATEVERIFY (OP_CTV)** is available per BIP119.
  The 32-byte template hash is `DefaultCheckTemplateVerifyHash`.
  https://bips.dev/119/

* **OP_TXHASH / OP_CHECKTXHASHVERIFY** is available per the current draft proposal,
  allowing scripts to hash and verify selected fields of the *spending transaction*
  without committing to a full transaction template.

* **Relative timelocks** exist (BIP68 / BIP112).

* **SHA256 preimage resistance holds**, even if ECDSA/Schnorr signatures become forgeable.

---

## Threat model

An attacker may:

* Forge signatures.
* Intercept, delay, reorder, or fee-bump transactions.
* Front-run in the mempool.
* Exploit shallow reorgs.

An attacker may **not**:

* Break SHA256 preimage resistance.
* Violate OP_CTV semantics.
* Violate OP_TXHASH semantics.
* Violate relative timelock rules.
* Rewrite deep chain history.

---

## High-level idea

This construction creates a **multi-phase envelope** that separates:

* *who can trigger execution* from
* *where value is allowed to go*.

Even if signatures are forgeable, funds can only move into a protected
Anchor envelope, and from there only along template-bound paths.

* **Phase 0** funnels all value into a predetermined Anchor envelope.
* **Phase 1** instantiates that envelope on-chain.
* **Phase 2** either:
  * reveals a one-time secret to complete a template-bound spend, or
  * uses an escape hatch without revealing the secret.

At no point does Phase 0 commit to final recipients.

---

## Data definitions

* **x**: one-time secret (recommended 32 bytes).

* **C = SHA256(x)**.

* **k**: `uint32` relative confirmation depth parameter.

* **T**: 32-byte CTV template hash for the intended reveal spend.

* **E**: 32-byte CTV template hash for the escape-hatch spend.

* **P_anchor**: Taproot output key committing to an Anchor script tree that:

  * embeds `C`,
  * enforces *reveal-or-escape* spending conditions,
  * optionally enforces a relative timelock `k` on the reveal path.

---

## Transactions and scripts

### Phase 0: Funding coin (initial UTXO)

**Purpose:**
Ensure that, even in a future where signatures are forgeable, all value must
enter the Anchor envelope and cannot be redirected elsewhere.

#### Phase 0 locking policy

The Phase 0 UTXO enforces the following:

1. **Anchor pinning**
   Any spend MUST create exactly one value-bearing output whose
   `scriptPubKey` equals `P_anchor`.

2. **No value leakage**
   No other value-bearing outputs are permitted.
   Transaction fees are paid by reducing the Anchor output amount.

3. **Fee bound**
   The Phase 0 script MUST enforce a bound on fee extraction, e.g.:

```

AnchorValue ≥ InputValue − MaxFee

```

This prevents an attacker from draining value via excessive fees while
preserving fee flexibility.

These conditions are enforced using **OP_TXHASH**, selecting and verifying:

* the number of outputs,
* the `scriptPubKey` of the Anchor output,
* and sufficient value information to enforce the fee bound.

No commitment is made to final recipients or future templates.

---

### Phase 1: AnchorPublishTx

**Properties:**

* Spends the Phase 0 UTXO.
* Creates exactly one output: the **Anchor UTXO**, locked to `P_anchor`.

The Anchor envelope is now instantiated on-chain.
An attacker may have triggered this spend, but cannot redirect value.

---

### Anchor UTXO locking script

A Taproot script tree with two spending paths.

#### Path 1: Reveal spend (normal)

Conditions:

1. **Relative depth gate**
   The Anchor UTXO must have aged by at least `k` blocks (CSV).

2. **Reveal check**
   `SHA256(x) == C`.

3. **Template enforcement**
   The spending transaction MUST match template `T` via OP_CTV.

---

#### Path 2: Escape hatch

Conditions:

1. **Template enforcement**
   The spending transaction MUST match template `E` via OP_CTV.

2. **No secret revealed**
   The value `x` is not disclosed on this path.

The escape path may be immediately available or time-delayed,
depending on the chosen policy.

---

### Phase 2: SpendAnchorTx

* **Reveal path witness:** `x` plus any required non-cryptographic data.
* **Escape path witness:** no `x`.

---

## Security properties

* **Quantum signature safety**
  Forged signatures do not enable theft. All value is confined to the Anchor
  envelope before any secret is revealed.

* **No redirect-after-reveal**
  Once `x` is revealed, OP_CTV pins the outputs. An interceptor cannot redirect
  value.

* **Observation is sufficient**
  If an attacker publishes Phase 0 or Phase 1 spends, the Anchor script still
  contains a usable escape hatch.

* **Reorg resistance**
  The relative timelock `k` mitigates shallow reorg games at the reveal boundary.

* **Prunable-friendly**
  Validation requires no historical transaction lookup.

* **Graceful degradation**
  A quantum attacker can force execution or cause delay, but cannot steal value.

---

## Where OP_TXHASH fits

OP_TXHASH is used **only in Phase 0** to enforce a *partial covenant*:

* it pins the **next envelope** (`P_anchor`),
* without pinning final recipients,
* without pinning templates `T` or `E`,
* without committing to exact output amounts.

This is the minimal constraint required to survive a future where
signatures are forgeable.

Because the destination is pinned only to the Anchor envelope,
Replace-By-Fee (RBF) is safe to allow in Phase 0.

---

## Summary

Phase 0 pins the **envelope**, not the destination.
Phase 1 instantiates the envelope.
Phase 2 chooses and enforces the destination.

The attacker is reduced from a thief to a griefer.

See: https://gist.github.com/earonesty/ea086aa995be1a860af093f93bd45bf2 for demo code.

-------------------------

reardencode | 2025-12-30 19:14:33 UTC | #2

I see a couple of problems with this construction:

1) the T and E must be known at creation time of Phase0 outputs which means that phase0 must contain outputs that can be consolidated to exactly match the T and E CTV output totals.
2) this would only be quantum safe if secp256k1 were disabled (NUMS points are spendable by a quantum adversary)
3) because of (1) the user cannot direct funds to a new quantum-safe address on withdrawal

I think a modified version of this using CCV instead of TXHASH could resolve issuse (1) and (3). CCV can enforce value flow and embed and extract the CTV reveal spend at spend time rather than creation time, and it can also be used to enforce the escape spend with value flexibility.

With CCV, this would look like a standard vault construct with the exception of the embedded hash and preimage for the reveal-spend.

-------------------------

simul | 2026-01-08 12:16:09 UTC | #3

What nums point?  Proposal does not rely on NUMS.  The escape hatch is *back to the original construction*.  Attacker can only grief.  Not spend.

-------------------------

reardencode | 2026-01-09 04:06:31 UTC | #4

Perhaps I misunderstood. Are you intending this script to operate in P2TR or P2TSH? If the TXHASH or CTV locks are in P2TR, there must be a NUMS point for the internal key which is then tweaked to become the P2TR key, and which the quantum adversary can attack. If this is intended to depend on P2TSH then I retract that comment.

-------------------------

simul | 2026-02-20 18:52:07 UTC | #5

(post deleted by author)

-------------------------

simul | 2026-02-20 18:52:17 UTC | #6

(post deleted by author)

-------------------------

simul | 2026-02-20 18:54:04 UTC | #7

As noted by @reardencode , this construction requires a new Segwit witness version that eliminates key-path spending entirely and enforces script-only evaluation. The goal is to make signature forgery irrelevant by removing the NUMS/key fallback path. This can be introduced as a soft fork by defining a new witness version whose semantics are enforced only by upgraded nodes.

a modified version of this using CCV instead of TXHASH could resolve fee issues. CCV can enforce value flow and embed and extract the CTV reveal spend at spend time rather than creation time, and it can also be used to enforce the escape spend with value flexibility.

-------------------------

