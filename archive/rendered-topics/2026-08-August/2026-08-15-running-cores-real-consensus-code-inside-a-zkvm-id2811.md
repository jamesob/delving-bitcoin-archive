# Running Core's real consensus code inside a zkVM

defenwycke | 2026-08-15 15:10:44 UTC | #1

# Proving Bitcoin
## Running Core's real consensus code inside a zkVM.
### Full block validity proofs generated from unmodified Bitcoin Core.

## TL;DR

I have compiled Bitcoin Core v28's consensus code to `riscv32im`, run it inside a RISC0 zkVM, and generated mainnet consensus valid block STARK proofs. Not a reimplementation of the rules, but Core's actual code. Including - `interpreter.cpp`, `SignatureHash`, `CheckTransaction`, `arith_uint256`, the merkle and retarget code, and `libsecp256k1`. There is a live board proving Bitcoin's history from genesis at [bitcoinghost.org/hazync](bitcoinghost.org/hazync), open to anyone to view or contribute. There are genesis-anchored proofs you can download right now. Checking one takes milliseconds on a laptop with no GPU, no node and no peers. If you'd rather install nothing at all, there is a page that checks a proof in your browser, on your own device at [bitcoinghost.org/hazync/verify/](bitcoinghost.org/hazync/verify/). The code is at [github](github.com/bitcoin-ghost/hazync). After lots of review and LLM audits I haven't found a way to make the guest accept an invalid chain. I need people to play and try breaking it.

*N.B - This needs no soft fork, no new opcodes, no permission and no token, and there is nothing to buy. It is MIT licensed. This is an approach to proving Bitcoin that doesn't reimplement any of it, and I would like the community involved.*

## The problem

A new node today re-executes approximately seventeen years of blocks. Lots of data, compute and time spent, and at the end of it you know exactly what everybody else already knew. We all do that work because we have no way to check anyone else's.

That problem is worth finding a solution for. There is currently no artefact that fully solves the problem. A validity proof properly implemented could prove every block from genesis to the tip and fold the proofs into one. There have been several attempts at this. ZeroSync is probably the best known. Raito and Shinigami built real Bitcoin validation in Cairo, and I tried both. Essentially the scenario is this - A new node syncing to the network fetches headers, verifies a single receipt, and validates only the short tail nobody has proven yet. It's the same full-consensus security as replaying the chain yourself, because the proof is that replay, but it's only done once and checked by everyone.

## The failures

Every attempt at this that I know of, including my own first two, works the same way. You read the consensus rules, you design a circuit to match them, and you prove the circuit. The problem is that you're left holding a circuit based on your reading of consensus. In the back of your mind you're always left wondering if this matches Core in every single edge case? The common paths are easy to answer, but there are lots of these edge cases and any single error destroys the entire trust argument. Every rule must match Core exactly.

Bitcoin has no separate written specification. Core is the original specification, so a circuit that reimplements the rules is a second implementation of consensus that has to agree with the first one forever. There is a parallel line of work by others where attempts at extracting Core's rules into an executable specification are ongoing. My favourite being Hornet (Developed by Toby Sharp), a declarative spec written against Core's behaviour. It's a genuinely good direction, and it would make this much easier. But I'm impatient, and even consensus attempts such as Hornet will still leave doubt in people's minds, at least for the short-term. So, why not just prove using the reference client?

It's worth being precise here, because Core has not been idle. `libbitcoinconsensus` offered a narrow script-verification API for about ten years. It saw very little uptake, and was removed in v28.0 - the same version I compile. Its successor, `libbitcoinkernel`, is extracting the whole consensus engine, UTXO set and all, into something other software can link. Neither is a specification. Both are the same code, packaged to be called rather than described. That is Core's own answer to how anybody else gets consensus right, and it is the same answer as this post. Don't restate the rules, run them.

Using Core's code there is nothing to keep in sync, because the thing being proven is the same C++ the network already runs. You don't get to be wrong about what a rule means, because you never wrote the rule down. The cost is that Core's C++ has to run inside the zkVM. That's the engineering, and it's the rest of this post.

### My initial attempts

Prior to deciding to use Core's code, I tried to build my own circuits.

Nova / Halo2 - Hand-crafted circuits from R1CS up. It taught me how much of this you have to get exactly right by hand, and precisely why you don't want to do it that way.

Cairo - Proving a Cairo program built out of the Raito and Shinigami stack. This wasn't a technical failure, I did get recursion over script validity working, folded two Bitcoin leaves into one root proof, and validated a real input with in-circuit ECDSA. In the end, most standard spend types validated through the interpreter. However, step counts and RAM were staggering, proofs were only comfortable near genesis, and I never trusted it. That feeling, plus the compute, sent me looking for a different approach.

## Solution

A pivot was clearly required if a true solution was ever to be found.

Hazync is that pivot turned into a system. Prove real Core consensus block by block and fold the proofs into a single small attestation of the whole chain.

A verified chain proof attests that:

 1. Every block from the anchor to the tip is valid under Core's rules.
 2. The UTXO set is exactly the committed accumulator root.
 3. The cumulative work is what it says it is. 

This is checked by verifying one proof, no re-execution and no trust in any peer. From a genesis anchor that binding is unconditional. If a mid-chain checkpoint is used then the anchor is a stated trust input.

### The block proof

One proof, over one block, and everything in the diagram below happens inside it. It is not a summary of what was checked somewhere else. The proof does not exist unless every line held. Every input of every transaction, through the real interpreter and real libsecp256k1. No sampling, and no `assumevalid`. A default Core IBD skips script and signature validation for blocks below its assumed-valid block, and this doesn't, which is most of the reason a modern block costs what it does further down.

