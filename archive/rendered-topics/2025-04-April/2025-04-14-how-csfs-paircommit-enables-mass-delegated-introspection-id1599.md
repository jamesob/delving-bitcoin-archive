# How CSFS+PAIRCOMMIT enables mass delegated introspection

josh | 2025-04-14 20:59:02 UTC | #1

*Disclaimer: This post is not an endorsement of any new opcodes. It is simply an exploration of new capabilities that could be enabled.*

## Terminology

*Mass delegated introspection* refers to the ability to authorize a wide range of potential transactions (e.g. different coinjoin variants, channel updates, or UTXO purchases) with a single signature, without knowing which will ultimately be executed.

## TLDR

CSFS+PAIRCOMMIT would make it possible to efficiently commit to many possible sighashes with a single signature, without introducing hash cycles and with the ability to commit to new sighashes on-the-fly.

## How it works

I had the opportunity to sit down with @reardencode at OP_NEXT and ask him a few questions about CSFS. For context, I've been thinking about how to solve [the bidding problem](https://delvingbitcoin.org/t/post-signature-cross-input-scripting-using-the-taproot-annex/1520/3?u=josh), which refers to the problem of efficiently authorizing many possible bids on different UTXOs, without knowing which will be accepted. At its simplest, this requires a delegated commitment to many possible txids. I imagine that this type of mass delegated introspection would be useful for many other use cases (like coinjoins, vaults, and channel factories), and I was curious if CSFS could enable this functionality.

We concluded that it can, but it would require a fairly bespoke script and an opcode which can verify a merkle proof, like PAIRCOMMIT. The idea is to:
1. Use CSFS to sign a merkle root
2. Use multiple PAIRCOMMITs to verify a leaf on a merkle tree of a fixed height
3. Use CSFS and CHECKSIG on a dummy private key to verify that the leaf equals the desired sighash.

Here is how the script might look (full credit to @reardencode for this implementation):

```
                  | <txsig> <sig> <path...> <leafhash>
DUP               | <txsig> <sig> <path...> <leafhash> <leafhash>
TOALTSTACK        | <txsig> <sig> <path...> <leafhash>
PAIRCOMMIT...     | <txsig> <sig> <root>
<key>             | <txsig> <sig> <root> <key>
CHECKSIGFROMSTACK | <txsig> <0|1>
VERIFY            | <txsig>
<G>               | <txsig> <G>
2DUP              | <txsig> <G> <txsig> <G>
FROMALTSTACK      | <txsig> <G> <txsig> <G> <leafhash>
CHECKSIGFROMSTACK | <txsig> <G> <0|1>
VERIFY            | <txsig> <G>
CHECKSIG          | <0|1>
```

It's a fairly complex script, but it works. We can extend this approach to multi-signature inputs, by duplicating the merkle root and verifying the threshold number of signatures.

It's worth noting that the leaf hash does not need to be a sighash. It can be any type of data, as long as there is an introspection opcode able to verify it.

## Limitations

The primary limitation of CSFS+PAIRCOMMIT is that the height of the merkle tree must be pre-committed to. This could be somewhat inefficient, if the merkle proof has to be unnecessarily large. Users could mitigate this by committing to multiple tapleafs with a varying number of PAIRCOMMITs, but this has its own costs.

A secondary limitation is that this approach can't be used with key path spends or in wallets that haven't pre-committed to the necessary tapscript(s). Users would need a special wallet and a larger, more complex descriptor, which isn't ideal. Alternatively, users might make a preparation transaction every time they want to use this functionality.

## Alternatives

I attempted to solve this problem in [this post](https://delvingbitcoin.org/t/post-signature-cross-input-scripting-using-the-taproot-annex/1520/3) by giving consensus meaning to the annex. The approach I described would work for both key path and script path spends without pre-commitments, but it's limited by the need to have multiple annexes. This is fine for certain multi-party protocols (like those involving buyers and sellers), but it's not ideal for coinjoins, vaults, channel factories, or the generalized use case.

An alternative approach would be to use the annex to commit to an additional script, which executes against the remaining elements on the stack, following a (hypothetical) `APPLYANNEXSCRIPT` opcode. This approach would be compatible with a soft fork and would allow for on-the-fly delegation across one or more signatures without needing pre-committed tapscripts with fixed merkle heights. Key path delegation, however, would require a new witness program and address type.

## Questions

1. *Would mass delegated introspection improve the practicality of coinjoins?* I imagine that the ability to make a single signature committing to many possible coinjoins would be useful, especially if funds are held in cold storage and one or more participants fail to sign. 

2. *Would channel factories become more practical with this capability?* My understanding is that channel factories suffer from the combinatorial number of state updates that users need to sign. I imagine that the ability to commit to many possible state updates with a single signature would be useful, but I'm curious what the community thinks.

3. *Is PAIRCOMMIT likely to be included in a soft fork, if CSFS is adopted?* Most discussions right now seem centered around CTV+CSFS, and I'm curious where PAIRCOMMIT currently stands. Are there obstacles to its adoption or alternatives under consideration?

4. *Is the need to pre-commit to CSFS and the height of the merkle tree acceptable?* It's not the most elegant solution, nor the most efficient, but it seems to work.

5. *Is it worth exploring annex-based alternatives?* Using the annex to commit on-the-fly to additional locking scripts could be an elegant way to do delegation across one or more signatures, without pre-commitments or complex wallet descriptors.

-------------------------

redundant | 2025-04-15 12:36:22 UTC | #2

Interesting write-up! One concern: committing to a merkle root with different sighashes in the scriptPubKey seems tricky, since you'd need the inputs to compute the sighash—but the inputs depend on the finalized scriptPubKey. The classic circular dependency issue. 

I'm assuming everything on the stack is from the witness stack, so I can see some interesting possibilities with a MuSig2 `<key>` and presigned transactions. I'll go back and read your bidding problem post—maybe that's the intended use case. If so, even combining the 'Schnorr trick' with MuSig2 keys could lead to some cool results.

-------------------------

stevenroose | 2025-04-15 13:25:40 UTC | #3

Just on PAIRCOMMIT itself. I think until now, people have always thought about using `OP_CAT` for building merkle trees and branches in Script. It's easy to leave vulnerabilities when doing that though, as the 64-byte tx vulnerability that the [Great Consensus Cleanup](https://delvingbitcoin.org/t/great-consensus-cleanup-revival/710) fixes has shown.

To do a merkle tree branch check safely with `OP_CAT`, you always have to first check the size of the pushes to avoid byte shifting attacks on the content of the nodes. The script would en up looking something like `OP_TOALTSTACK OP_SIZE <32> OP_EQUALVERIFY OP_2DROP OP_FROMALTSTACK OP_CAT OP_SHA256` for each node in the tree branch. This is 9 bytes instead of the 1-byte `OP_PAIRCOMMIT`. This is a simplification, in practice you'll also need to know which side of the tree you're taking, so another piece will have to be added for that at each level, but this will also have to be added to PAIRCOMMIT.

Not sure if a 9 bytes saving is sufficient motivation for an opcode.

-------------------------

