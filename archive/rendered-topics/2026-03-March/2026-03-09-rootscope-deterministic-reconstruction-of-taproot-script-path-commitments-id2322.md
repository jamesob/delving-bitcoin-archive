# RootScope: deterministic reconstruction of Taproot script-path commitments

AaronZhang | 2026-03-09 19:28:32 UTC | #1

Taproot script-path spends can be surprisingly difficult to inspect
manually.

Given a witness, reconstructing the full commitment chain —
”TapLeaf → TapBranch → TapTweak → output key → bech32m address” —
requires several sequential steps. Each step has its own failure
modes: tagged hash label, byte order, parity bit, lexicographic
ordering. Mistakes often produce silent mismatches.

RootScope is a small tool that performs this reconstruction
deterministically and exposes every intermediate value.

Given script + control_block, it computes:

* TapLeaf hash
* each TapBranch step up the Merkle path
* TapTweak and the tweaked output key
* bech32m address
* optional expected-address match check

There is also a fetch-witness helper that accepts a txid + vin and
resolves the witness from a block explorer, prefilling the inputs
for real transaction inspection.

**Validation:**

* BIP 341 wallet-test-vectors.json — 12 script-path paths across
  6 test cases, including multi-path and mixed-depth examples
  (notably cases 5 and 6): 12/12 PASS.


* Additional vectors from Mastering Taproot \[1\] cover single-leaf,
  dual-leaf, and a 4-leaf balanced tree, plus a 3-leaf unbalanced
  tree where two paths of different depths (65-byte vs 97-byte
  control block) both reconstruct to the same output key.

Implementation is pure Python (stdlib only, hashlib).


Note: not intended for high-speed signing — reconstruction of a
single witness is perfectly fast at this scale.

Note on annex: if the witness contains an annex (final entry
starting with 0x50), the control block position shifts. Annex
handling is not currently implemented; a witness with an annex
will produce an incorrect result.

A concrete example: block 787999, txid
[f1de8d3b3894a7cc0efffa7332bf7236ad089195b9d642121f4844f13e01f1e0](https://mempool.space/tx/f1de8d3b3894a7cc0efffa7332bf7236ad089195b9d642121f4844f13e01f1e0),
vin 0 — the tool walks through each hash step and confirms the
reconstructed address matches the output.

Full repro instructions and batch mode docs are in *REPRODUCE.md.*

Repo: https://github.com/aaron-recompile/rootscope

A few open questions I’d be interested in feedback on:

1. Are there edge cases in the reconstruction flow I may have
   missed — unusual control block structures or non-standard
   leaf versions seen in the wild? (Non-0xc0 leaf versions are
   supported; the tool will note the version but continue.)

2. For script-template labeling: is conservative, manually-
   confirmed classification preferable to automatic pattern
   matching for this kind of tooling?

3. Are there existing public datasets of script-path spends
   that would make sense to integrate with a batch pipeline?

Aaron Zhang

\[1\] https://github.com/aaron-recompile/mastering-taproot

-------------------------

AaronZhang | 2026-03-25 22:30:14 UTC | #2

One interesting use case that came up in later experiments:

In a recent IK + CSFS run on Bitcoin Inquisition:

https://delvingbitcoin.org/t/bitcoin-inqusition-29-2/2236/6

I realized that OP_INTERNALKEY implicitly assumes that **the internal key is trusted.**

In practice, this trust comes from the control block verification — reconstructing the Taproot commitment chain and ensuring that P → Q holds for the given UTXO.

From this perspective, reconstruction is **not just a debugging aid,**
but a prerequisite for reasoning about **identity-bound authorization**.

-------------------------