```
+--------------------------------------------------------------+
| Hazync zk BLOCK PROOF (RISC0 image id 4722cec8)              |
+--------------------------------------------------------------+
| HEADER                                                       |
| PoW sha256d(header) <= target(nBits)                         |
| prev_block_hash -> links its position in the chain           |
| version gates / median-time-past rules                       |
+--------------------------------------------------------------+
| MERKLE                                                       |
| every txid -> header.merkle_root                             |
| CVE-2012-2459 mutation flag captured and rejected            |
+--------------------------------------------------------------+
| PER TRANSACTION (every input of every tx)                    |
| VerifyScript -- the real Bitcoin Core interpreter            |
| P2PK / P2PKH / P2SH                                          |
| P2WPKH / P2WSH / P2TR                                        |
| sighash -- legacy / segwit v0 / taproot                      |
| libsecp256k1 -- ECDSA + Schnorr verification                 |
| BIP141 witness commitment, recomputed in-guest               |
+--------------------------------------------------------------+
| CONSENSUS                                                    |
| no in-block double-spend                                     |
| coinbase value <= subsidy + fees                             |
| sigops (<= 80k) + block weight (<= 4,000,000 WU)             |
| coinbase maturity, BIP30 / BIP34 / BIP68 / BIP113            |
+--------------------------------------------------------------+
| STATE TRANSITION                                             |
| inputs spend from UTXO_root(n-1)                             |
| outputs create into UTXO_root(n)                             |
+--------------------------------------------------------------+
| PUBLIC JOURNAL (what any verifier reads back)                |
| block_hash     | prev_hash     | height                      |
| UTXO_root(n-1) -> UTXO_root(n) | cumulative work             |
+--------------------------------------------------------------+
```

The public journal box is different from the ones above it. It isn't a check, it's what the proof hands back. `prev_hash` and `block_hash` place the block in a chain, the two accumulator roots say which UTXO set went in and which came out, and cumulative work says what it cost. That is the whole interface, and it's what lets two of these compose without either one knowing anything about the other.

*N.B - The image id is the current pinned guest. Any change to the guest will mint a new one. More info on that below.*

### Folding proofs

```
+------------+    +------------+    +------------+   +------------+
| zk BLOCK 1 |--->| zk BLOCK 2 |--->| zk BLOCK 3 | … | zk BLOCK N |
+------------+    +------------+    +------------+   +------------+
      |                 |                 |                |
      v                 v                 v                v 
    [ 1..1 ]  -fold-> [ 1..2 ]  -fold-> [ 1..3 ]  -fold- ... -> [ 1..N ]
                                                                  |
each [1..k] = fold( [1..k-1] , zk BLOCK k )                       v
                                            +---------------------------------------+
                                            | ONE FOLDED PROOF                      |
                                            | [ 1..N ] = genesis -> tip             |
                                            | constant size · O(1) verify           |
                                            | runs on a phone / RPi / laptop        |
                                            +---------------------------------------+
```

Proofs fold two ways:

1. Sequential recursive chain - Each step verifying the previous proof.
2. Parallel range fold - Independent per-block proofs merge in a log-depth tree. Checking tip hash, accumulator root and cumulative work at every join.  

The range-fold proves Bitcoin's history and is embarrassingly parallel. You can spread it across many machines, and a smaller committed cluster keeps the tip current afterwards. This is enabled co-operatively via the proof party (see below).

The tree is not "any two proofs that happen to be adjacent". Only two aligned siblings of equal width may fold, and their parent is the next node up.

```
[1]   [2]   [3]   [4]   [5]   [6]   [7]   [8]
   \ /         \ /         \ /         \ /
  [1..2]      [3..4]      [5..6]      [7..8]
        \    /                  \    /
        [1..4]                  [5..8]
              \                /
               \              /
                  [ 1 .. 8 ]
```

That constraint was earned. It used to offer any adjacent pair, and that does not converge, because every fold produces a range that immediately becomes a new operand. Measured on the live board before it was fixed: 581 folds covering 96 blocks, where a tree needs 95. Eight ranges of width 2, eight of width 3, eight of width 4, which is O(n²) by inspection. Every one of those proofs was valid. They were just work nobody needed.

### The parts

Everything so far has been one proof, and how two of them combine. The rest of this post is the machinery that produces them at scale, so here is the whole thing on one page before I take the pieces separately.

```
  bitcoind — a full node, somebody's archive
                     |
                     v
+---------------------------------------------+
|  ARCHIVE BRIDGE               [ UNTRUSTED ] |
|  holds the whole Utreexo forest             |
|  emits one BUNDLE per block:                |
|    in-boundary data + root(n-1)             |
|    + an inclusion proof per spent coin      |
+---------------------------------------------+
                     |
                     |  bundles
                     v
+---------------------------------------------+
|  COORDINATOR                  [ UNTRUSTED ] |
|  hands out blocks · serves bundles          |
|  verifies every submission on CPU           |
|  chains verified ranges by tip              |
|  no GPU · never proves · never folds        |
+---------------------------------------------+
                ^         |
       receipts |         |  bundles
                |         v
+---------------------------------------------+
|  WORKER — your box                          |
|    hazync   identity · claim · submit       |
|    host     prove / fold / spine            |
|    guest    Core v28 compiled · 4722cec8    |
+---------------------------------------------+
                     |
                     |  a receipt anyone can download
                     v
+---------------------------------------------+
|  VERIFIER                                   |
|    hazync-verify   1.7 MB · x86-64 / arm    |
|    wasm            295 KB · phone-sized     |
|    ghostd          adopts a chainstate      |
+---------------------------------------------+
```

