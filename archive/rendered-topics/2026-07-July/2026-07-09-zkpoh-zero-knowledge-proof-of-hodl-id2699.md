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

