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

