# zkPoH: Zero-Knowledge Proof-of-Hodl

fabohax | 2026-07-09 14:15:35 UTC | #1

I’m sharing a small proof-of-concept for **zkPoH**, a Zero-Knowledge Proof of Hodl system for Bitcoin.

The idea is simple: prove that you control a set of Bitcoin UTXOs whose combined value is at least **1 BTC**, without revealing the UTXOs, addresses, exact balance, tx history, or private keys.

How it works in the current prototype:

1. A Bitcoin UTXO snapshot is generated off-chain.

2. The snapshot is committed into a Merkle tree.

3. The Merkle root becomes the public commitment.

4. The prover privately selects up to four UTXOs from that snapshot.

5. Rust generates the witness inputs for Noir.

6. The Noir circuit checks:

   * each selected UTXO belongs to the committed snapshot;

   * the Merkle paths are valid;

   * the sum of the selected UTXO values is at least `100,000,000 sats`.

7. If the constraints pass, the verifier only learns:
   **this prover satisfies the 1 BTC threshold against this snapshot.**

For the prototype, the Merkle tree uses Blake2s over fixed encodings, and Bitcoin ownership is checked off-circuit with signed WIF ownership proofs. Future work could move ownership verification into the circuit, add Schnorr/Taproot support, snapshot epochs, nullifiers, Utreexo commitments, and arbitrary thresholds.

This is experimental and educational, not production-ready, but it shows a concrete path for private Bitcoin proof-of-hold primitives.

Repo: github.com/fabohax/zkPoH

-------------------------

murch | 2026-07-09 18:17:42 UTC | #2

(post deleted by author)

-------------------------

treeforest | 2026-07-09 23:30:35 UTC | #3

Wow. This is some rad innovation. I haven't been keeping up much with zk happenings in Bitcoin and am happy to see this.

-------------------------

fabohax | 2026-07-10 01:34:03 UTC | #4

Without an ownership/signing step, the current construction would only prove that *some* UTXOs exist in the snapshot and that their sum passes the threshold. A malicious prover could otherwise pick arbitrary UTXOs from the chain.

So the design needs an explicit ownership binding step.

The intended flow is:

1. The verifier gives the prover a fresh challenge:

```
challenge = H(
  "zkPoH-v1",
  merkle_root,
  threshold,
  verifier_nonce,
  expiry,
  context
)

```

2. The prover privately selects UTXOs from the committed snapshot.

3. For each selected UTXO, the prover signs that challenge with the key controlling the output.

For example, for a P2WPKH output:

```
sig_i = Sign(privkey_i, challenge)

```

Then the proof checks:

```
MerkleVerify(utxo_i, merkle_path_i, merkle_root) == true
HASH160(pubkey_i) == scriptPubKey_pubkeyhash_i
VerifySignature(pubkey_i, challenge, sig_i) == true
sum(values_i) >= threshold

```

So the proof is not just:

```
“These UTXOs exist and sum to >= 1 BTC”

```

but:

```
“I know the keys that control hidden UTXOs in this snapshot, and their hidden values sum to >= 1 BTC.”

```

In the current PoC, this ownership step is not yet cleanly integrated into the Noir circuit. That should be clarified in the README. Right now, the circuit mainly demonstrates the Merkle inclusion + private threshold logic. The next step is to bind ownership signatures to the same challenge, snapshot root, and threshold.

There are two possible versions:

**v0 — off-circuit ownership check**
Simpler to implement. The prover signs a challenge for each selected UTXO and the verifier checks those signatures separately. The downside is that this may reveal the UTXOs/pubkeys to the verifier, so it weakens the privacy model.

**v1 — in-circuit ownership check**
Better version. The prover keeps the selected UTXOs, pubkeys, signatures, and Merkle paths private. The circuit verifies internally that:

* each UTXO belongs to the committed snapshot;

* the prover knows the corresponding private keys or valid signatures;

* the signatures are bound to the challenge;

* the total value passes the threshold.

The verifier only sees the public result:

```
valid proof
snapshot root = X
threshold >= 1 BTC
challenge/context = Y

```

but not which UTXOs were used.

For Taproot, this should ideally use Schnorr verification against the x-only output key. For broader Bitcoin script support, BIP-322-style message signing is probably the right direction, although proving arbitrary script satisfaction inside a ZK circuit is more complex.

The Lightning use case is very interesting, but it also needs care. A channel funding output is usually not equivalent to a simple single-key UTXO. Depending on whether it is 2-of-2 multisig, MuSig2, Taproot key-path, etc., the proof may need to show either unilateral participation, cooperative control, or some channel-specific ownership condition without revealing the actual funding output.

PoC needs to make ownership binding explicit, and the strongest version should verify that binding inside the ZK proof.

-------------------------

