# Proving UTXO set inclusion in zero-knowledge

halseth | 2024-09-16 13:01:59 UTC | #1

I’ve been working on a proof-of-concept for proving UTXO set inclusion in zero knowledge, and I feel it is now ready for more eyes: https://github.com/halseth/utxozkp

The `utxozkp` tool was primarily made as a research effort to prototype a possible way of making Lightning channel announcements more private. With this in hand one can prove that a LN channel exists on-chain without revealing the specific UTXO.

It has various other applications as well of course - any setting where UTXO set inclusion can be used as an anti-DOS measure would benefit.

Note that in its current form one only proves that the prover can sign *some* UTXO, and there are no guarantees about the UTXO not being reused for multiple proofs. In the future the aim is to generalize this enough to make it possible to selectively reveal information like UTXO size, tapscript paths, output age etc.

**Proving time and proof size**

The tool is very much a rough draft at this point. On my laptop (Apple M1 Max, 32GB), proving time is around 6 min, and the result is a proof of UTXO ownership of 1.4 MB. Verification is much quicker at around 300 ms. Both timings assuming the UTXO set is already loaded into memory as a Utreexo data structure.

The running time and resulting proof size have intentionally not been optimized until the direction of the project is decided.

**Architecture**

`utxozkp` is based on the [RISC Zero](https://github.com/risc0/risc0) STARK platform and [Utreexo](https://dci.mit.edu/utreexo). For more information check out the [README](https://github.com/halseth/utxozkp/blob/c3402a72005d8d1058f758ed277df9e6cafcac72/README.md).

Any feedback and suggestions for the path forward are welcome!

-------------------------

1440000bytes | 2024-09-16 22:06:50 UTC | #2

[quote="halseth, post:1, topic:1142"]
**Proving time and proof size**

The tool is very much a rough draft at this point. On my laptop (Apple M1 Max, 32GB), proving time is around 6 min, and the result is a proof of UTXO ownership of 1.4 MB. Verification is much quicker at around 300 ms. Both timings assuming the UTXO set is already loaded into memory as a Utreexo data structure.
[/quote]

How does this compare with [aut-ct](https://github.com/AdamISZ/aut-ct)?

-------------------------

