# Machine-checked soundness of BIP340 batch verification with 128-bit derived randomizers

dni | 2026-11-02 08:53:43 UTC | #1

[sipa/bips#219](https://github.com/sipa/bips/issues/219) and [#222](https://github.com/sipa/bips/pull/222) propose shrinking BIP340's batch-verification randomizers from 256 to 128 bits, which speeds up batch verification in libsecp256k1 by up to about 9%. The pen-and-paper argument treats the randomizers as independent uniform values. In practice they are derived deterministically from the batch. With incremental derivation, as in [secp256k1#1134](https://github.com/bitcoin-core/secp256k1/pull/1134), the attacker can even choose later entries after seeing earlier randomizers.

I've proven soundness for both derivations in Lean 4, with Mathlib and VCVio: https://github.com/dni/bip340-batch-verify

## Results

The randomizer hash is modeled as a random oracle with `b`-bit outputs, and `q` is the number of oracle queries the attacker makes. The challenge hash is an arbitrary function.

| Setting | Bound |
|----|----|
| Independent uniform `b`-bit randomizers | `2^-b` |
| Randomizers hashed from the whole batch (the BIP340 text) | `(q + 1)/2^b`, for batches of any size |
| Incremental derivation (randomizer `i` hashes the first `i` entries) | `(q + 1)/2^b` |
| Both on secp256k1 with `b = 128` | `(q + 1)/2^128` |
| Lower bound (grinding), both derivations | `1 - (1 - 2^-b)^q ≈ q/2^b` |

So 128-bit randomizers are sound, and the only cost of deriving them deterministically is offline grinding at about one hash per attempt: around `2^64` hashes get an invalid batch accepted with probability around `2^-64`.

The grinding attack: from one valid signature `(pk, m, r, s)`, the batch `[(pk, m, r, s - δ), (pk, m, r, s + δ)]` contains two invalid signatures whose errors cancel, so it passes exactly when its second randomizer is `1`. Hashing these batches for `δ = 1, …, q` reaches the lower bound against both derivations.

## Proof idea

If one signature is invalid, then once the other randomizers are fixed, at most one value of its randomizer lets the batch pass. For incremental derivation, cut the winning batch after its last invalid signature. That prefix still passes, and its equation involves only randomizers of its own prefixes, so the last of those to be answered must hit a single value. One fresh answer can complete several cached extensions of the queried prefix, but each p so the total stays at one bad value per query. The verifier's own queries cost
`1/2^b` in total rather than `1/2^b` each, which is where the `+ 1` comes from.

## Trust base

* secp256k1 is defined as a Mathlib `WeierstrassCurve`, using Mathlib's proven oven prime with kernel-checked Pratt certificates, and the group is proven to
  have exactly `n` points.
* Everything uses only Lean's standard axioms. There's no `native_decide`, so e is checked by the kernel.
* The model is checked against the BIP340 test vectors.

## Caveat: the seed encoding

The proof assumes that distinct batches give distinct hash inputs. Since BIP340ssages of arbitrary size, its seed `seed_hash(pk_1..pk_u || m_1..m_u || sig_1..sig_u)` concatenates messages without their lengths, so moving a byte frame seed. The derivation in #1134 (`type || sig64 || msg || pk33` per entry,
without `msglen`) has the same property.

This doesn't affect Bitcoin, where every signed message is a 32-byte sighash. Fnown, since each extra batch still costs a challenge-hash evaluation, but thebound doesn't follow. Hashing each message's length before the message makes thn the result applies to messages of any length.

Feedback welcome, especially on whether the model matches what implementations

-------------------------

