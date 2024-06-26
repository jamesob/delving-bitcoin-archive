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

real-or-random | 2024-03-07 11:00:44 UTC | #3

[quote="ZmnSCPxj, post:2, topic:224"]
Maybe the best bet would be to focus on the “pure math” SECP256K1 signature validations (i.e. have the `SIGHASH`es come from the CPU and then push them into the GPU for validation).
[/quote]

This may indeed work, and it has been discussed before:

https://bitcointalk.org/index.php?topic=3238.20
https://github.com/bitcoin-core/secp256k1/issues/1214

But I'm not aware that anyone is seriously looking into it, at least not currently.

[quote="ZmnSCPxj, post:2, topic:224"]
I believe SECP256K1 Schnorr signatures as used in Taproot can be batch validated, meaning you only do a single calculation to validate multiple transactions that use the keyspend path. Since only a single calculation is needed, there seems to be no advantage to using a GPU in that case.
[/quote]

Yes, but batch verification does not mean that the work of verifying $n$ Schnorr signatures is the same (or similar) to the work of verifying a single Schnorr signature. Batch verification of $n$ signatures will yield nice speed-ups as compared to verifying the $n$ signatures individually, but it's [at most a factor of ~2 in practice](https://github.com/jonasnick/secp256k1/blob/869e7097d9835945a1663a321239418ad2f93ca4/doc/speedup-batch.md).

-------------------------

CubicEarth | 2024-03-12 00:08:45 UTC | #4

I wonder if there could be an expanded set of data that accompanied each block that could make processing faster. The extra data would be preprocessed in some way. It would be large in size, perhaps 10% or even 150% of the size of the blockchain, but it would be much faster to verify. For users who had very fast and unlimited internet, they could prefer to download this extra data so that syncing could happen faster.

-------------------------

vostrnad | 2024-03-12 04:08:49 UTC | #5

You should take a look at utreexo, which works exactly how you're describing. Its main goal is different, reducing the size of the stored chainstate (to ~1 kB), but a direct consequence of that is fewer (~none) disk operations so it could also speed up block validation.

-------------------------

real-or-random | 2024-04-15 13:46:13 UTC | #6

[quote="real-or-random, post:3, topic:224"]
But I’m not aware that anyone is seriously looking into it, at least not currently.
[/quote]

Something to watch out for is the ZPrice competition:
https://www.zprize.io/

They're currently running a competition seeking for implementation improvements of ZK proofs and related tech, also using GPUs. See for example https://github.com/td-kwj-zp2023/webgpu-msm-bls12-377 for a submission to the competition that uses WebGPU to speed up multi-scalar multiplication (also known as multi-exponentiation), which is the most important building block in batch verification of ZK proofs, but also in batch verification of signatures. Though they work on a different elliptic curve, their work shows what's possible:
https://docs.google.com/spreadsheets/d/1JR8Rzern0DkXc8oHZWcGlPjxSCfP4n4-ZDrpYihgo8M

If I read this correctly, that's a multi-scalar multiplication of size 2^20 in 1277 milliseconds...

The winners of the competition should be announced soon.


edit: Okay, maybe this example is not all that impressive. That's 1.2 usec / scalar, and libsecp256k1 can do 4.3 usec / scalar on my laptop CPU (on size 2^15, I haven't tried 2^20). Anyway, it will be interesting to see the results of the competition...

-------------------------