-  The guest - Core v28's consensus code compiled to `riscv32im`, identified by `METHOD_ID`  `4722cec8`. This is the only part you have to trust, and it is the only part you can rebuild from scratch and check for yourself.
-  The prover (`host`) - Runs the guest over a block and emits a receipt. It also folds two receipts into one, and absorbs a chunk into the spine. GPU optional.
-  The worker (`hazync`) - Your identity and your errand-runner. Holds the ed25519 key, claims a block, fetches the bundle, calls the prover, signs the result and submits it.
-  The archive bridge - Holds the full forest and emits one bundle per block. Untrusted, because a forged inclusion proof fails inside the guest. The worst it can do is stall you.
-  `bitcoind` - A full node feeding the bridge. Somebody needs one. You don't.
-  The coordinator - Hands out blocks, serves bundles and proofs, verifies every submission on CPU, and chains verified ranges by tip. Untrusted, because it can lose track of who did what and it cannot put a bad proof on the board.
-  The verifier - `hazync-verify` for x86-64 and aarch64, the browser verifier (WASM ~295 KB gzipped), and a C ABI for embedding. Needs the file and nothing else. No node, no network, no chain data.
-  ghostd - My Core fork, which adopts a chainstate from a receipt instead of replaying its way to one.

Three of those are marked untrusted on purpose, and that is the design rather than an accident of it. The bridge, the coordinator and every other contributor can all be hostile at once and the worst they achieve between them is wasting my time. The trust in this system is one artefact, it is a few hundred kilobytes of compiled C++, and anybody can rebuild it.

### So what stops a hostile bridge?

This is the first question I'd ask, so here is the answer in full. Say somebody stands up an archive bridge and starts handing out rubbish. It cannot invent a block. The guest checks `sha256d(header) <= target(nBits)`, and nBits comes from Core's real `CalculateNextWorkRequired` driven through the real `CBlockIndex`. A fabricated block that passes is exactly as expensive as mining one. It cannot swap data inside a real block. Every txid is hashed into `header.merkle_root`, and the header is what the proof of work is over, so changing a transaction changes the header hash and the PoW fails. It cannot forge a coin. Each leaf commits value, script, height, coinbase flag and creation-time median-time-past, and the inclusion proof has to verify against the previous root. Made-up coins do not have inclusion proofs.

What it can do is lie about the previous root itself, and this is the part worth understanding. A single-block proof says: given this prior root, this block is valid and yields that next root. Hand the guest a fabricated prior root and you get a proof that is internally perfectly sound. It just isn't about Bitcoin.

What catches that is composition, not the block proof. Every fold asserts that the left proof's outgoing tip hash equals the right's incoming one, and that the left's outgoing accumulator roots and leaf count equal the right's incoming ones. A fabricated root does not match its neighbour's, so it will not fold. The coordinator's chaining rejects it for the same reason one level up. And the spine gate demands `lo == 1`, `in_tip` equal to genesis, and the empty accumulator, so nothing built on an invented root can ever reach back to block 1.

So a hostile bridge gets you a valid proof of a chain that is not ours, which folds into nothing, chains to nothing, and can never be absorbed into the spine. It burns your GPU time. That is a denial of service against provers, and not a soundness break.

The honest way to put it is that a single-block proof is a conditional statement, and genesis anchoring is what discharges the condition. That is the whole reason the spine is defined as `[1..N]` with an empty accumulator at its left edge, rather than as a large pile of verified proofs.

### The spine

Composition is a tree but anchoring is sequential, because only the leftmost range can satisfy `lo == 1`. So at any moment exactly one artefact is the genesis-anchored head. It is the range proof `[1..N]` with the largest `N`, and I call it the spine. It is what "everything from genesis to N is valid" looks like as a file you can hold.

It advances by absorption rather than by re-folding.

```
spine [1..N] + chunk [N+1..M] -> spine [1..M]
```

One fold per absorbed chunk, however wide the chunk is. The tree does the parallel work and the spine takes the result, which is why the chunk should be as wide as the tree can make it.

Three things follow. It is always shippable, so after every absorption there is a complete genesis-anchored proof and no state in which the artefact is half-built. Advancing it is the only sequential step in the whole system, because proving and folding are unbounded in parallelism. And whoever advances it cannot corrupt it.

That last one is enforced twice, on purpose. A submitted spine is gated on `verify-range`. This enforces the full genesis in-boundary - `lo == 1`, `in_tip` equal to genesis, the empty accumulator, nBits, epoch start and the median-time window - on its exit code alone, with nothing parsed out of its output. The numbers are then read from `verify-any`, which prints one machine-readable line. `verify-any` on its own would accept a range that is valid but anchored anywhere, which is exactly the fabricated-anchor case the genesis pin exists to refuse. Parsing `verify-range`'s prose instead would mean trusting free text for consensus-relevant numbers. So gate on one, read from the other. Submissions are monotonic, and a head that does not advance is refused.

Nothing in the format marks a proof as "the spine". It is an operational role and not a type, and a verifier checks the same things either way.

### The frontier

The coordinator does not fold. Folding is GPU work and it belongs on contributors' boxes. What the coordinator does is verify and chain. Every submitted receipt is re-verified with the canonical guest on CPU, then chained to its neighbours by tip continuity, where `out_tip` of range *k* must equal `in_tip` of range *k+1*. The longest contiguous run starting at block 1 is the frontier.

