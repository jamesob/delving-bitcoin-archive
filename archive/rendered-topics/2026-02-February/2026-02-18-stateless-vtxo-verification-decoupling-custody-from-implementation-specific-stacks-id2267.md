# Stateless VTXO Verification: Decoupling Custody from Implementation-Specific Stacks

jgmcalpine | 2026-02-20 13:43:45 UTC | #1

#### **Author's Note / Update:**

*Following constructive feedback from the Second team, I want to acknowledge that the original framing in this post—specifically the use of terms like "proprietary environments" and "Implementation-Coupled Custody"—is inaccurate and overly adversarial. Bark (and the wider Ark ecosystem) is fully open-source, and exit paths are verifiable from off-chain data alone without vendor lock-in. My intent was to highlight the friction of "Client-Stack Dependency" in constrained no_std environments, but the original vocabulary missed the mark. I have left the text below as originally written so that the context for the replies is preserved. I highly encourage readers to read the comments below for a more accurate representation of the technical realities, particularly regarding open-source verifiability and mempool policy.***

I. Introduction: Bifurcated Custody and the “Half-Key” Problem**

As the VTXO ecosystem evolves, lead implementations are diverging to optimize for different throughput and latency profiles. While this flexibility is beneficial, it creates a “Verification Gap.” In the VTXO model, custody is bifurcated into what we call the **Half-Key Problem**: spending authority requires both a private key and the specific off-chain transaction context (the “Map”). Unlike the L1 UTXO model, where this context is a public good preserved on-chain, VTXO context is ephemeral and off-chain.

Without independent verification of this “Map,” user custody becomes coupled to implementation-specific software stacks. If a user cannot verify or reconstruct their exit path outside of their Ark Service Provider’s (ASP) proprietary environment, they are in a state of **Implementation-Coupled Custody**.

This post proposes a **Stateless VTXO Verification** model and a neutral “Audit Ingredient” schema. Our goal is to provide a path for independent auditability—specifically for high-security, `no_std` environments like hardware wallets—ensuring that users can verify their off-chain state regardless of the underlying implementation’s design choices.

## **II. Forensic Observations: The Dialects of VTXOs**

In developing **libvpack-rs**, a stateless `no_std` verifier, we observed significant “dialect” differences in how implementations construct the VTXO identity and transaction templates.

#### 1. Indexing Standards

\*   **Arkade (by Ark Labs)** treats the VTXO as being synonymous with the **Virtual Transaction** itself. Their 32-byte ID commits to the entire transaction (assuming the user occupies the primary output).

\*   **Bark (by Second)** treats the VTXO as a **distinct output** within a potentially multi-output transaction. Their 36-byte ID (OutPoint) allows for a more flexible topology where a single virtual transaction might create multiple VTXOs (e.g., in a batched payment).

#### **2. The nSequence Signal**

We observed divergent uses of the `nSequence` field in pre-signed exit trees:

\*   **Arkade:** Utilizing `0xFFFFFFFE` to signal RBF compatibility within the constraints of TRUC (BIP-431) policies.

\*   **Bark**: Exploring `0x00000000` to leverage BIP-68 relative timelocks.

These choices change the `SIGHASH` of every transaction in the tree. An independent verifier must “know the dialect” to prove that a signature is valid.

## **III. Proposal: The Minimal Viable VTXO (MVV) Schema v0.1 (RFC)**

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

## **IV. libvpack-rs: Structural vs. Consensus Validity**

The primary goal of `libvpack-rs` is to provide a “Stateless Verifier.” It is important to distinguish between the two pillars of validity it aims to address:

1\.  **Structural Validity (Current Focus):** Proving the off-chain “Map” is mathematically sound. `vpack` verifies that the Merkle path leads to a valid L1 anchor and that the transaction preimages match the signatures.

2\.  **Consensus Validity (Roadmap):** Proving the money exists on-chain. While `vpack` is stateless, our roadmap includes an interface to ingest Compact Block Filters (BIP-158) or external block explorer data to verify that the `anchor_outpoint` remains unspent.

The decision to keep the library `no_std` is deliberate. For VTXO-based scaling solutions to reach their full potential, we should provide a path for verification within high-security, restricted environments like hardware wallets. This would allow devices like a hardware wallet to run a lean Rust library to verify the “Map” before a user signs a Forfeit transaction.

