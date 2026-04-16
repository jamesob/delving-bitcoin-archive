# [BIP Draft] Witness Version 3: ML-DSA-65 Post-Quantum Key-Path Spending

echelonresearch | 2026-04-16 08:28:42 UTC | #1

Hello Delving Bitcoin,

Posting a draft BIP with a working reference implementation that adds
post-quantum key-path spending to Bitcoin using FIPS 204 ML-DSA-65
(NIST Level 3), under a new witness version 3.

BIP text:
  [https://github.com/btcpqresearch/bip-pq-witness-v3/blob/7bbfb294337201eb7273692110b1d137e07d3e40/bip-pq-witness-v3.mediawiki](https://github.com/btcpqresearch/bip-pq-witness-v3/blob/main/bip-pq-witness-v3.mediawiki)

Reference implementation (patch + functional tests):
  [https://github.com/btcpqresearch/bip-pq-witness-v3](https://github.com/btcpqresearch/bip-pq-witness-v3)

Applies cleanly against bitcoin/bitcoin at commit
c97ac44c34e8b84244d2ae765c7ee358b37b012e.

Design summary

\* Output: OP_3 <32-byte SHA-256 of ML-DSA-65 public key>. Address is
  bech32m, with HRP prefix bc1r on mainnet.
\* Witness stack: \[3309-byte signature, 1952-byte pubkey\], with optional
  annex following the Taproot 0x50 convention.
\* Verification runs inside VerifyWitnessProgram() without invoking the
  Script interpreter. No script execution, no stack semantics, and no
  element-size limits apply.
\* Sighash: tagged hash "PQSighash" with epoch byte 1 and ext_flag 3.
  Reuses the BIP 341 per-transaction precomputed hashes; kept distinct
  from Taproot sighash via tag, epoch byte, and spend-type byte.
\* Key-path only for v3. Script-path extensions are deferred to a
  follow-up BIP in the style of BIP 342.
\* New script flag SCRIPT_VERIFY_WITNESS_V3, deployment bit
  DEPLOYMENT_WITNESS_V3, and five PQ-specific script errors.

Deployable as a soft fork under BIP 9. Non-upgraded nodes treat witness
v3 outputs as anyone-can-spend per BIP 141, preserving the soft-fork
property.

Context

BIP 361 (Post Quantum Migration and Legacy Signature Sunset) references
a to-be-defined PQ signature scheme as a prerequisite for its migration
phases. This proposal is offered as one possible candidate for that role.
The migration question itself is out of scope here.

This complements BIP 360's P2MR output type (witness version 2) by
providing a concrete post-quantum signature scheme for the actual spend
side. Prior PQ-related discussions have taken place on this list and
elsewhere; pointers to posts or drafts worth citing would be welcome.

Tradeoffs

Scheme choice. ML-DSA-65 (NIST Level 3) over SLH-DSA (signatures \~8x
larger), Falcon (floating-point determinism concerns in the reference
implementation), or hybrid ECDSA+PQ (adds complexity with no additional
quantum security). Level 3 keeps signatures under 3.5 kB. This is not
an obviously correct choice; pushback is welcome.

New witness version vs. Tapscript opcode. A dedicated version keeps PQ
verification outside Tapscript's 520-byte stack element limit and weight
model (tuned for 64-byte Schnorr signatures). A new opcode like
CHECKSIG_ML_DSA would reuse more Taproot plumbing but inherit those
constraints.

Witness size and fee impact. A simple 1-in/1-out PQ spend is \~1400 vB
(\~5600 WU). At 10 sat/vB this costs \~14,000 sats. Bitcoin Core's
per-transaction standardness weight cap (MAX_STANDARD_TX_WEIGHT =
400 kWU) bounds a single standard PQ transaction at roughly 72 inputs.
The block-level consensus limit (MAX_BLOCK_WEIGHT = 4 MWU) is ten
times larger; a PQ-maximal block packs \~720 PQ inputs across its
transactions.

liboqs dependency. The reference implementation uses liboqs 0.15.0 for
rapid prototyping. A consensus-critical external dependency is
undesirable long-term; a pinned, audited, or fully in-tree ML-DSA-65
implementation will be required before any activation attempt. This is
the largest open implementation question.

Validation cost. ML-DSA verification is materially slower than Schnorr
on current hardware. Implementations should benchmark worst-case block
validation cost and consider whether weight alone suffices or if an
additional policy/consensus accounting mechanism (e.g. sigop-style
budget) is needed. A deployment without a reasoned answer here would not
be ready.

Activation. No specific BIP 9 parameters are proposed here. Those belong
in a separate deployment BIP.

Asking for

1. Review of the sighash construction. Is epoch=1 / ext_flag=3 the
   right way to share BIP 341 precomputed data without cross-version
   signature confusion, or should the PQ sighash use an independent
   precompute?
2. Feedback on the witness-version vs. Tapscript-opcode approach.
3. Thoughts on the liboqs question. Is there appetite for an in-tree
   ML-DSA implementation, or does the proposal's viability depend on one?
4. Pointers to prior PQ discussions I should cite.

Happy to respond in separate threads on migration, activation, or
scheme-aesthetics.

Thanks for reading.

Echelon Labs Research
[echelonresearch@protonmail.com](mailto:echelonresearch@protonmail.com)

-------------------------

