# Leaf Version as Flags

benthecarman | 2024-03-20 03:59:20 UTC | #1

After reading through https://delvingbitcoin.org/t/64-bit-arithmetic-soft-fork/397?u=benthecarman and having some discussion about OP_CAT it seems like it may be better to treat the leaf version as a set a flags rather than a version number. 

One potential problem is that the current leaf version is 0xc0 which has a binary representation of 11000000, so we'd have 2 flags that default on. 

Curious on people's thoughts on using it as flags vs just an incremental version number

-------------------------

