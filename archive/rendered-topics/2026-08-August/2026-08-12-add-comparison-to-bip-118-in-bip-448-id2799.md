# Add comparison to BIP-118 in BIP-448

orfeas | 2026-08-12 21:00:20 UTC | #1

[BIP 448](https://bips.dev/448/) helpfully compares itself to [BIP 346 (OP_TXHASH)](https://bips.dev/346/) and [BIP 119 (CHECKTEMPLATEVERIFY)](https://bips.dev/119/), but not to [BIP 118 (SIGHASH_ANYPREVOUT)](https://bips.dev/118/). What's the diff of applications enabled by 448 and 118? According to the discussion in the [LNHANCE thread](https://delvingbitcoin.org/t/lnhance-bips-and-implementation/376), the main drawback of APO is that it's too specific to covenants, is this right?

-------------------------

cmp_ancp | 2026-08-13 01:42:09 UTC | #2

APO does implement the most exciting application of CSFS + TEMPLATEHASH, that is, rebindable signatures, and in a cheaper way. 

However, there are other applications of these opcodes that it doesn’t implement, like the cut of interactivety in certain protocols that depend on musig (e.g. ark), delegation, DLCs, etc. Even more, these opcodes could turn into flexible building blocks for other applications, specially after future softforks.

-------------------------

orfeas | 2026-08-13 02:57:14 UTC | #3

Some proposals have been rejected because they enable too much. Is there a similar fear for 448?

-------------------------

