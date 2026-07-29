# BIP 360 (P2MR) implemented in Bitcoin Core on regtest: vector results, measurements, spec feedback

jeanpablojp | 2026-07-28 21:29:41 UTC | #1

I spent the last few days implementing BIP 360 (P2MR) inside Bitcoin Core as a learning exercise, and ended up with results that seem worth sharing: all official construction vectors passing, two bugs found in the PQC vector file, some measured witness sizes, and a few places where the spec text could be tighter.

To be clear up front: this is an independent, educational implementation on regtest only. I'm not affiliated with the BIP authors and I'm not proposing anything about activation. The spec version is pinned to v0.12.x, bitcoin/bips commit [0fdf6ff](https://github.com/bitcoin/bips/commit/0fdf6ffdbb394a73c80978ae647322ceda8b9337) (2026-07-24).

- Code: [jeanpablojp/bitcoin, branch `p2mr-regtest`](https://github.com/jeanpablojp/bitcoin/tree/p2mr-regtest)
- Notes and findings: [jeanpablojp/bitcoin-p2mr](https://github.com/jeanpablojp/bitcoin-p2mr)

## How small the consensus change is

|part | lines|
|--- | ---|
|consensus (interpreter, error codes) | ~96|
|policy and block script flags | ~33|
|address recognition | ~59|
|unit tests | ~540|
|functional test | 492|

The consensus delta is under a hundred lines because BIP 360 reuses BIP 341/342 almost entirely: the witness v2 branch shares the tapscript execution, the signature message and the Merkle path walk, and only the commitment check differs (computed root == 32-byte program, no key tweak). I think that's a point in the proposal's favor and worth stating with numbers.

One thing that cost me a debugging session and that I'd pass on to anyone else implementing this: patching the script interpreter is not enough. `PrecomputedTransactionData::Init()` decides whether to precompute the BIP 341 sighash midstate by pattern-matching the spent outputs, and it doesn't recognise witness v2 programs. If you only patch the interpreter, everything works right up until you verify a real signature.

## Test vector results

`p2mr_construction.json`: all nine vectors are satisfied. The seven construction cases match byte for byte (leaf hashes, Merkle root, scriptPubKey, bech32m address and control blocks), and each control block is walked back to the root through the consensus code; the remaining two are error vectors, checked as such.

`p2mr_pqc_construction.json`: three vectors carry two bugs between them. Confirmed with an independent Python recomputation straight from the spec formulas:

- In `p2mr_three_leaf_complex` and `p2mr_three_leaf_alternative`, `intermediary.leafHashes[0]` and `[2]` are swapped relative to the tree's depth-first order. The values are right, the order isn't; the same trees in `p2mr_construction.json` list them in depth-first order. It looks like the same class of mistake as the control-block ordering fix in [bips PR #2202]: that fix didn't reach the pqc file's `leafHashes`.
- In `p2mr_different_version_leaves`, the control block for the leaf with `leafVersion` 0xfa starts with byte 0xc1. By the spec's own control byte rule it must be 0xfb, and as published that control block cannot satisfy validation.

Fix submitted as [bitcoin/bips#2220]. Documentation problems (dead rust ref-impl link, a vector filename that 404s from the BIP text, the version header behind the changelog, and a witness size example that labels a 32-byte Merkle path "(empty)") were submitted as [bitcoin/bips#2221].

## Measured witness cost vs taproot script path

Regtest, one input, one P2TR output, `SIGHASH_DEFAULT`, a `<key> OP_CHECKSIG` leaf. These numbers are produced and asserted by the functional test (`feature_p2mr.py`), so they're reproducible:

|spend | control block | weight | vsize|
|--- | --- | --- | ---|
|P2MR, 2-leaf tree | 33 B | 513 | 129|
|P2MR, 4-leaf tree | 65 B | 545 | 137|
|P2TR script path, 2-leaf tree | 65 B | 545 | 137|

The 32-byte saving the BIP claims over an equivalent taproot script path holds exactly (no internal key in the control block). Put another way, P2MR buys one extra tree level for free relative to taproot: the 4-leaf P2MR spend and the 2-leaf P2TR script path spend both cost 137 vB.

While checking that arithmetic at other depths I noticed the BIP's "always 32 bytes smaller" claim has one exception: at depth m = 7 the two control blocks fall on opposite sides of the compact size boundary (257 vs 225 bytes), so the saving there is 34 bytes. The BIP already footnotes the boundary for the P2MR side; it just isn't carried into the comparison. Fix submitted as [bitcoin/bips#2223].

To be fair about what this compares: these are script paths on both sides. A taproot key path spend is a single 64-byte signature and stays far cheaper than any P2MR spend. That's the tradeoff the proposal makes, and the P2MR vs P2TRv2 discussion is precisely about whether it's worth it. The numbers above don't settle that; they just confirm the script-path arithmetic in the BIP.

## Spec feedback

Three things I'd raise, beyond the PR-sized fixes above:

1. **The control byte's low bit.** Script Validation says the low bit "is unused and must be 1", but unlike every other rule in that section there is no explicit "Fail if" clause. The footnote implies enforcement (it describes the bit catching deserialization bugs "with an immediate error"). I implemented the strict reading (reject when the bit is 0, with a dedicated script error), but I'd suggest the text make the Fail clause explicit either way.
2. **The single-leaf vector.** `p2mr_single_leaf_script_tree` builds a depth-0 output (Merkle root == leaf hash). That's valid as construction, but since v0.12.0 spending it is anyone-can-spend. Should the vector carry a warning? My regtest RPCs simply refuse to build depth-0 outputs.
3. **No spend-path vectors.** The official vectors are construction-only: script tree in, root/address/control blocks out. There are no spending transactions or signatures, unlike BIP 341's script-path spend vectors, so everyone implementing consensus has to invent their own spend coverage. I have unit tests and an end-to-end functional test (including rejection of an invalid spend inside a hand-built block) that could be turned into proper spend-path vectors. Would that be welcome upstream?

## Closing

The branch is regtest-only by construction: the helper RPCs refuse to run on any other chain, since a witness v2 output is anyone-can-spend everywhere BIP 360 is not active. I'm also not claiming correctness beyond what's tested: the official construction vectors, ~540 lines of unit tests (mutation-tested), and the functional test.

Review is welcome, and I'm curious whether anyone else is implementing this. If spend-path test vectors would be useful upstream I'm happy to generate and submit them, and a Bitcoin Inquisition adaptation is a natural next step if there's interest.

[bips PR #2202]: https://github.com/bitcoin/bips/pull/2202
[bitcoin/bips#2220]: https://github.com/bitcoin/bips/pull/2220
[bitcoin/bips#2221]: https://github.com/bitcoin/bips/pull/2221
[bitcoin/bips#2223]: https://github.com/bitcoin/bips/pull/2223

-------------------------