## **V. The Policy Trap: Beyond Consensus**

A major challenge for independent verifiers is **Mempool Policy**. An exit path may be valid under consensus rules but unbroadcastable under current relay policies.

For example, an Ark tree 20 levels deep is valid on-chain, but BIP-431 (TRUC) limits mempool packages to a depth of 2. If a user is not aware of this “Policy Trap,” they may believe they have a valid exit when they actually have a “pinned” or un-relayable transaction. We are exploring how `vpack` can include a **Policy Auditor** to warn users if their VTXO “Map” is structurally sound but policy-invalid.

## **VI. Open Questions & Future Research**

1\.  **Path Exclusivity:** Current verification proves a path \*exists\*. Proving that no \*malicious\* paths exist (e.g., ASP backdoors) requires full tree disclosure. How do we standardize full-tree audits for mobile clients?

2\.  **Generic vs. Semantic Connectors:** We propose treating Connectors as generic VPackOutputs (blobbing value + script) to simplify verification. Is there a use case where the verifier *must* understand the semantic logic of a Connector (e.g., inspecting the witness stack) that this generic approach misses?

3\.  **Nostr Backup Standard:** Should we formalize a NIP for the encrypted storage of `VtxoIngredient` blobs?

## **VII. Conclusion**

The diversity in current Ark implementations is a sign of a healthy protocol. Our goal with `vpack` is to ensure that this diversity does not lead to implementation-coupled custody. By adopting a stateless verification model and a standardized “Audit Ingredient” export, we allow implementations to move fast while giving users the tools to **verify, not trust.**

\*You can view the current implementation of the stateless verifier at \[github.com/jgmcalpine/libvpack-rs\]( https://github.com/jgmcalpine/libvpack-rs ).\*

-------------------------

nwoodfine | 2026-02-20 05:30:05 UTC | #2

Neil from Second here! Great proposal, getting more diverse ways of verifying validity of VTXOs can only be a good thing. [More elaborate responses on our forum](https://community.second.tech/t/announcing-v-pack-independent-verification-and-visualization-for-the-vtxo-ecosystem/183/5), but two things worth correcting here: Bark (our Ark implementation) is fully open source and exit paths are verifiable from the output data alone, so the “proprietary environment” / “Implementation-Coupled Custody” framing is not a fair description of the situation. And mempool policy shouldn’t be a major challenge thanks to Bark’s adoption of package relay + CPFP in emergency exits.

-------------------------

jgmcalpine | 2026-02-20 13:38:34 UTC | #3

Hi Neil, thanks for taking the time to read through the proposal and for the feedback. 

You are completely right to call out the framing, and I own that misrepresentation. Using the phrase "proprietary environment" was simply the wrong terminology. I know Bark is fully open-source and that all exit paths are cryptographically verifiable from the off-chain data alone.

My intent was to highlight the friction of Client-Stack Dependency in highly constrained, `no_std` environments. Even with open-source node software, a hardware wallet can't easily import a heavy, implementation-specific software stack just to parse a VTXO and verify an exit path. The goal of vpack is to act as a neutral, lightweight translation layer so hardware wallets can verify that data without needing to "know" the entire Bark or Arkade codebase. I let the wording drift into an adversarial tone ("Implementation-Coupled Custody"), which doesn't reflect the open nature of what your team is building. I appreciate you checking me on that and am pivoting the V-PACK documentation to focus on **Implementation-Agnostic Verification** rather than 'custody' to better reflect these technical realities.

Regarding the mempool policy: that is an excellent point. I painted the TRUC depth limit as a universal "Policy Trap," but Bark’s adoption of package relay and CPFP for emergency exits is exactly the kind of robust engineering that solves this. An independent verifier needs to correctly recognize Bark's specific CPFP architecture so it doesn't falsely warn a user about a policy trap that your implementation has already neutralized.

I'm looking forward to digging deeper into the Bark architecture to ensure the `VtxoIngredient` schema accurately accommodates your package relay designs. Thanks again for the constructive pushback!

-------------------------

ErikDeSmedt | 2026-02-20 16:52:30 UTC | #4

Would be happy to see you playing with it. Really happy to get some thorough feedback on how exits of vtxos are implemented. We try to do our best but getting in depth external review is super helpful.

-------------------------

