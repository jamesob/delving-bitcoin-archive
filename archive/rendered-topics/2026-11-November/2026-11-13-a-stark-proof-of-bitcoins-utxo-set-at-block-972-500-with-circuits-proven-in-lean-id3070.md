# A STARK proof of Bitcoin's UTXO set at block 972,500, with circuits proven in Lean

l-henri | 2026-11-13 10:06:23 UTC | #1

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
* Trusted. The soundness of Stwo/FRI/LogUp, Fiat–Shamir in the random oracle model (including inside recursion, the point jonasnick raised in the block-wide signature aggregation thread ( https://delvingbitcoin.org/t/block-wide-signature-aggregation-via-snarks/2875 )), and a short list of native checks in the verifier. The full ledger of what is proven and what is trusted is in docs/reference/verification.md ( https://github.com/starknet-innovation/starked-bitcoin/blob/main/docs/reference/verification.md ).

A practical use: checking an assumeUTXO snapshot from anyone

chainproof verify-snapshot takes a Core dumptxoutset file and checks it against the seal: base block, coin count, total amount and hash_serialized. On a mainnet node running Core v31.1, I dumped a snapshot at 972,500 with rollback (164,539,066 coins, 9.39 GB). The check passed in 72 s using 0.5 GB of memory on an arm64 machine.

Today Core only loads snapshots whose hashes are hardcoded. Next, I plan to try a small patch that accepts a snapshot attested by a verified seal and skips background validation. That turns "a node in minutes" from a trust choice into a proof, with the caveats above. I expect this to be the controversial part, and I'd like opinions before going further.

Feedback is most welcome, and especially attempts to harden it. Cheers!

-------------------------

