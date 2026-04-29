# Post-Quantum Migrations via Commit-then-Reveal (ala BIP16) - An Alternative to Sunsetting

jsalser | 2026-04-29 20:27:59 UTC | #1

### **Summary**:

Current post-quantum (PQ) migration strategies (like BIP 361) focus on address "sunsetting," but they leave users vulnerable to a "mempool race" where a quantum attacker could front-run a migration transaction the moment the public key is revealed, particularly for users who do not migrate in time (prior to quantum computers being able to effect an attack in less than 10 minutes).

This proposal introduces a **Commit-then-Reveal** soft-fork. It allows users to anchor a hash of their intended migration (the "assertion") via separate on-chain transaction. By the time the owner reveals the public key and signs the move to a QR destination, the "truth" of the destination is already immutable and deep in the chain. This effectively neutralizes Shor’s algorithm by replacing a real-time race with a historical audit.

### **Proposal**:

#### The Problem: The ECDSA Reveal Window

Even if we provide quantum-resistant (QR) addresses (e.g., Witness v3 ML-DSA), moving legacy P2PKH/P2PK funds requires revealing the public key in the mempool. Current estimates (April 2026) suggest a 9–15 minute window where a quantum-capable attacker could derive the private key and broadcast a competing transaction with a higher fee, hijacking the migration.

#### The Mechanism: Stealth Claims

The proposal splits the migration into two distinct asynchronous phases:

*Phase 1: The Commitment (The Stealth Claim)*

The user creates a migration transaction to a QR address but does not broadcast it. Instead, they broadcast only a hash of that transaction.

These hashes are included in an additional on-chain “anchoring” transaction. This would be done as a separate transaction from a QR address with hash data in the OP_RETURN. This would contain two parts - a reserved space for the first N hex characters of the HASH160 to aid with lookup and the remainder for the SIGHASH_ALL digest (with an above market fee, as something like CPFP would allow a malicious mining pool to crack then republish a transaction and give it all to themselves as a fee, theoretically).

Because only a hash is on-chain, no public key is revealed except the anchoring transaction (presumably from a QR address). An attacker has no target with a known/plausible decipher mechanism.

To prevent UTXO set bloat, the standard would be to drop the assertions after a month if unused. Best practice would be to publish this transaction preimage mere days before migrating.

*Phase 2: The Reveal*

After the hash is anchored by *N* blocks (say, 100, ensuring re-org protection), the user broadcasts the full data: the transaction preimage (from which the Sighash is derived), the ECDSA public key, and the signature.

#### Consensus Rule: Earliest Assertion

The soft-fork establishes a new validation rule: “The earliest valid commitment for a UTXO defines its only valid spend path.”

If an attacker cracks the key after the Reveal and tries to send the funds to a different address, the nodes will reject it.

The network will only honor the transaction that matches the hash previously anchored in Phase 1 (if present). This allows indefinite backwards compatibility if no anchor is present and existing on-chain users to safely sit idle until this soft-fork is activated. 

#### Technical Precedent and Parallels

This logic is a direct extension of BIP 16 (P2SH) and the recently proposed BIP 360 (P2MR). In both cases, the network accepts a hash as a “commitment” to future conditions. This proposal simply applies that logic to the migration of legacy UTXOs - as one possible option for those who choose to use it - to ensure that the owner’s intent is timestamped before their key is exposed.

#### Key Advantages

Neutralizes Front-Running: Attackers cannot “race” the owner because the owner’s destination was decided many blocks ago.

Reduces “Q-Day” Panic: Cold storage users can “lock in” their migration path any time without actually moving the coins yet or can simply wait indefinitely if they have never re-used an address.

Non-Confiscatory: Unlike a mandatory freeze, this is a voluntary tool that preserves property rights while enhancing security.

#### Disadvantages

This requires an additional transaction from an unrelated address.

More data in the UTXO set.

Requires pre-committing to a fee with a future 1-shot / must-settle migration attempt.

-------------------------

