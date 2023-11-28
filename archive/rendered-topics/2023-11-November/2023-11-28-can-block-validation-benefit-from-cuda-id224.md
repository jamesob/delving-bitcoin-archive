# Can block validation benefit from CUDA?

ss01x | 2023-11-28 03:17:39 UTC | #1

One of the biggest bottlenecks for running a full node is validating each transaction in a block.
If I'm not mistaken, the current Bitcoin Core implementation uses all available threads on the processor.

I wonder if this process could benefit from introducing parallelism via CUDA computing (or another way of making use of the GPU).

-------------------------

