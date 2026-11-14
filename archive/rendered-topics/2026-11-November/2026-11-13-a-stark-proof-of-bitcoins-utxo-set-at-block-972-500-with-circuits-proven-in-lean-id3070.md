# A STARK proof of Bitcoin's UTXO set at block 972,500, with circuits proven in Lean

l-henri | 2026-11-13 10:11:27 UTC | #1

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

* No scripts or signatures. The trust here is the same as with assumevalid. The block data proof commits to witness data so that a script module can be added later without re-proving the rest.
* The replay starts from Core's assumeUTXO snapshot at 935,000. The starting set is checked against the hash_serialized pinned in Core's chainparams. Blocks before that are covered only by the header chain. The coins come with the same trust as loading Core's snapshot, which is the trust assumeUTXO users already accept, but it is not a proof from genesis. A from-genesis would cost a bit of money that I have not spent yet (probably 1-1.5k USD). Our current estimate for the whole chain's block proofs is roughly 325 H100-hours. 
* No external review yet - have at it :slight_smile: 

How it's built

* Prover. Stwo (circle STARKs over M31), through a Zig port (stwo-zig ( https://github.com/teddyjfpender/stwo-zig )), plus a CUDA backend that produces byte-identical proofs. A large block (2^23 trace rows) proves in 2.7 s on an H100. Blocks are aggregated by recursion per epoch, epochs are joined, and the seal wraps the chain, the header chain and the set passes into one proof.
* Circuits written in Lean 4. The Zig prover interprets constraints exported from Lean. There are machine-checked proofs that any accepted header segment satisfies the header rules, that SHA-256 matches FIPS 180-4, that the block checks hold, and that the set transitions and recursion glue compose into the seal's statement (sealT_soundW).
* Spec vs Core. The Lean specs are differentially fuzzed against Core's own C++ code. That covers about 750,000 header-rule cases and 100,000 block and transaction cases, with 0 mismatches. They are tested, not proven, against Core's code.
* Trusted. The soundness of Stwo/FRI/LogUp, Fiat–Shamir in the random oracle model (including inside recursion, the point jonasnick raised in the block-wide signature aggregation thread ( https://delvingbitcoin.org/t/block-wide-signature-aggregation-via-snarks/2875 )), and a short list of native checks in the verifier. The full ledger of what is proven and what is trusted is in docs/reference/verification.md ( https://github.com/starknet-innovation/starked-bitcoin/blob/a3e06f53f8b28310933265540e8b4189fe630f66/docs/reference/verification.md ).

A practical use: checking an assumeUTXO snapshot from anyone

chainproof verify-snapshot takes a Core dumptxoutset file and checks it against the seal: base block, coin count, total amount and hash_serialized. On a mainnet node running Core v31.1, I dumped a snapshot at 972,500 with rollback (164,539,066 coins, 9.39 GB). The check passed in 72 s using 0.5 GB of memory on an arm64 machine.

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

