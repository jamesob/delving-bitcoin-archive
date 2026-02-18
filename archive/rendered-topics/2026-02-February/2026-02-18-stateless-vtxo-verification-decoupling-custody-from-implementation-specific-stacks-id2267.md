# Stateless VTXO Verification: Decoupling Custody from Implementation-Specific Stacks

jgmcalpine | 2026-02-18 20:01:06 UTC | #1

**I. Introduction: Bifurcated Custody and the “Half-Key” Problem**

As the VTXO ecosystem evolves, lead implementations are diverging to optimize for different throughput and latency profiles. While this flexibility is beneficial, it creates a “Verification Gap.” In the VTXO model, custody is bifurcated into what we call the **Half-Key Problem**: spending authority requires both a private key and the specific off-chain transaction context (the “Map”). Unlike the L1 UTXO model, where this context is a public good preserved on-chain, VTXO context is ephemeral and off-chain.

Without independent verification of this “Map,” user custody becomes coupled to implementation-specific software stacks. If a user cannot verify or reconstruct their exit path outside of their Ark Service Provider’s (ASP) proprietary environment, they are in a state of **Implementation-Coupled Custody**.

This post proposes a **Stateless VTXO Verification** model and a neutral “Audit Ingredient” schema. Our goal is to provide a path for independent auditability—specifically for high-security, `no_std` environments like hardware wallets—ensuring that users can verify their off-chain state regardless of the underlying implementation’s design choices.

**II. Forensic Observations: The Dialects of VTXOs**

In developing **libvpack-rs**, a stateless `no_std` verifier, we observed significant “dialect” differences in how implementations construct the VTXO identity and transaction templates.

1\. Indexing Standards

\*   **Ark Labs** treats the VTXO as being synonymous with the **Virtual Transaction** itself. Their 32-byte ID commits to the entire transaction (assuming the user occupies the primary output).

\*   **Second Tech** treats the VTXO as a **distinct output** within a potentially multi-output transaction. Their 36-byte ID (OutPoint) allows for a more flexible topology where a single virtual transaction might create multiple VTXOs (e.g., in a batched payment).

**2. The nSequence Signal**

We observed divergent uses of the `nSequence` field in pre-signed exit trees:

\*   **Ark Labs:** Utilizing `0xFFFFFFFE` to signal RBF compatibility within the constraints of TRUC (BIP-431) policies.

\*   **Second Tech**: Exploring `0x00000000` to leverage BIP-68 relative timelocks.

These choices change the `SIGHASH` of every transaction in the tree. An independent verifier must “know the dialect” to prove that a signature is valid.

**III. Proposal: The Minimal Viable VTXO (MVV) Schema v0.1 (RFC)**

To enable independent verification without slowing down innovation, we propose a standardized **“Audit Ingredient”** format. This schema is intended as a draft for comment by the community.

```rust
/// The Minimal Viable VTXO (MVV) Ingredient Schema (v0.1 RFC)

pub struct VtxoIngredient {
  /// Implementation version to identify template logic (nSequence/Identity     dialects)
  pub dialect_version: u32,
  pub amount: u64,
  pub script_pubkey: Vec<u8>,
  pub vout: u32,
  /// The relative timelock required for the exit
  pub exit_delta: u32,
  pub sequence: u32,
  /// Merkle path to the L1 Anchor
  pub path: Vec<VPackPathStep>,
  /// The L1 Anchor (TxID and Vout) where the tree is rooted
  pub anchor_outpoint: OutPoint,
}

pub struct VPackPathStep {
  /// Every output in this transaction *except* the one leading to the user.
  /// Includes other users’ VTXOs and protocol anchors/connectors.
  pub other_outputs: Vec<VPackOutput>,
  /// The ASP’s signature authorizing this specific step in the tree
  pub signature: [u8; 64],
  pub parent_index: u32,
}

pub struct VPackOutput {
  pub value: u64,
  pub script_pubkey: Vec,
}
```

Structurally, V-PACK treats all non-user outputs (whether they are ‘siblings’ belonging to other users or ‘connectors’ used for fee management) as a single category of **Other Outputs**. By carrying the literal value and `script_pubkey` for these components, the verifier can reconstruct the TxID and verify the signature without needing to understand the implementation-specific semantics of each output type.

By exporting this schema, ASPs allow users to move their “Map” into a neutral environment. A tool like `vpack` can then “re-bake” the templates and verify the user’s spending authority.

**IV. libvpack-rs: Structural vs. Consensus Validity**

The primary goal of `libvpack-rs` is to provide a “Stateless Verifier.” It is important to distinguish between the two pillars of validity it aims to address:

1\.  **Structural Validity (Current Focus):** Proving the off-chain “Map” is mathematically sound. `vpack` verifies that the Merkle path leads to a valid L1 anchor and that the transaction preimages match the signatures.

2\.  **Consensus Validity (Roadmap):** Proving the money exists on-chain. While `vpack` is stateless, our roadmap includes an interface to ingest Compact Block Filters (BIP-158) or external block explorer data to verify that the `anchor_outpoint` remains unspent.

The decision to keep the library `no_std` is deliberate. For VTXO-based scaling solutions to reach their full potential, we should provide a path for verification within high-security, restricted environments like hardware wallets. This would allow devices like a hardware wallet to run a lean Rust library to verify the “Map” before a user signs a Forfeit transaction.

**V. The Policy Trap: Beyond Consensus**

A major challenge for independent verifiers is **Mempool Policy**. An exit path may be valid under consensus rules but unbroadcastable under current relay policies.

For example, an Ark tree 20 levels deep is valid on-chain, but BIP-431 (TRUC) limits mempool packages to a depth of 2. If a user is not aware of this “Policy Trap,” they may believe they have a valid exit when they actually have a “pinned” or un-relayable transaction. We are exploring how `vpack` can include a **Policy Auditor** to warn users if their VTXO “Map” is structurally sound but policy-invalid.

**VI. Open Questions & Future Research**

1\.  **Path Exclusivity:** Current verification proves a path \*exists\*. Proving that no \*malicious\* paths exist (e.g., ASP backdoors) requires full tree disclosure. How do we standardize full-tree audits for mobile clients?

2\.  **Generic vs. Semantic Connectors:** We propose treating Connectors as generic VPackOutputs (blobbing value + script) to simplify verification. Is there a use case where the verifier *must* understand the semantic logic of a Connector (e.g., inspecting the witness stack) that this generic approach misses?

3\.  **Nostr Backup Standard:** Should we formalize a NIP for the encrypted storage of `VtxoIngredient` blobs?

**VII. Conclusion**

The diversity in current Ark implementations is a sign of a healthy protocol. Our goal with `vpack` is to ensure that this diversity does not lead to implementation-coupled custody. By adopting a stateless verification model and a standardized “Audit Ingredient” export, we allow implementations to move fast while giving users the tools to **verify, not trust.**

\*You can view the current implementation of the stateless verifier at \[github.com/jgmcalpine/libvpack-rs\]( https://github.com/jgmcalpine/libvpack-rs ).\*

-------------------------

