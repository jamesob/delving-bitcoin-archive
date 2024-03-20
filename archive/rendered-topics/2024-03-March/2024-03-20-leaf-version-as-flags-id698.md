# Leaf Version as Flags

benthecarman | 2024-03-20 03:59:20 UTC | #1

After reading through https://delvingbitcoin.org/t/64-bit-arithmetic-soft-fork/397?u=benthecarman and having some discussion about OP_CAT it seems like it may be better to treat the leaf version as a set a flags rather than a version number. 

One potential problem is that the current leaf version is 0xc0 which has a binary representation of 11000000, so we'd have 2 flags that default on. 

Curious on people's thoughts on using it as flags vs just an incremental version number

-------------------------

vostrnad | 2024-03-20 05:32:04 UTC | #2

The leaf version must be even which reduces the number of flags to 7, and it cannot be 0x50 (01010000) because that's the starting byte for the annex. Further restrictions are desirable if we want to support some forms of static analysis, see [footnote in BIP341](https://github.com/bitcoin/bips/blob/b3701faef2bdb98a0d7ace4eedbeefa2da4c89ed/bip-0341.mediawiki#cite_note-7). Given all this, if we ever really need a flag it's probably better to implement it using OP_SUCCESSx.

-------------------------

