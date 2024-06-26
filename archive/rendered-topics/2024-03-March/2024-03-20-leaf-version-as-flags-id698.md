# Leaf Version as Flags

benthecarman | 2024-03-20 03:59:20 UTC | #1

After reading through https://delvingbitcoin.org/t/64-bit-arithmetic-soft-fork/397?u=benthecarman and having some discussion about OP_CAT it seems like it may be better to treat the leaf version as a set a flags rather than a version number. 

One potential problem is that the current leaf version is 0xc0 which has a binary representation of 11000000, so we'd have 2 flags that default on. 

Curious on people's thoughts on using it as flags vs just an incremental version number

-------------------------

vostrnad | 2024-03-20 05:32:04 UTC | #2

The leaf version must be even which reduces the number of flags to 7, and it cannot be 0x50 (01010000) because that's the starting byte for the annex. Further restrictions are desirable if we want to support some forms of static analysis, see [footnote in BIP341](https://github.com/bitcoin/bips/blob/b3701faef2bdb98a0d7ace4eedbeefa2da4c89ed/bip-0341.mediawiki#cite_note-7). Given all this, if we ever really need a flag it's probably better to implement it using OP_SUCCESSx.

-------------------------

Chris_Stewart_5 | 2024-04-12 16:28:32 UTC | #3

Hi Ben! 

I don't believe this can be changed anymore in v1 taproot. We serialize the entire byte representation of the leaf version into the tap leaf hash.

In other words, we commit to the full byte rather than an individual bit in the leaf version.

https://github.com/bitcoin/bitcoin/blob/0de63b8b46eff5cda85b4950062703324ba65a80/src/script/interpreter.cpp#L1828


[Whenever we roll out witness version 2, i believe we could change semantics around this](https://github.com/bitcoin/bitcoin/blob/0de63b8b46eff5cda85b4950062703324ba65a80/src/script/interpreter.cpp#L1929). However, when we do roll out version 2 we could totally redefine semantics for leaf versions any way. With the pace we deploy things currently, 256 leaf versions more than adequate imo.

-------------------------

vostrnad | 2024-04-13 13:47:08 UTC | #4

Committing to the leaf version byte (i.e. the full set of flags) is exactly what we want. I don't see how that would be a problem.

-------------------------

ProofOfKeags | 2024-04-15 19:16:34 UTC | #5

Committing to it would be fine. I think it just means that we have different interpretations of when we have "the next version". Rather than an enumerated mutually exclusive type we have conjunctions of features. AFAICT that's still compatible with committing to the whole thing.

-------------------------

