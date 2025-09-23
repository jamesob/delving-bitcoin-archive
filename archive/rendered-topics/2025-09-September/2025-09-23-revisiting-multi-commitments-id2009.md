# Revisiting Multi-Commitments

moonsettler | 2025-09-23 11:55:30 UTC | #1

# Revisiting Multi-Commitments

Based on my conversations with @JeremyRubin, @salvatoshi and @reardencode this is an attempt to sum up the perceived tradeoffs of different approaches to proposed multi-commitments in bitcoin script.

* `OP_CAT`: `<x1> <x2> OP_CAT`
* `OP_PAIRCOMMIT`: `<x1> <x2> OP_PAIRCOMMIT`
* `OP_VECTORCOMMIT`: `<x1> .. <xn> <n> OP_VECTORCOMMIT`
* `OP_TWEAKADD`: `<t> <pub> OP_TWEAKADD` (counts as SigOp)
* `OP_SHA256TAGGED`: `<msg> <tag> OP_SHA256TAGGED`
* `OP_SHA256STREAM`: `OP_SHA256INIT <x1> OP_SHA256STREAM .. <xn> OP_SHA256STREAM OP_SHA256FINALIZE`

## In a world where script is restored (GSR):

* `OP_CAT` we naturally have, so all the concerns regarding introspection are already moot, big drawback is the witness malleability often requiring additional inefficiency introduced to scripts using it

Preferred:

* `OP_TWEAKADD` is pretty evidently useful. `OP_CAT` already enables state carrying with caboose, this would make it much neater (unlocks [MATT](https://merkle.fun)). `OP_TWEAKADD` necessarily counts as a SigOp and as a multi-commitment has worse witness malleability (as per [BIP](https://github.com/bitcoin/bips/pull/1944)) issues than `OP_CAT`.
* `OP_SHA256STREAM` makes certain `OP_CAT` and `OP_SHA256` heavy scripts more streamlined, does not suffer from stack element size limit for overall message length, post GSR the execution cost assigned to all operations ensures safety.

Neutral:

* `OP_SHA256TAGGED` is a minor optimization of `<msg> <tag> OP_SHA256 OP_DUP OP_CAT OP_CAT OP_SHA256` where tagged midstates can be pre-computed or cached for efficiency.

Bad fit:

* `OP_PAIRCOMMIT` and
* `OP_VECTORCOMMIT` are built around the assumption that consensus on `OP_CAT` can not be reached due to it being computationally complete with the other opcodes; thus making it hard to reason about second order effects.

## In a world without CAT (no state carrying):

Preferred:

* `OP_PAIRCOMMIT` does not enable
  * fine-grained introspection
  * state-carrying covenants
  * bigint operations
  * new arithmetic capabilities using lookup tables
  * manipulation of the taptree

Neutral:

* `OP_VECTORCOMMIT` more general form, provides optimization over multiple calls of `OP_PAIRCOMMIT`, for larger number of smaller pieces the cost benefits are more significant
* `OP_SHA256TAGGED` in the absence of `OP_TWEAKADD` and `OP_CAT` this opcode serves as a mildly less efficient but adequate pair-commitment 

Bad fit:

* `OP_SHA256STREAM` would introduce a lot if not most of the capabilities (or shenanigans) that serve as rationale for not activating `OP_CAT`, trivially allows for transaction and parent transaction introspection via [CAT tricks I](https://medium.com/blockstream/cat-and-schnorr-tricks-i-faf1b59bd298) and [CAT tricks II](https://medium.com/blockstream/cat-and-schnorr-tricks-ii-2f6ede3d7bb5)

## In a world without CAT (but with MATT):

Preferred:

* `OP_TWEAKADD` and 
* `OP_VECTORCOMMIT` are the basic building primitives for [MATT](https://merkle.fun), vector state carrying can also be achieved with multiple calls to `OP_TWEAKADD` but witness malleability could be a problem as per [BIP](https://github.com/bitcoin/bips/pull/1944) and complicate scripts unnecessarily.

Neutral:

* `OP_PAIRCOMMIT` in @salvatoshi’s opinion MATT is expected to carry a more complex state than 2 elements
* `OP_SHA256TAGGED` while it sounds more generally helpful, does not really enable the inspection or manipulation of the taproot script tree paired with `OP_TWEAKADD` due to lack of internal concatenation on the message part, still it is an adequate substitute for `OP_PAIRCOMMIT` and `OP_VECTORCOMMIT`

Bad fit:

* `OP_SHA256STREAM` same rationale for all cases without `OP_CAT`

Feedback and preferences or corrections and alternative suggestions are welcome!

-------------------------

