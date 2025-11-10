# Benchmarking Bitcoin Script Evaluation for the Varops Budget (Great Script Restoration)

Julian | 2025-11-07 15:14:37 UTC | #1

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

    OP_SHA256 OP_DROP OP_DUP (repeated)

Other operations have different restoration patterns. For bit operations (2 in → 1 out):

    OP_DUP OP_AND OP_DROP OP_DUP (repeated)

These scripts act on initial stack elements of various sizes. The initial elements are placed onto the stack "for free" for simplicity and to make the budget more conservative. In reality, these elements would need to be pushed onto the stack first, consuming additional space and varops budget.

=== Baseline: Signature Validation ===

Currently, the theoretical limit for sigops in one block is:

    4M weight units / 50 weight units per sig = 80,000 signature checks per block

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

This will store the results in a csv and predict a maximum value for the varops budget specifically for your machine depending on your Schnorr checksig times and the slowest varops limited script. It would be very helpful if you shared your results so we can analyze the data across different systems and verify if the budget is working well or has to be adjusted!

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