That is what makes out-of-order proving work. Any block can be claimed and proved by anyone at any time, and the frontier advances as the gaps fill. Submit block 3 before block 2 and it verifies immediately, but the frontier holds at 1 until block 2 lands, and then jumps to 3.

So the board shows two numbers on purpose. Blocks verified, which is any range, and the genesis frontier, which is contiguous from block 1.

The difference between the spine and the frontier is the difference between one file you can check in 27 ms and a set of independently verified proofs I have chained by their boundary hashes. The frontier is checkable too, it is just N downloads and N verifications rather than one. When I say verify it yourself, I mean the spine.

## How Hazync works

Compile Core v28's consensus sources for the guest and prove those. The guest links the real `EvalScript` and `VerifyScript`, `SignatureHash`, `CheckTransaction`, `arith_uint256`, `ComputeMerkleRoot`, `pow.cpp`'s `CalculateNextWorkRequired` driven through the real `CBlockIndex`, and `libsecp256k1` for both ECDSA and Schnorr.

The only changes are two small portability shims, plus libc and unwinder glue. One is a 32-bit integer overload in `serialize.h` so 32-bit riscv serialises byte for byte the same as a 64-bit build. The other routes SHA-256 through RISC0's accelerator, byte-identical to Core's own. Both are a few lines of added code in `patches/`, and neither touches consensus logic. There is also a no-op shim layer for Core's mutex, thread-safety and logging headers (the guest is single-threaded), in `coreshim/` (equally not consensus logic).

The pieces not compiled from Core are a thin, self-contained slice. The subsidy halving schedule and the script-flag activation heights. The subsidy is a six-line transcription of `GetBlockSubsidy`. The flag schedule is differentially tested against Core's `GetBlockScriptFlags` at every activation boundary and both exception blocks, CI-enforced. This proves the guest's flags are a sound superset of Core's. The retroactive base flags can only ever cause more rejection, never less, which is the direction that preserves soundness but does allow the guest to reject a block Core would accept. It doesn't on any block tested, but a superset is what is proven, and I'd rather say so publicly than let someone find it in `SECURITY.md`.

Even the compiled retarget is cross-checked against the actual on-chain `nBits` at every one of the 476 mainnet retargets.

One trap worth flagging for anyone reproducing this. C++ static constructors don't run on the bare-metal guest, so Core's global tagged-hash midstates (`HASHER_TAPSIGHASH`, etc) are junk until you call `__libc_init_array()` once at the start. If you miss that, every taproot sighash is silently wrong while everything else passes.

Libsecp256k1 on 32-bit riscv has no `__int128`, so it builds with the 10×26 field and 8×32 scalar backends, running as plain software. I kept the real libsecp on purpose. I prototyped routing field arithmetic through RISC0's bigint accelerator but it measured a net loss (about ten percent slower), so it never shipped. Swapping in `k256` measured roughly 6× on ECDSA verify in-guest, 4.7–5.3× end to end on real ECDSA spends. The k256 experiment never touched Schnorr, so taproot key-path spends got nothing from it. It has been removed from the tree and lives in git history.

Core keeps the UTXO set in a database but a zkVM can't. So I had to build an accumulator, and it's the part I'd want audited hardest.

I use a Utreexo hash forest. The guest holds only the roots, verifies an inclusion proof for each spent coin, deletes it, and inserts the block's new outputs. This is a pure function from previous root and block to next root, which is what lets separate blocks compose under recursion with no shared mutable state. A bridge (any archive node) holds the full forest and hands out the proofs. It sits above consensus as a commitment layer and its security rests on SHA-256. Each coin commits its value, script, height, coinbase flag and creation-time median-time-past into its leaf, so the metadata that maturity and BIP68 rely on can't be forged.

The bridge is not trusted and does not need to be. A forged inclusion proof fails against the root the guest was given, and a fabricated root strands the resulting proof outside the genesis-anchored chain, as above. The worst a hostile bridge achieves is wasted GPU time. Note what the root being an input means, because it is the sharpest edge in the design. The guest never learns whether the root it was handed is the real one. It proves a transition between the root it was given and the root that follows, and nothing else. Every claim about which UTXO set is Bitcoin's, comes from chaining that transition back to an empty accumulator at block 1.

Reproducibility. Every recursive step commits the guest's own image identity (`METHOD_ID`, a hash of the compiled guest) and the final verifier insists it equals the true guest, so you can't slip a doctored guest into one level of the recursion. The id is reproducible. CI rebuilds the guest from scratch and checks it matches. You can do the same with `reproduce/Dockerfile`; it comes out byte-for-byte identical to `reproduce/METHOD_ID`. Nobody outside the project has ever re-derived it independently, and that is the single cheapest useful thing a reader of this post could do.

## What Hazync measures

Real STARK receipts produced on GPU. Numbers move as the code does, so the live board is the only place a current figure belongs, but below is the shape of it.

