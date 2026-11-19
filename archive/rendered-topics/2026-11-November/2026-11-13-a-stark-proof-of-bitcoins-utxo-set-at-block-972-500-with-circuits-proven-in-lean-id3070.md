# A STARK proof of Bitcoin's UTXO set at block 972,500, with circuits proven in Lean

l-henri | 2026-11-17 10:53:51 UTC | #1

Hi everyone.

I've been working on a side project to see how far a STARK prover can go on Bitcoin consensus. Here is what I have so far.

Demo: https://starknet-innovation.github.io/starked-bitcoin/
Repo https://github.com/starknet-innovation/starked-bitcoin

What the proof says

There is one proof file (a "seal") of 1.6 MB. A browser verifies it in about half a second. It shows three things:

1. Headers. There is a header chain from genesis to the tip with the stated chainwork. Every header follows Core's mainnet header rules: SHA256d linkage, proof of work, retargeting, median time past, minimum version and exact chainwork. The one exception is the wall-clock time-too-new rule. As with any SPV-style check, the verifier compares the chainwork against a trusted minimum.
2. Blocks 935,001 to 972,500. Each block is replayed: raw block data against the Merkle root, txid and wtxid, Core's structural block and transaction checks, amounts, and every coin created and spent.
3. The coin set. The UTXO set after block 972,500 is the result of that replay. Its hash_serialized equals Core's: Core's gettxoutsetinfo / dumptxoutset at 972,500 gives 9f92d143…c8c4eb, the value the proof outputs. The proof also commits to a Merkle root over the set, so anyone can produce \~1 KB inclusion and non-inclusion proofs for single outpoints. The demo page does this by downloading one \~2 MB segment of the set per lookup.

What it does not say

* No scripts or signatures, as with assumevalid. Also not yet checked: BIP68 relative locktimes and the full sigop cost (only legacy sigops are counted). Core under assumevalid enforces both, so today's guarantee is a bit weaker than assumevalid (see the reply below). The block data proof commits to witness data so that a script module can be added later without re-proving the rest.
* The replay starts from Core's assumeUTXO snapshot at 935,000. The starting set is checked against the hash_serialized pinned in Core's chainparams. Blocks before that are covered only by the header chain. The coins come with the same trust as loading Core's snapshot, which is the trust assumeUTXO users already accept, but it is not a proof from genesis. A from-genesis would cost a bit of money that I have not spent yet (probably 1-1.5k USD). Our current estimate for the whole chain's block proofs is roughly 325 H100-hours.
* No external review yet - have at it :slight_smile:

How it's built

