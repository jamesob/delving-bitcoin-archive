# Optimistic ZK verification using MATT

halseth | 2024-07-31 12:54:53 UTC | #1

Earlier I posted about [Elftrace](https://github.com/halseth/elftrace), a tool for creating Bitcoin Scripts to verify RISC-V execution: https://delvingbitcoin.org/t/verification-of-risc-v-execution-using-op-ccv/313 

Since then it has had some updates, and I wanted to share that the toolchain has had some additions making it easier to compile Rust without having to write custom assembly for every new program. It now also support doing input/output to the program using standard IO.

With this in place we are able to compile the **Winterfell STARK library** as a dependency, making it possible to _verify-the-verification_ of  ZK proofs in Bitcoin Script (this still requires CAT and a covenant).

A more detailed writeup is here, with example code: https://github.com/halseth/elftrace/blob/831f537bc1509bd45c350e103f7fc73aa818f7dd/docs/zk_verifier.md

-------------------------

