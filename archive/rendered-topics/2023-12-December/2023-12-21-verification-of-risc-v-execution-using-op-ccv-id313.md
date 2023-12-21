# Verification of RISC-V execution using OP_CCV

halseth | 2023-12-21 13:59:05 UTC | #1

Hi, I just posted a proof of concept tool that can trace the execution of RISCV-32 binaries (ELFs), and generate Bitcoin script that can be used to verify this execution on-chain: [Bitcoin Elftrace](https://github.com/halseth/elftrace)

It relies on the OP_CHECKCONTRACTVERIFY covenant opcode from [MATT](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-November/021182.html).

The way it uses is OP_CCV is by committing to a single 32-bit hash (a merkle root) in the output, then the spender of this transaction will modify this root according to the rules and commit the new root in the new output. 

So essentially is just needs a covenant opcode that lets you enforce the embedding of a dynamic piece of data in the output and a (static) taptree. This is exaclty what OP_CCV enables, but perhaps some of the other covenant proposals could be used for the same purpose.

Feedback appreciated!

TLDR: https://twitter.com/johanth/status/1737778712987287990

-------------------------