* Prover. Stwo (circle STARKs over M31), through a Zig port (stwo-zig ( https://github.com/teddyjfpender/stwo-zig )), plus a CUDA backend that produces byte-identical proofs. A large block (2^23 trace rows) proves in 2.7 s on an H100. Blocks are aggregated by recursion per epoch, epochs are joined, and the seal wraps the chain, the header chain and the set passes into one proof.
* Circuits written in Lean 4. The Zig prover interprets constraints exported from Lean. There are machine-checked proofs that any accepted header segment satisfies the header rules, that SHA-256 matches FIPS 180-4, that the block checks hold, and that the set transitions and recursion glue compose into the seal's statement (sealT_soundW).
* Spec vs Core. The Lean specs are differentially fuzzed against Core's own C++ code. That covers about 750,000 header-rule cases and 100,000 block and transaction cases, with 0 mismatches. They are tested, not proven, against Core's code.
* Trusted. The soundness of Stwo/FRI/LogUp, Fiat–Shamir in the random oracle model (including inside recursion, the point jonasnick raised in the block-wide signature aggregation thread ( https://delvingbitcoin.org/t/block-wide-signature-aggregation-via-snarks/2875 )), and a short list of native checks in the verifier. The full ledger of what is proven and what is trusted is in docs/reference/verification.md ( https://github.com/starknet-innovation/starked-bitcoin/blob/a3e06f53f8b28310933265540e8b4189fe630f66/docs/reference/verification.md ).

A practical use: checking an assumeUTXO snapshot from anyone

chainproof verify-snapshot takes a Core dumptxoutset file and checks it against the seal: base block, coin count, total amount and hash_serialized. On a mainnet node running Core v31.1, I dumped a snapshot at 972,500 with rollback (164,539,066 coins, 9.39 GB). The check passed in 72 s using 0.5 GB of memory on an arm64 machine; 70.7 s of that is reading the 9.4 GB snapshot to recompute its hash. Verifying the proof itself takes about 0.5 s in a browser.

Today Core only loads snapshots whose hashes are hardcoded. Next, I plan to try a small patch that accepts a snapshot attested by a verified seal and skips background validation. That turns "a node in minutes" from a trust choice into a proof, with the caveats above. I expect this to be the controversial part, and I'd like opinions before going further.

Feedback is most welcome, and especially attempts to harden it. Cheers!

-------------------------

AdamISZ | 2026-11-14 01:12:38 UTC | #2

> Spec vs Core. The Lean specs are differentially fuzzed against Core’s own C++ code. That covers about 750,000 header-rule cases and 100,000 block and transaction cases, with 0 mismatches.

Very cool! I especially like this Lean part. It's unfortunate about bitcoin that this fuzzing approach is the only one we can realistically take, but, good that you tried it.

You referred to batch signature verification indirectly (the message from Jonas you referred to was about the soundness of SNARK recursion), but that seems like a potentially interesting question here: I know that secp operations are going to be expensive in STARKs, but acceleration exists (I've been using it myself a bit in a project using RiscZero that provides that kind of acceleration), so I'm wondering if at some point you can't do batch-verify on all the Schnorr signatures into one multi-scalar multiplication. Not even just blockwide, but blockchain-wide?

(I'm not even mentioning Scripts. I *guess* that part is easier than signatures).

I mean obviously even in this assume-utxo model this is already interesting, but "going the whole hog" would be even more interesting.


The longish verification time is presumably the same kind of O(log^2 n) that you get for communication/size, and why you have 72 seconds recorded. That's a bit of a bummer. Could we consider doing the same trick as Risc Zero of "shrink wrap": proof with Groth16 that the stark verifier passes, then get trivial verification time (the *size* doesn't really matter, of course, but the time does I think. I'm imagining lighter clients like phones).

(At the same time I know a big part of the community is now considering anything non-QR as not interesting, but I disagree).

-------------------------

ajtowns | 2026-11-14 09:07:13 UTC | #3

I think it might be interesting to split this up into perhaps four parts:

 * header validation (proof of work, median time, block version signalling, etc) -- stuff light clients can already do, doesn't need to be zero-knowledge at all, because headers are cheap and worth having anyway.
 * per-block undo data commitments
 * utxo set updates:
     * a zk proof for blocks B1...Bn
     * assumes you know the block headers
     * assumes you have the utxo set commitment just prior to block B1; call it X
     * assumes that you have undo data commitments for those blocks, call them U1...Un
     * for each block Bi, validates:
       * Ui is a subset of the utxo set prior Bi
       * it consists of some ordered set of txs T0...Tm
       * T0 is a coinbase tx and spends no utxos
       * each utxo in Ui is spent exactly once by exactly one tx T1..Tm
       * each other input in Tj (2<j<=m) is a unique output of some other Tk, (1<=k<j)
       * the utxo set after Bi is the utxo set prior to Bi - Ui plus the unspent outputs of T0...Tm
     * only validates the undo data and utxo set, doesn't check for invalid signatures, bad timelocks, inflation bugs, etc
 * per-block validation:
   * given a block hash and its undo data commitment, validate it properly (ie, witness commitment, inputs >= outputs, coinbase <= subsidy + inputs - outputs, signatures, script logic, timelocks, coinbase maturity, block weight/sigop limits, etc)

I think a utxo-set-update proof system is likely to last longer and be more useful than the per-block validation stuff, because generally only the latter changes with any soft forks that might get introduced, and if you're willing to take history for granted and just want to validate new blocks, then the utxo set verification is the hard part to be confident about.

I'm mostly speculating that by providing the complicated validation proof system with just the block and undo data/commitment (rather than the full utxo set), you might get more efficient proofs.

But I also think the undo data proof would possibly be helpful for things like swiftsync where you're still running the validation code directly. And the utxo set maintenance should be reasonable to fully specify, I think, if you generalise it to also cover invalid blocks.

-------------------------

l-henri | 2026-11-17 11:37:21 UTC | #4

<!-- Draft reply to https://delvingbitcoin.org/t/a-stark-proof-of-bitcoins-utxo-set-at-block-972-500-with-circuits-proven-in-lean/3070
     Answers AdamISZ (post 2) and ajtowns (post 3). Review before posting: public post with security claims.
     Evidence for the correction: lean/StarkedAir/Spec/Block.lean:6 (BIP68 and full sigop cost deferred),
     lean/StarkedAir/Spec/BlockFlat.lean:149 (legacy sigops only), docs/design/utxo.md rules 1-7. -->

Thank you both for the answers & interest! Replying to both in this post.

1. A correction. I wrote earlier that the proof covers every rule except signatures and scripts, like assumevalid. Checking against @ajtowns's list, two `ConnectBlock` rules are missing: BIP68 relative locktimes and the full sigop cost. Currently only legacy sigops are checked. P2SH and witness sigops aren't. Core under assumevalid still enforces both, so today's guarantee is a bit weaker than assumevalid. Both fit the existing design, since the undo records already carry each spent coin's height and script. Time-based BIP68 locks also need older blocks' median time past. I've updated the first post.

2. To @AdamISZ comment on the 72 s checktime. My post wasn't clear so I edited it. 70.7 of the 72 s is reading the 9.4 GB snapshot from disk to recompute its hash. The proof itself verifies in about 0.5 s in a browser (wasm). A Groth16 wrap wouldn't buy much verification time. It would shrink 1.6 MB to a few hundred bytes, which matters for verifying on another chain though. The cost would be a trusted setup, and losing the hash-only security assumption that keeps a STARK plausibly post-quantum. I'd rather keep that for now.

3. On batching Schnorr signatures. I agree that a random linear combination makes it one MSM, and nothing stops it spanning many blocks. I see two caveats: most historical signatures are ECDSA, which doesn't batch the same way, and secp256k1 over M31 is foreign-field arithmetic, which I haven't costed yet. It's the obvious shape for a signature module.

4. The 4 layers you mention @ajtowns are close to how this is built currently.

   1. A header chain proof: You're correct it does not need ZK. What it gives is one proof for \~970k headers and an MMR commitment over block hashes that the later proofs bind to.
   2. Undo data commitments. Each block proof outputs a commitment to the block plus, per input, the spent coin (amount, scriptPubKey, height, coinbase flag), i.e. Core's undo data.
   3. UTXO set updates. A multiset argument over coin events across the chain: each spend consumes an existing coin exactly once, the undo records are those coins, unspendable outputs are excluded, and the duplicate-coinbase cases match Core. It ends in Core's `hash_serialized`
   4. Full per-block validation. Part of this is already in there: Merkle and witness commitments, `CheckTransaction`, finality, BIP34, legacy sigops, weight, and (because they need the undo data) coinbase maturity, inputs ≥ outputs and coinbase ≤ subsidy + fees. What's missing: signatures, scripts, BIP68 and full sigop cost.

The difference from your split is that amounts and maturity sit with the coin-set layer rather than per-block validation. I agree with your main point: layer 3 is the long-lived part across soft forks and the hard one to be confident in, and it's the part with the Lean soundness proofs.
Your suggestion that layer 4 only needs the block plus its undo commitment is how the script module is meant to plug in. A proven undo commitment could also serve SwiftSync-style validation, as you say. I'd be happy to try that if someone wants it.

Thanks again to both of you for the review! I'll keep working on this and see how I can make it better :slight_smile:

-------------------------

ajtowns | 2026-11-18 03:02:59 UTC | #5

[quote="l-henri, post:4, topic:3070"]
2. 70.7 of the 72 s is reading the 9.4 GB snapshot from disk to recompute its hash.
[/quote]

What's the (necessary) structure of the hash? Is it just a merkle-tree ordered by txid/vout (with MMR determining the layout), or is some rearrangement necessary?

[quote="l-henri, post:4, topic:3070"]
3. UTXO set updates. A multiset argument over coin events across the chain: ... It ends in Core’s `hash_serialized`
[/quote]

I guess I'm wondering if it would be "better" if core output the merklized hash you use above, rather than (or as well as) calculating the current hash serialized?

[quote="l-henri, post:4, topic:3070"]
The difference from your split is that amounts and maturity sit with the coin-set layer rather than per-block validation.
[/quote]

Right -- I'm mostly wondering if moving more stuff into the block validation layer could make the utxo-set-update proofs faster/cheaper (particularly on the proving side)?

In some sense block validation (given a trusted hash of the undo data) is the easy part: with a trusted link to the undo data, and no requirement to calculate the utxo set, you can use the actual code to validate blocks in parallel and out of order. Or you can just ignore it entirely, treating the assumeutxo/zkproof set as the "genesis/pre-mine", on the assumption that that's what everyone else has agreed upon, and has already been widely verified; just using your node to do full validation of future blocks, not past history, but being confident that it'll report consistent results to anyone else who will ever accept any descendant block from the assumeutxo/zkproof point.

I guess I'm thinking:
 * have core and this agree on a utxo snapshot hash that's easy to calculate and easy to prove with
 * do this proof every 14,112 blocks (per the bip452 proposal)
 * have the assumeutxo snapshot hash plus block hash at those heights be independently verifiable prior to downloading any data

If the utxo snapshot hash is a per-utxo merkle tree, that might also be useful in some sense for ~quarterly proof of reserves as well which presumably could also be done as a zkproof to avoid revealing explicit utxos.

-------------------------

l-henri | 2026-11-19 09:27:48 UTC | #6

Hello hello,
There are two structure for the hash. Both are over the coins in Core's order (txid, then vout):

* **`hash_serialized`** is Core's: one SHA-256 stream over every coin's serialization. It's sequential, so checking a snapshot against it means reading all 9.4 GB in order, which is where the 70 s goes. The proof handles it in segments by passing the SHA-256 state from one segment to the next.
* **A Merkle root**, which the proof also outputs. It's Blake2s, and each leaf is the previous coin's outpoint followed by the coin, so one path proves either that a coin is present or, through the two neighbours around it, that it's absent (\~1 KB). No reordering is needed. But the shape above the leaves currently follows how I cut the set into proving segments, so it isn't canonical yet. Agreeing with Core would mean fixing the shape (e.g. a plain balanced tree, or an MMR, over fixed-size chunks).

> I guess I'm wondering if it would be "better" if core output the merklized hash

I think it would. Compared with `hash_serialized` it gives:

* per-coin inclusion and exclusion proofs against a value Core itself reports;
* parallel computation, and a snapshot that can be downloaded and checked chunk by chunk, instead of one 9 GB sequential hash at the end.

The open choice is the hash function. Blake2s (or another prover-friendly hash) is cheap to prove, but Core doesn't have it today. SHA-256 is native to Core but much more costly to prove.

> I'm mostly wondering if moving more stuff into the block validation layer could make the utxo-set-update proofs faster/cheaper

Possibly! But I haven't measured the split yet. My expectation is that hashing the block bytes dominates the cost. And every layer needs that.

Amounts and maturity are a few columns on coin-event rows that exist anyway. I'll measure the per-component cost and report back.

> have core and this agree on a utxo snapshot hash that's easy to calculate and easy to prove with / do this proof every 14,112 blocks / have the assumeutxo snapshot hash plus block hash at those heights be independently verifiable prior to downloading any data

If this is useful to Core I'd be happy to adapt whatever is needed to make it work. If there's interest, I'd be glad to write up a concrete hash proposal (leaf encoding, tree shape, hash function) for discussion.

What you describe is close to what exists. The 1.6 MB proof attests the block hash at the snapshot height, `hash_serialized` and the Merkle root, and `chainproof verify-snapshot` checks the proof before reading the snapshot. What's missing is your first point (a hash both sides compute natively) and a Core that accepts the result.

On proof of reserves: that is very doable. What is missing would be ownership signatures, and folding it in a zk proof. 

As a follow up question: the current proof was generated from a assumeUTXO snapshot. Do you think it would be valuable to create a proof from genesis? I'm waiting before doing this, given it has a cost :smile:

-------------------------