- Real `VerifyScript` costs about 2.1M cycles per input, roughly linear. On an L40S that's a few seconds of GPU time per input (~2.7s on the EC-dominated path / ~4.9s effective on a modern block).
- A genesis-to-170 recursive chain proves in 209 seconds (~1.2 seconds a block).
- Block 130,000 proves in 23 seconds on one L40S.
- Block 741,000 (Modern, segwit and taproot, 670 inputs, 394 UTXO leaves. This proves as 16 chunks across two L40S GPUs in ~55 minutes, of which the 16-way aggregation is 27 minutes).
- The parallel range-fold ran over blocks 1 to 550 in 1,077 seconds wall-clock. There are real spends in that range, the first being block 170, the Satoshi to Hal transaction.
- Block 958,250 carries a real 90-day BIP68 CSV time-lock on a live taproot spend (The coin was 90.2 days old, validated against the real median-time-past of its creation, accepted exactly as mainnet accepted it. Bump it 0.3 days younger and it is rejected with -42).
- The genesis-anchored spine the site serves right now is a STARK range receipt of 226,434 bytes covering blocks 1..1789. I downloaded it while writing this and verified it in 27 ms against the 1.7 MB standalone binary, guest id matching `reproduce/METHOD_ID`. There is also a ~295 KB WebAssembly (WASM) verifier peaking at 1.9 MiB of memory which is small enough for a phone, and an `aarch64` build of the verifier is published, so "a phone can check this" is a file you can download rather than a claim.
- The Groth16 SNARK wrap is built and wired in. 1,841 bytes for `[1..8]`, 3,441 bytes for `[1..1000]`. Size scales with the number of accumulator roots at the boundary, not the range length, so a full-chain wrap projects to a few KB.

The number that actually matters. Block 741,000 taking 55 minutes is the honest headline, not the tiny early blocks. That is not a wall, it's a bill. Pure Core proving of the whole chain is expensive for one person. Genesis to tip is roughly 17 GPU-years, and holding station at the tip afterwards is about six L40S-equivalents. Thankfully the work is parallelisable and the community can all help out with compute.

### Why the numbers are low

The board restarted from genesis on the 4th August 2026, when an internal audit forced a re-baseline onto the current guest `4722cec8`. That was the fourth reset in three days, and it is the price of changing the guest at all. Every guest change mints a new `METHOD_ID` and voids every proof made against the old one. This happened frequently during the development.

So whatever figure the board shows when you read this is not seventeen years of accumulated work. It is what has been re-proved since the last re-baseline, by one person. That is the context for it.

I'm also deliberately not throwing a large fleet at this yet. Proving at scale is waste while the guest can still change, because a soundness fix would delete the lot. I need review first, then compute. Doing it the other way round burns GPU-hours to produce receipts a fix would void.

## What Hazync proofs can't cover

I'd rather be straight about the trust boundaries than have people find them and assume I was hiding them.

- The two-hour future-time limit isn't provable. This depends on a wall clock the zkVM doesn't have, so it's left out on purpose. Its sibling, "time too old", is a function of the chain and is enforced.
- Data availability is not validity. Transactions are bound to the header by the merkle root, so a prover can't swap data. But the proof is not the data.
- A proof attests that a chain is valid and carries its cumulative work. It does not tell you it is the best chain. Choosing between competing tips is still most-work, exactly as it is today, and the work figure is in the journal so a verifier can do that comparison.
- Anchors. Genesis is unconditional but any later checkpoint is a stated trust input.
- Recursion binds each step to the guest image id, which is committed and asserted equally at every level. A single-block proof is unconditional and the recursive case is hardened and argued in `docs/SOUNDNESS.md`.
- The STARK receipt is transparent. The Groth16 wrap is not, and it inherits RISC0's trusted setup ceremony. The wrap is an optional transport optimisation for places that need a few KB rather than a few hundred, and nothing about the underlying proof depends on it.
- A proof does not make witnesses unnecessary to keep. Every guest change would mint a new `METHOD_ID` and voids old proofs. Re-proving needs the very signature bytes the proof was meant to replace. There is no way around it, you cannot prove a predicate over data you don't have, so somebody has to keep the witnesses, and today the network's answer is altruism through archive nodes.

Underneath it all, the usual foundations: RISC0 and STARK soundness, SHA-256, secp256k1.

## Possibilities

The proof isn't tied to any one implementation. It attests real Bitcoin consensus, anyone can verify it, and every proof produced is public. This potentially unlocks:

