# Benchmarking Bitcoin Script Evaluation for the Varops Budget (Great Script Restoration)

Julian | 2025-11-10 20:59:13 UTC | #1

Hello everyone interested in Great Script Restoration and the Varops Budget,

The main concerns that led to the disabling of many opcodes in v0.3.1 were denial-of-service attacks through excessive computational time and memory usage in Bitcoin script execution. To mitigate these risks, we propose to generalize the sigops budget in a new Tapscript leaf version and apply it to all operations before attempting to restore any computationally expensive operations or lifting any other script limits.

Similar to the sigops budget (which is applied to each input individually), the varops budget is based on transaction weight, a larger transaction has proportionally more compute units available. Currently, the budget is set to 5,200 units per weight unit of the transaction.

To validate that this approach is working and that the free parameters are reasonable, we need to understand how it constrains script execution and what the worst-case scripts are.

=== Benchmark Methodology ===

For simplicity, we benchmark the script evaluation of block sized scripts with the goal of finding the slowest possible script to validate. This block sized script is limited by:

\- Size: 4M weight units

\- Varops budget: 20.8B compute units (4M × 5,200)

To construct and execute such a large script, it must be looped until one of the two limits is exhausted. For example, a loop of OP_DUP OP_DROP would take an initial stack element and benchmark the copying and dropping repeatedly until either the maximum size or the varops budget is reached. Computationally intensive operations like arithmetic or hashing on large numbers are generally bound by the varops budget, while faster operations like stack manipulation or arithmetic on small numbers are bound by the block size limit.

For simple operations like hashing (1 in → 1 out), we create a loop like:

```
OP_SHA256 OP_DROP OP_DUP (repeated)
```

Other operations have different restoration patterns. For bit operations (2 in → 1 out):

```
OP_DUP OP_AND OP_DROP OP_DUP (repeated)
```

These scripts act on initial stack elements of various sizes. The initial elements are placed onto the stack “for free” for simplicity and to make the budget more conservative. In reality, these elements would need to be pushed onto the stack first, consuming additional space and varops budget.

=== Baseline: Signature Validation ===

Currently, the theoretical limit for sigops in one block is:

```
4M weight units / 50 weight units per sig = 80,000 signature checks per block
```

Using nanobench, we measure how long it takes to execute pubkey.VerifySchnorr(sighash, sig) 80,000 times. On a modern CPU, this takes between one and two seconds.

If we want the varops budget to limit script execution time to be no slower than the worst case signature validation time, we need to collect benchmarks from various machines and architectures. This is especially important for hashing operations, where computational time does not scale linearly and depends on the implementation, which varies between chips and architectures.

The varops cost of each opcode depends on the length of its arguments and how it acts on the data; whether it copies, compares, moves, performs hashing, or does arithmetic. More details can be found in the BIP: https://github.com/rustyrussell/bips/blob/guilt/varops/bip-unknown-varops-budget.mediawiki

=== How to Help ===

To collect more data, we would like to run benchmarks on various machines. You can run the benchmark by:

1\. Checking out the GSR prototype implementation branch:

https://github.com/jmoik/bitcoin/tree/gsr

2\. Compiling with benchmarks enabled (-DBUILD_BENCH=ON)

3\. Running the benchmark:

./build/bin/bench_varops --file bench_varops_data.csv

