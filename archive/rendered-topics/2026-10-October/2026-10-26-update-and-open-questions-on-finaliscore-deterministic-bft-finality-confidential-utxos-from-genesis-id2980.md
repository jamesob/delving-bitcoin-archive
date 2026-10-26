# Update and open questions on FinalisCore (deterministic BFT finality + confidential UTXOs from genesis)

finalis-core | 2026-10-26 10:43:49 UTC | #1

Testnet-2 has been running since 24 October 2026 with the leader VRF moved to RFC 9381. Mainnet has not launched. Testnet coins have no value. Current status and one-command node setup are in the repo and on the project site.

I am looking for independent critique of the core claims before mainnet, particularly:

1. \*\*At most one transition finalizes per height\*\* under the stated Byzantine and network assumptions (f < n/3, asynchronous network). Spec: \`spec/TWO_PHASE_FINALITY.md\`. Implementation: \`src/node/node_consensus.cpp\` and \`src/consensus/canonical_derivation.\*\`.

2. \*\*Confidential transactions cannot create value.\*\* Pedersen commitments + range proofs (secp256k1-zkp). Balance is checked homomorphically; excess authorization and non-negative range proofs are required. Spec: \`spec/CONFIDENTIAL_UTXO_SPEC.md\` §§3, 4, 7, 15. Code: \`src/crypto/confidential.\*\` and \`src/utxo/validate.\*\`.

3. \*\*Deterministic derivation of committee, checkpoints, and rewards\*\* from finalized history only. No speculative views. Specs: \`CHECKPOINT_DERIVATION_SPEC.md\`, \`COMMITTEE-SELECTION.md\`, \`ECONOMICS.md\`.

The full list of claims, reading order, and how to test them is in \`docs/REVIEWING.md\`. Findings can still change the protocol. Report security issues privately per \`SECURITY.md\`; other feedback can go here or as GitHub issues.

Repo: https://github.com/FinalisCore/finalis-core  
Whitepaper and specs are in the repo. I am also looking for additional independent validators on testnet-2.

Questions or counter-examples are welcome.

-------------------------

