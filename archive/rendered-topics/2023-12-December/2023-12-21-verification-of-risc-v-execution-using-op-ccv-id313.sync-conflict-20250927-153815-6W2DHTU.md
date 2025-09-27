# Verification of RISC-V execution using OP_CCV

halseth | 2023-12-21 13:59:05 UTC | #1

Hi, I just posted a proof of concept tool that can trace the execution of RISCV-32 binaries (ELFs), and generate Bitcoin script that can be used to verify this execution on-chain: [Bitcoin Elftrace](https://github.com/halseth/elftrace)

It relies on the OP_CHECKCONTRACTVERIFY covenant opcode from [MATT](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-November/021182.html).

The way it uses is OP_CCV is by committing to a single 32-bit hash (a merkle root) in the output, then the spender of this transaction will modify this root according to the rules and commit the new root in the new output. 

So essentially is just needs a covenant opcode that lets you enforce the embedding of a dynamic piece of data in the output and a (static) taptree. This is exaclty what OP_CCV enables, but perhaps some of the other covenant proposals could be used for the same purpose.

Feedback appreciated!

TLDR: https://twitter.com/johanth/status/1737778712987287990

-------------------------

sjors | 2025-03-26 14:08:28 UTC | #2

IIUC you're using the newly proposed `OP_CCV` as well as `OP_CAT`, but otherwise only existing op codes. It seems one particular annoyance you need to work around when emulating a 32 bit system is the [31 bit limitation](https://bitcoin.stackexchange.com/a/122944/4948) of `CScriptNum` .

Would your code get significantly easier and produce more compact leaves by allowing slightly bigger numbers? IIUC the Great Script Restoration project goe straight to 64 bit, but it seems just a few extra bits would do the trick here? https://rusty.ozlabs.org/2024/01/19/the-great-opcode-restoration.html 

Are there other currently disabled op codes that would help?

-------------------------

