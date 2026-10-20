# Evaluating Deterministic BFT Finality within a Confidential UTXO State Machine

finalis-core | 2026-10-20 20:21:11 UTC | #1

Hello everyone,

I would like to share the architectural layout of an open-source, from-scratch C++ Layer-1 protocol called FinalisCore and open a technical discussion regarding its state transition model, specifically looking for critiques on optimization and edge-case validation rules.

The protocol attempts to isolate a unique trade-off footprint: bringing strict deterministic finality to a UTXO-based architecture while enforcing functional transactional privacy starting from block zero.

1\. Deterministic BFT State Transitions

Rather than relying on classic heavy longest-chain consensus rules where deep state reorganization (reorgs) presents a rolling probabilistic risk, Finalis moves the ledger via bounded epochs (32 blocks).

• Blocks are completely finalized the millisecond a floor(2N/3) + 1 cryptographic quorum of the designated epoch committee signs the block hash.

• The state machine operates on absolute step-increments: height = finalized_height + 1.

• Committees are selected dynamically per epoch, derived deterministically from the finalized historical chain database based on locked validator stakes.

2\. Native Privacy via secp256k1-zkp

To mitigate address-linking and transaction graph heuristics without introducing heavy Layer-2 or zk-SNARK trusted setups, all outputs utilize Pedersen commitments with non-negative range proofs.

• The cryptographic library used is secp256k1-zkp (modeled similarly to the structural implementations found in Elements/Liquid).

• Validators evaluate target balance equations purely via homomorphic addition to confirm inputs and outputs balance cleanly, without ever exposing the raw asset amounts.

3\. Current Sandbox & Open Technical Questions

We have deployed our initial public testing framework (testnet-1) to verify the liveness/safety bounds under intentional network partitions. The high-level specification validation was mapped and executed via TLA+.

As we look to scale this sandbox organically from our single bootstrap validator to a decentralized subset of independent node operators, I am highly interested in gathering technical feedback on the following implementation concerns:

1\. Mempool Invalidation during Fast Committee Shifting: In a strict deterministic finality bounds model, input selections can become brittle if a new validator committee enforces restrictive propagation thresholds. How do structural UTXO architectures handle dynamic wallet fallback selection pipelines when consensus leadership rotates rapidly?

2\. Pedersen Commitment Batch Verification Bottlenecks: While homomorphic structural proof validation guarantees transaction sanity, the raw CPU cycles required to batch-verify range proofs under standard consumer architecture hardware limits our target 3-minute block windows under heavy simulations. Are there distinct pipeline parallelization patterns or cache-locality improvements you would recommend for optimization inside the secp256k1-zkp boundary?

The codebase is published under the MIT license, and you can review the repository, Whitepaper, and configuration layouts directly on GitHub:

• Repository: https://github.com/FinalisCore/finalis-core

• Technical Whitepaper: https://github.com/FinalisCore/finalis-core/blob/8042421e6d497faba6dc7a1f6467aac2e08a194e/docs%2Fwhitepaper%2FWHITEPAPER.md

• Testnet Setup Guide: https://github.com/FinalisCore/finalis-core/blob/8042421e6d497faba6dc7a1f6467aac2e08a194e/testnet%2FREADME.md

(Mandatory Protocol Note: The live environment is currently entirely non-commercial. All testnet FLS tokens utilized for testing committee bonding rules maintain absolute zero real-world economic value).

I would deeply appreciate any code review, architectural critique, or protocol-level feedback from this community.

-------------------------

