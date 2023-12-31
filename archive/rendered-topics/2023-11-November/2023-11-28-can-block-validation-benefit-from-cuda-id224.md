# Can block validation benefit from CUDA?

ss01x | 2023-11-28 03:17:39 UTC | #1

One of the biggest bottlenecks for running a full node is validating each transaction in a block.
If I'm not mistaken, the current Bitcoin Core implementation uses all available threads on the processor.

I wonder if this process could benefit from introducing parallelism via CUDA computing (or another way of making use of the GPU).

-------------------------

ZmnSCPxj | 2023-11-29 09:43:05 UTC | #2

My naive understanding is largely no.

At least part of the validation involves looking up unspent transaction outputs in a database. This is cached in memory (IIRC) but I think random access to general memory would not fit most GPUs, which are generally targeted more for vector computations (i.e. some kind of uniform memory access).

Many CPUs have SHA256 hardware implementations and I imagine using that would be faster than pushing bytes into the GPU and then pushing the hashes back. Not to mention that the most consensus-important SHA256-use would be `SIGHASH` calculation, which would involve branching (i.e. the differences between `SIGHASH_ALL`, `SIGHASH_NONE`, and `SIGHASH_SINGLE`, as well as the absence/presence of `SIGHASH_ANYONECANPAY` changes the sequence of bytes that are fed into the SHA256 machine).

Maybe the best bet would be to focus on the "pure math" SECP256K1 signature validations (i.e. have the `SIGHASH`es come from the CPU and then push them into the GPU for validation).  However, again, GPUs like consistent single-instruction-multiple-data vectorized calculations. And I believe SECP256K1 Schnorr signatures as used in Taproot can be batch validated, meaning you only do a single calculation to validate multiple transactions that use the keyspend path. Since only a single calculation is needed, there seems to be no advantage to using a GPU in that case.

Note: I have not read recent bitcoin core code, so my knowledge may be out-of-date.

-------------------------