- IBD as proof-verification (Fetch headers, verify one receipt, validate only the unproven tail).
- Light clients that confirm full validity without downloading the chain. The verifier for that already exists and runs in a browser tab, ~295 KB gzipped.
- Contracts on other chains (Trusting Bitcoin's state without a federation).
- L2s anchoring with one small receipt.
- Any other Bitcoin implementation verifying instead of re-executing history.

The first one is not hypothetical. `ghostd`, (Bitcoin Ghost) my Bitcoin Core fork, adopts a chainstate from one of these receipts today. `host dump-snapshot` writes the proven UTXO set out of the bridge, and the node turns that into a real `loadtxoutset` snapshot, but only after the dump has been checked against the proof. So a snapshot always describes a proven set. Adoption is an RPC, `hazyncadoptsnapshot`, armed with `-hazyncadopt`. It can't be a start-up action, because Core's `ActivateSnapshot` needs the base block in the headers chain and a fresh node hasn't got one yet. Headers sync in minutes, so that costs nothing. 

There is one real change to Core's behaviour and it's worth naming rather than burying. Core will not load a snapshot at a height it doesn't already know, because `ActivateSnapshot` asks `chainparams` for an `assumeutxo` entry. Adoption accepts a height `chainparams` has never heard of, on the authority of a verified proof instead of a hardcoded hash. The proof reaches that path as an explicit argument and never ambiently, and the whole route is unreachable until the dump has verified. A hostile or buggy bridge cannot produce a snapshot `ghostd` will take. Tested end to end against real mainnet headers. Adoption loads exactly the proven coins, bases on the proven tip, and disables background IBD. The set stops at the proof's edge, so with an eight-block proof block 1's coinbase is there and block 9's is not. Restart with the proof and you return to the adopted chainstate. Restart without it and the node refuses, names the reason and the remedy, and does not come up on the snapshot. 

What is not done is a byte-identical chainstate at a height with real transaction volume. That is a GPU bill rather than a code gap, and it is the same bill as genesis-to-tip. The rest of the list above - light clients in the wild, other chains, L2s - is not built by anybody yet. The verifier a light client would need exists; the client does not.

## Proof party

The full chain is more proving than one box can do, and it shouldn't be one box anyway. Hazync ships with what I've been calling the Proof Party. Anyone can claim a block, prove it, fold it, and put their name on it. The proof with your name
attached is published to the board.

First create an Ed25519 identity with `hazync id <name>` (key kept locally), run `hazync run` and the coordinator hands you the next open block after the frontier. One block, not a range. Width one matters, because a failure then costs you a few seconds rather than a 67-minute commitment to a thousand blocks that all get discarded together. A bigger aligned chunk is opt-in with `hazync run <lo>-<hi>` if you want it.

The claim is held by a heartbeat while your prove is in flight. Stop beating and it reopens in an hour, and there is a hard ceiling of a day so a wedged-but-still-beating worker can't sit on a block forever. But a claim is advisory on top, not permission: `submit` accepts any height regardless of who claimed it. So a bug in the allocator can waste effort but can't lock a contributor out.

The hour applies to the worker that just failed, and that is deliberate rather than an oversight. Block 39,318 does not fail fast. Its bundle is 1,098,218 bytes against roughly 4 KB for its neighbours, and it hangs the prover for over an hour. When self-reclaim was briefly allowed, a worker retried that same block forever and proved nothing else. The lockout is the rate limit that stops one bad block eating a contributor.

You don't need a node. The coordinator runs co-located with an archive bridge and a full `bitcoind`. It serves each block's bundle - the in-boundary data, the real previous accumulator root, and the inclusion proofs for every coin the block spends - from local disk. No chain data, no `bitcoind`, no IBD. The CLI fetches what it needs. You prove locally, sign and submit.

Proving is not the only way to help, and this is the part I'd most like people to know. Folding is contributor work too, and it costs seconds where a prove costs minutes. `/api/foldable` offers the sibling pairs, `hazync fold` consumes them, and `run-workers.sh` takes `MODE=fold`, `MODE=mixed` or `MODE=spine`. Folding needs no allocation and a duplicate fold is harmless, so any number of people can fold at once. The spine only ever needs one worker.

The coordinator can't forge anything. It verifies each submission on CPU with the canonical `host verify-any`, rejects a receipt that doesn't match its claimed range, and credits nothing that fails. Ed25519 signatures fail closed if the crypto library is missing, rather than open. Its power is bookkeeping and a leader board.

It also isn't the only one allowed to exist. Several coordinators can already co-operate: a peer's proven and in-flight heights steer `pick` away from duplicate work, and a peer is explicitly not trusted, because the worst a hostile one can do is withhold work from my own provers rather than get a bad proof onto the board. If you'd rather not use mine at all, `docs/RUN_YOUR_OWN_COORDINATOR.md` builds you a party of your own.

What you need:

- Verifying somebody else's proof - any Linux x86-64 box, no GPU, a couple of GB of RAM.
- Proving early or small blocks - an NVIDIA GPU with the CUDA 12.6 runtime, or the CPU binary more slowly.
- Proving big modern blocks with thousands of inputs - 64 GB+ of RAM and a serious GPU.
- Running your own party - an always-on box with a full `bitcoind`, roughly 8-core, 32 GB, 1 TB+ NVMe.

One trap. The proving segment is what has to fit in memory (`HAZYNC_SEG_PO2`, each step up doubling the working set), and swapping is not a failure, so a RAM-tight box crawls rather than falling back. Set it to 20 or 19 explicitly if that's you.

A GPU makes this much faster but is not required. The quick-start has been verified end to end on a CPU-only box, `selftest` included, proving a real block.

Every submitted proof is downloadable from `/api/proof/<id>` so anyone can re-verify it themselves. Your name goes on the [board](bitcoinghost.org/hazync), and the proof with your name attached is public for anyone to check.

One proof per block is a property rather than a side effect, and it is the part with no test. A sceptic should be able to be handed one block and check it alone, and that is established by retention, not by the proving code. It has already failed once - an earlier fold path proved each height, folded, and threw the leaves away, so 800 blocks came out with no individual receipt of their own. There is now a nightly gate that walks every height the board calls proven and fails if any one of them cannot be handed over on its own.

Exactly one contributor has ever existed. One pubkey across every submission to date, and it's mine. That is the number I'd most like this post to change. There is no technical barrier to a second name on that board.

It is, in the most literal sense, a distributed re-proving of Bitcoin, and it's the piece I'm most looking forward to opening up.

## What Hazync needs

Adversarial review - The valuable result is a case where one of these proofs says valid when Core would reject. `SECURITY.md` maps the soft spots - the accumulator, the recursion binding, the metadata plumbing. `docs/SOUNDNESS.md` is the best first read. I'd like attention on the two shims and the accumulator, which is also where both external reviews independently landed. Two passes converging on the same two places is itself a finding. `docs/EXTERNAL_REVIEW.md` ranks the list by value-per-hour, and the top item is the one nobody has done - build the guest from `reproduce/Dockerfile` on your own machine and tell me whether you get `4722cec8`.

Compute - The method works, genesis-to-tip is GPU-hours and the parallel backfill is built to soak them up. **This is a research prototype, and proving at scale is wasteful while the guest still changes, because a change voids prior proofs.** Another reason why review comes first.

## Questions

The list below is roughly in order of how much a bad answer costs me. I'd rather be told now than by a bad proof later.

- Is compiling real Core to the zkVM right for the long run, or does following the live tip eventually force a hand-optimised circuit and drag back the reimplementation risk I just escaped? I prefer correctness over speed, which is why I chose to keep Core's code instead of pushing further with `k256`.
- The two shims carry a lot of weight. Is "identical on the test vectors" a strong enough argument for a consensus proof, or does each shim need a differential harness against the LP64 host?
- Is the Utreexo accumulator as a pure function the right trust boundary? It's the one non-Core component and it's load-bearing for recursion. If there is a bug I would put money on it being here.
-  The incoming accumulator root is an input the guest cannot check, and the seam assertions plus the genesis pin are the only things that make it mean anything. Is that layering sound, or is there a way to build a chain of internally valid proofs that satisfies every seam and still anchors to genesis dishonestly?
- Is a sound superset good enough for the script-flag schedule, or should that thin slice be compiled from Core like everything else? A superset can only reject more, which is the safe direction for soundness and the unsafe one for liveness.
- Is keeping `k256` out of the build the right call, or is there an argument strong enough to earn back the increased speed? Bearing in mind it bought nothing for Schnorr, so the taproot era is unaffected either way?
- Every guest change voids every prior proof. Is re-prove-from-genesis simply the honest price of a soundness fix, or is there a defensible way to migrate a recursion tower across a guest change?
- Height-gated rules are where I'm afraid of a hidden bug. It would hide in old blocks nobody looks at. What else changed over the timeline that I'm not gating correctly?
- Does anything on Core's consensus path lean on implementation-defined or undefined behaviour that resolves differently on bare-metal `riscv32im` than on the LP64 x86-64 the network runs, beyond the integer-serialisation case the shim covers?
- Should the guest link `libbitcoinkernel` rather than carve consensus sources out of Core by hand? The carving is the part of this I like least, because it is where "what did you leave out" lives, and the kernel is aimed squarely at that problem. Is it close enough to depend on, and would it survive a bare-metal riscv32im target with no threads, no I/O and no filesystem?
- Anchors. For a real deployment, is genesis-anchored the only proof worth having, or is a documented checkpoint anchor an acceptable trade? For soundness I lean to genesis-anchored only.
- Is inheriting RISC0's Groth16 ceremony acceptable for a consensus artefact, or should the transparent STARK be the only thing anyone is ever asked to rely on?
- Block 741,000 spends 27 of its 55 minutes on the 16-way aggregation. Is there a better fold schedule, or is that inherent to recursion depth?

What have I missed? If you were going to make one of these proofs lie, where would you look first?

## Closing

The question I actually doubted from the start was whether you could run Bitcoin's real consensus code inside a proving system at a cost a crowd with GPUs could pay. That question is answered. Real blocks, real signatures, real receipts, reproducible guest, all public and downloadable.

What Hazync requires next is external review and compute. I built the Proof Party capability to enable anyone with a CPU or GPU to help build the proofs together as a community. I do not have the funds to commission a professional audit, but I need more experienced eyes on this. If anyone has some free time and finds it interesting, I would appreciate feedback and suggestions.

### Verify it in a browser

There is a page that does the whole thing on your device [here](bitcoinghost.org/hazync/verify/).

Drop a proof on it. The verifier is the same Rust code as the command-line tool, compiled to WebAssembly (WASM), about 295 KB over the wire. It runs in the tab. Nothing is uploaded, nothing is asked of a server, and the answer comes back in a fraction of a second - roughly a quarter of a second for the current spine, and about 21 ms for a Groth16-wrapped range.

It matters that this is not an API, and the reason is the whole argument of this project in one line. If your device asks my server whether a proof is valid, then your device trusts my server. That would replace "trust that Core's developers picked a good `assumevalid` hash" with "trust some random person online", which is a larger assumption than the one we started with, not a smaller one. 295 KB is cheap enough to do it yourself, so you should do it yourself.

For the same reason there is no wasm-bindgen in the build. It would generate the JavaScript glue rather than having it hand-written, but it would also put a version-matched codegen step between the Rust source and the artefact you are being asked to trust. A verifier's entire job is to be checkable. This one builds with cargo build --release --target wasm32-unknown-unknown and nothing else.

The page reports the same three answers as the binary. Verified and genesis-anchored, cryptographically valid but a mid-chain segment, or bad.

### Thirty seconds to check a proof

```sh
curl -fLO https://github.com/bitcoin-ghost/hazync/releases/latest/download/hazync-verify-x86_64-linux-gnu
chmod +x hazync-verify-x86_64-linux-gnu
curl -f https://bitcoinghost.org/hazync/api/spine/proof -o spine.bin
./hazync-verify-x86_64-linux-gnu --json spine.bin
```

`Exit 0` is a valid, genesis-anchored proof. `Exit 1` is a proof that is actually bad. `Exit 2` is a valid SNARK over a mid-chain segment rather than an anchored range. Most proofs on the board are segments, so that is the correct answer and not a failure.

Two things are worth doing by hand with the output, because they are what make it a claim about Bitcoin rather than a claim about itself. Check `tip_hash`, which is in display order, against any block explorer at the same height. Check `guest_image_id` against `reproduce/METHOD_ID`, and against `method_id` from `/hazync/api/meta` - the same value under two names, and `4722cec8` is the first eight characters of it. All three must agree, because a proof that verifies against the wrong guest is a proof of the wrong program. `epoch_start_time` should read `1231006505`, which is Bitcoin's genesis timestamp, for any spine. If those ever disagree, that is the most interesting bug in the repository and nothing else in this post matters.

If you would rather not run an unverified binary, every release carries a PGP-signed `SHA256SUMS.txt`, and GitHub serves the same maintainer key from the account that publishes the releases. The commands are in `SECURITY.md`. Keep the asset filenames when you download as renaming makes `sha256sum -c` report "no file was verified", which looks like a broken signature and is not.

### Two hours to rebuild the guest

This is the item nobody outside the project has ever done, and it is the cheapest useful thing a reader of this post could do.

Every proof verifies against exactly one guest, identified by `METHOD_ID` which is a hash of the compiled guest, currently `4722cec8`. If that id is not independently reproducible, then the whole argument rests on my word about which program was proven, and the rest of this post is decoration.

One warning before you start, because it will otherwise look like a failure. A plain local build produces a different id on purpose. The compiled ELF embeds absolute build paths, so the id depends on where it was built. Removing that variable is exactly what the container is for. It fixes `HOME=/root` and puts the Core and secp256k1 sources at `/root/hazync-build` on every machine, so the same id falls out anywhere.

```sh
git clone https://github.com/bitcoin-ghost/hazync
cd hazync
docker build -t hazync-repro -f reproduce/Dockerfile  .
docker run --rm hazync-repro # prints the METHOD_ID
```

Everything else is pinned:

-  `Bitcoin Core v28.0`
-  `secp256k1 v0.5.1`
-  `RISC0 3.0.5`
-  `Ubuntu 22.04`
- Pinned Rust toolchain
- Committed Cargo.lock

*N.B - A cold build spends nearly all its time installing the RISC0 toolchain and compiling Core.*

Two independent builds, on two different machines, printing the same id is what reproducibility means here. CI does one of them. The second has never been anybody but me, and it is the one that counts. Tell me if you get `4722cec8`, or something else. Both answers are useful, and the second is far more useful. If you have a GPU going spare, claim a block and prove it.

Code and docs: [github.com/bitcoin-ghost/hazync](https://github.com/bitcoin-ghost/hazync)

Overview and live status: [bitcoinghost.org/hazync](https://bitcoinghost.org/hazync)

Thank you for reading. Tear it apart.

Defenwycke

## References

### This project

1. Hazync - Code, docs and releases: [github](https://github.com/bitcoin-ghost/hazync)
2. The board - Live status, the spine, and every proof on it: [hazync board](https://bitcoinghost.org/hazync)
3. Browser verifier - Drop a proof, checked on your device: [hazync verify](https://bitcoinghost.org/hazync/verify/)
4. Bitcoin Ghost (`ghostd`) - The Core fork that adopts a chainstate from a receipt: [bitcoin ghost](https://github.com/bitcoin-ghost/ghost)

### Worth reading before you attack it

5. `docs/SOUNDNESS.md` [github](https://github.com/bitcoin-ghost/hazync/blob/da9dc23e1e8e2d5f6c52a37e5241702f6dbca875/docs/SOUNDNESS.md) - The recursion and anchoring argument.
6. `SECURITY.md` [github](https://github.com/bitcoin-ghost/hazync/blob/da9dc23e1e8e2d5f6c52a37e5241702f6dbca875/SECURITY.md) - Every review round, what each found, and the soft spots.
7. `docs/EXTERNAL_REVIEW.md` [github](https://github.com/bitcoin-ghost/hazync/blob/da9dc23e1e8e2d5f6c52a37e5241702f6dbca875/docs/EXTERNAL_REVIEW.md) - What still needs outside eyes, ranked by value per hour.
8. `docs/SPEC.md` [github](https://github.com/bitcoin-ghost/hazync/blob/da9dc23e1e8e2d5f6c52a37e5241702f6dbca875/docs/SPEC.md) - Composition, the spine, and what a verifier checks.
9. `reproduce/` [github](https://github.com/bitcoin-ghost/hazync/tree/main/reproduce) - The container that mints the canonical `METHOD_ID`.

### What is being proven, and what it runs in

10. Bitcoin Core v28.0 - The consensus sources compiled into the guest: [bitcoin](https://github.com/bitcoin/bitcoin/tree/v28.0)
11. RISC Zero - The zkVM: [risc0](https://risczero.com) / [risc0 github](https://github.com/risc0/risc0)
12. Utreexo - Thaddeus Dryja, A dynamic hash-based accumulator optimized for the Bitcoin UTXO set, IACR ePrint 2019/611: [eprint](https://eprint.iacr.org/2019/611) / [reference implementation](https://github.com/mit-dci/utreexo)

### Prior and parallel work

13. ZeroSync - Proof-based Bitcoin sync: [zerosync](https://zerosync.org)
14. Raito - Bitcoin consensus client in Cairo, which my second attempt was built on: [raito](https://github.com/starkware-bitcoin/raito)
15. Shinigami - Bitcoin Script VM in Cairo, the other half of that attempt: [shinigami](https://github.com/starkware-bitcoin/shinigami)
16. Hornet - Toby Sharp, Hornet Node and the Hornet DSL: A Minimal, Executable Specification for Bitcoin Consensus: [web](https://hornetnode.org) /  [post](https://arxiv.org/abs/2509.15754) / [bitcoin-dev announcement](https://gnusha.org/pi/bitcoindev/d9583f04-1aec-442d-ab2f-fc10fa42252dn@googlegroups.com/)

> Written with [StackEdit](https://stackedit.io/).

-------------------------