This will store the results in a csv and predict a maximum value for the varops budget specifically for your machine depending on your Schnorr checksig times and the slowest varops limited script. It would be very helpful if you shared your results by opening a PR [here](https://github.com/jmoik/varopsData) so we can analyze the data across different systems and verify if the budget is working well or has to be adjusted!

Cheers

Julian

-------------------------

ajtowns | 2025-11-10 09:02:45 UTC | #2

[quote="Julian, post:1, topic:2094"]
2. Compiling with benchmarks enabled (-DBUILD_BENCH=ON)
[/quote]

Probably want to specify the build options more completely if you want benchmark results to be comparable? (`CMAKE_BUILD_TYPE` as something other than `Debug` in particular)

(Also `#ifdef DEBUG` and `#ifdef USE_GMP` instead of `#if` in various val64 code)

[quote="Julian, post:1, topic:2094"]
It would be very helpful if you shared your results so we can analyze the data across different systems
[/quote]

Probably should provide a git repo to add the CSV's to via PR?

I'm seeing errors, which look like they lead to useless data, fwiw:

```txt
Script error: OP_DIV or OP_MOD by zero
653/875: MOD_DUP_100Bx2                 0.000 seconds (     0 Schnorrs,    0.0% varops used)
```

May want to fix those first?

[quote="Julian, post:1, topic:2094"]
If we want the varops budget to limit script execution time to be ...
[/quote]

Capping at 100% varops budget seems to make the data less useful in evaluating whether the varops ratios are accurate, than if no capping was in place? eg these XOR tests:

```txt
511/875: DUP_XOR_DROP_DUP_10KBx2        0.514 seconds ( 12714 Schnorrs,  100.0% varops used)
509/875: DUP_XOR_DROP_DUP_100KBx2       0.446 seconds ( 11051 Schnorrs,  100.0% varops used)
514/875: DUP_XOR_DROP_DUP_1MBx2         0.477 seconds ( 11807 Schnorrs,  100.0% varops used)
516/875: DUP_XOR_DROP_DUP_2MBx2         0.475 seconds ( 11769 Schnorrs,  100.0% varops used) 
```

don't seem to give any useful information about relative performance at different input sizes, because in each case the varops limit is hit?

I would have expected to be able to use the data generated either to produce per-hardware calculations of the constant-and-per-byte factors for each opcode (and thus review the hardcoded varops factors) or alternatively to be able to judge what would cause the worst case blocks for my hardware (particularly for old hardware) -- eg, "a block full of schnorr checks will take 5s to validate, but a block of X hash256 ops with Y bytes of data each will take 10s to validate".

Looking at <100% varops hashing results, I think I can calculate:

 * ripemd: constant = 0.322, per byte = 0.00251
 * sha256: constant = 0.362, per byte = 0.00306
 * hash160: constant = 0.549, per byte = 0.00299

Or normalized from "seconds per block over-filled with repeating script" to "schnorr ops per op", perhaps:

 * ripemd: constant = 0.006 schnorr ops per op, plus 0.000047 schnorr ops per byte
 * sha256: constant = 0.007 plus 0.000057 per byte
 * hash160: constant = 0.010 plus 0.000056 per byte

Or normalized from 80k schnorr ops per block to 20.8B compute units per block:

 * ripemd: constant = 1554, per byte = 12.1
 * sha256: constant = 1747, per byte= 14.8
 * hash160: constant = 2650, per byte= 14.4

Those compare to current varops figures of constant=0, per byte=10, I believe. I'm surprised that ripemd seems to be faster than sha256 for me. Presumably either taking the DUP costs into account or setting up an absurdly large stack so that DUP is unnecessary would generate more accurate figures than I've done above though. FWIW DUP/DROP benchmarks seems to imply figures of 276 compute units per operation plus 0.05 compute units per byte for me (as opposed to 0 + 1 per byte).

-------------------------

Julian | 2025-11-10 20:57:20 UTC | #3

Hi aj, thanks for taking the time,

[quote="ajtowns, post:2, topic:2094"]
Probably want to specify the build options more completely if you want benchmark results to be comparable? (`CMAKE_BUILD_TYPE` as something other than `Debug` in particular)

[/quote]

Yes I have been using the default `CMAKE_BUILD_TYPE=RelWithDebInfo`, since this is probably what most use. I am not sure if the binaries on bitcoincore.org are built in release instead, anyways the performance difference between the two seems to be negligible.

[quote="ajtowns, post:2, topic:2094"]
Probably should provide a git repo to add the CSV’s to via PR?

[/quote]

That’s a great idea, I have opened a repo here and will update the post: https://github.com/jmoik/varopsData

[quote="ajtowns, post:2, topic:2094"]
I’m seeing errors, which look like they lead to useless data, fwiw:

```

```

[/quote]

The script errors are expected for some benchmarks and can be ignored since they produce a time of 0 seconds and we are interested only in the worst times, but you are correct, I will add more specific stacks/scripts to benchmark OP_DIV / OP_MOD properly.

[quote="ajtowns, post:2, topic:2094"]
Capping at 100% varops budget seems to make the data less useful in evaluating whether the varops ratios are accurate, than if no capping was in place? eg these XOR tests:

```

```

[/quote]

With 20.8 B compute units, the scripts are already fairly long, so you don’t gain much from not capping here, of course you lose the reference on how many ops were executed and larger stack element sizes might be cut of early and therefore the time / input bytes might be shifted downwards slightly.

If no capping was in place, some scripts would run extremely long.

If we want to measure performance relative to input bytes, we don’t need to call EvalScript() at all and remove all the noise from restoring the stack and the rest of the script interpreter (duplicating a large stack element is quite slow).

This benchmark is not designed to measure time / input size, I wanted to ensure that the budget is sufficient for any input size and having similar values for those XOR script tells me that the XOR cost scales appropriately.

[quote="ajtowns, post:2, topic:2094"]
I would have expected to be able to use the data generated either to produce per-hardware calculations of the constant-and-per-byte factors for each opcode (and thus review the hardcoded varops factors)

[/quote]

If we take the 5,200 compute units per weight as constant or renormalize this is interesting indeed, but there are too many free variables, the 5,200 was initially derived from the 10x hashing factor (520 bytes for a single script element x 10, such that one hashing opcode could always pay for a current max size element).

Therefore, I assumed the 10x for hashing (as well as the other costs) as constant and kept the 5,200 as the free parameter.

[quote="ajtowns, post:2, topic:2094"]
Those compare to current varops figures of constant=0, per byte=10, I believe. I’m surprised that ripemd seems to be faster than sha256 for me.

[/quote]

Yes that seems reasonable, although a bit slow, did you get those values through a linear fit of the different input sizes? I am also surprised, on my machine RIPEMD is the slowest hashing algorithm by far and the slowest script overall, the only one going above 80,000 Schnorrs. But unlike SHA, RIPEMD is still limited to 520 bytes in tapscript v2, maybe your machine is very slow with SHA on large inputs?

I don’t think we want to assign different costs to different hashing algorithms, since this is highly dependent on the implementation.

[quote="ajtowns, post:2, topic:2094"]
FWIW DUP/DROP benchmarks seems to imply figures of 276 compute units per operation plus 0.05 compute units per byte for me (as opposed to 0 + 1 per byte).

[/quote]

I have also experimented with a flat cost for each operation, since we do have interpreter overhead and acting on an empty input is obviously more expensive than zero, but it does not seem to be that important since we do have the block size limit.

[quote="ajtowns, post:2, topic:2094"]
Or normalized from “seconds per block over-filled with repeating script” to “schnorr ops per op”, perhaps:

[/quote]

We have actually worked with this normalization initially and it is helpful when there is no varops limit, but felt like it is simpler to discuss these benchmarks by using the block sized script and comparing it to the known time of 80,000 Schnorrs.

-------------------------

instagibbs | 2025-11-13 16:25:14 UTC | #4

[quote="Julian, post:1, topic:2094"]
Using nanobench, we measure how long it takes to execute pubkey.VerifySchnorr(sighash, sig) 80,000 times. On a modern CPU, this takes between one and two seconds.

[/quote]

TIL it was that slow. I worry that benchmarking against the *worst case* as the new “average” is the wrong goal as this may potentially become the average way of interacting with Bitcoin (unlike using lots of signatures, which seems rare). Basically we should expect \~every block to approach that level of verification cost, so I’d expect we would target something we would be happy with for IBD or at tip with relatively turbulent mempools. *waving wildly* On the order of 100ms?

edit: Or maybe I’m wrong and the average GSR block would be significantly faster, just like now with sigops. :person_shrugging: Might be harder to predict what will actually be deemed useful vs “more sigops” which seem to have marginal utility?

-------------------------

Julian | 2025-11-13 16:33:51 UTC | #5

Since the limit of the computational budget is fairly high I don’t expect the average block to come close to this budget at all, so fully validating the block should be way faster than the worst case, probably not quite on the order of 100ms, doing some benchmarks on real blocks would be interesting.

But also I am not sure how much we should worry about IBD times, to my understanding the default IBD in core uses assumevalid with very recent checkpoint blocks, so the average user does not even fully validate \~99% of scripts.

-------------------------

ajtowns | 2025-11-13 22:22:48 UTC | #6

[quote="instagibbs, post:4, topic:2094"]
TIL it was that slow. I worry that benchmarking against the *worst case* as the new “average” is the wrong goal
[/quote]

Signatures are also cached, meaning that if we have seen the tx recently, re-verifying the signature for the block is much faster; whereas all the other logic isn't cached (there is a script cache, but that is only reused mempool-to-mempool and block-to-block, eg in a reorg, not mempool-to-block). So a high budget for non-signature operations likely has a worst impact on average validation time than a nominally-equivalent budget for signature operations.

Also, you can only have ~62k distinct signatures in 4MB of witness data, so using the full budget would presumably mean ~22% of your operations should/could be cache hits even if none of the data had been seen before.

(80k sigops is, of course, the figure from segwit v0/BIP141 which is 4x the original number of sigops per block. Taproot/BIP340-342 preserves that figure pretty closely, so that I think you could get about 79,988 BIP340 verify ops in a block, though it also introduces the potential for combining sig operations into a single validation to reduce computation time)

(Also, there's probably a difference between single-core cpu time and wall-clock time when multiple validatation threads are in operation, at least for average blocks (versus worst-case blocks where all the work is in a single 4MB witness script). 1000ms of cpu time, divided between 4 or 8 threads is already perhaps 150ms-300ms of wall clock time, after all)

[quote="Julian, post:5, topic:2094"]
doing some benchmarks on real blocks would be interesting
[/quote]

Doing benchmarks on the more powerful logic new opcodes enable might be more interesting; eg the [OP_CAT based zkp verifier](https://bitcoinmagazine.com/technical/a-zero-knowledge-proof-is-verified-on-bitcoin-for-the-first-time-in-history)? How many zkps (implemented that way) could we verify in a block, if 100ms (or whatever) of cpu time were the only constraint? Seems to have some 223k operations per script, 30k ADD/DUP, 20k IF/ENDIF/SUB, 10k SWAP/LESSTHAN/TOALT/FROMALT, 400 CAT/1SUB/DEPTH/SHA256, 1 CHECKSIG, etc.

-------------------------

instagibbs | 2025-11-14 15:27:00 UTC | #7

[quote="Julian, post:5, topic:2094"]
Since the limit of the computational budget is fairly high I don’t expect the average block to come close to this budget at all, so fully validating the block should be way faster than the worst case,

[/quote]

I think in that case, the budget should be lowered to whatever worst case target we would have in mind.

[quote="Julian, post:5, topic:2094"]
But also I am not sure how much we should worry about IBD times, to my understanding the default IBD in core uses assumevalid with very recent checkpoint blocks, so the average user does not even fully validate \~99% of scripts.

[/quote]

We should not justify new resource requirements based on practical hacks to get around the historical issues, imo.

[quote="ajtowns, post:6, topic:2094"]
Doing benchmarks on the more powerful logic new opcodes enable might be more interesting

[/quote]

Agreed!

-------------------------

