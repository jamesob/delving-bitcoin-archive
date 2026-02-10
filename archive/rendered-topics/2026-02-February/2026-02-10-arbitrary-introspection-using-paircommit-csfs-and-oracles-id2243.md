# Arbitrary Introspection Using PAIRCOMMIT, CSFS, and Oracles

ademan | 2026-02-10 19:14:44 UTC | #1

# TL; DR

By combining `OP_CHECKSIGFROMSTACK` with a vector commitment, you can build public blockchain oracles that provide powerful introspection abilities to bitcoin scripts.
While oracle based verification is well understood, this scheme has unusually strong incentives for honest behavior relative to typical oracle constructions.
Using multiple oracles, and potentially slashing bonded collateral, can further strengthen these incentives.
In this scheme, trust may be reduced enough to support very advanced protocols, including MEVil-generating ones.
Since one motivation for preferring `OP_PAIRCOMMIT` over `OP_CAT` is to avoid enabling MEVil-generating protocols, this scheme slightly weakens the motivation for `OP_PAIRCOMMIT`.

# [`OP_PAIRCOMMIT`](https://github.com/lnhance/bips/blob/5221a4e75e983e7040f5a3d039fb62d775371500/bip-0442.md)

BIP-442 defines `OP_PAIRCOMMIT` which pops two items from the stack and pushes a hash which commits to both.
One of the motivations for `OP_PAIRCOMMIT` is to support vector commitments in Bitcoin script without enabling as much functionality as `OP_CAT` does.

# [`OP_CHECKSIGFROMSTACK`](https://github.com/lnhance/bips/blob/5221a4e75e983e7040f5a3d039fb62d775371500/bip-0348.md)

BIP-348 defines `OP_CHECKSIGFROMSTACK` which pops three items from the stack: a signature, a message, and a public key, and pushes whether the signature is correct given the message and public key.
`OP_CHECKSIGFROMSTACK` makes it significantly easier to consume oracle commitments in Bitcoin script.

# The Concept

This is likely obvious, but worth stating explicitly.

An oracle can sign a commitment to the contents of a Bitcoin block such that a native Bitcoin script can verify arbitrary facts about that block, its ancestors, transaction fields, prevouts, values, etc.

This scheme works with both `OP_CAT` and `OP_PAIRCOMMIT` but is mostly uninteresting if `OP_CAT` is activated.

Given a vector commitment `vc(x0, x1, x2, ...)` that commits to the items `x0..xN`, we can define a block commitment `bc(b)`.
The specific fields below are illustrative, almost anything derivable from block data could be committed to instead.

## Commitment Structure

### Input Commitment

```
ic(i) = vc(i.txid, i.vout, i.sequence, i.ssig)
```

### Output Commitment

```
oc(o) = vc(o.amount, o.spk)
```

### Transaction Commitment

```
tc(tx) = vc(
	tx.version,
	tx.nin,
	vc(ic(tx.in[0]), ic(tx.in[1]), ic(tx.in[2]), ...),
	tx.nout,
	vc(oc(tx.out[0]), oc(tx.out[1]), oc(tx.out[2]), ...),
)
```

### Block Commitment

```
bc(b) = vc(
	b.version,
	bc(b.previous),
	b.time,
	b.bits,
	b.nonce,
	vc(
		tc(b.tx[0]),
		tc(b.tx[1]),
		tc(b.tx[2]),
		...
	)
)
```

# Significance

It’s widely understood that complex, arbitrary computations can be verified on-chain if you’re willing to trust a third party oracle.
What is potentially interesting here is that oracle commitments to data derived from blocks have stronger incentives to behave honestly than typical oracle schemes.

In this scheme, the oracle only publishes a single signature per block over a deterministic commitment to blockchain data.
Any party can independently recompute the commitment from the blockchain and verify what the oracle signed.

This gives the scheme three properties that encourage honest oracle behavior.

1. **Universal verifiability**
   Anyone already validating the blockchain can verify the oracle commitments themselves.
2. **Cheating cannot be selectively hidden**
   Because there is only one commitment per block\* (see Caveats), the oracle cannot cheat selectively.
   All verifiers verify all (public) data commitments out of self-interest.
3. **Provability**
   Fraud proofs are trivial to construct and share.

These properties mean the oracle has a significantly reduced chance of cheating without detection, increasing the expected cost of dishonest behavior.
When multiple such oracles are combined, the chance of successful, economically rational cheating is also reduced, further diminishing incentives to attempt it.
As a result, this scheme may be sufficient to enable a wide array of advanced protocols, including MEVil-generating protocols (for instance AMMs) that I and others wish to avoid.
Since a key motivation for `OP_PAIRCOMMIT` is to avoid enabling exactly these kinds of smart contracts, this observation slightly weakens the case for `OP_PAIRCOMMIT`.

Furthermore, I suspect (but don’t know) that oracles could have BitVM secured bonds which could be slashed if they publish signatures over invalid commitments.
In this case, oracles would publish their signature and a proof that they calculated the block commitment `bc(x)` correctly.
This is perhaps the most interesting, and potentially offers a substantial improvement in incentives compared to other oracle constructions.
However, I’m not positive this can work, as I’m not well versed in the exciting developments in the BitVM arena.

But taken together, using multiple oracles producing trivially verifiable commitments, with slashable bonds, this scheme may make powerful introspection possible with only `OP_PAIRCOMMIT` and `OP_CHECKSIGFROMSTACK`.

# Further Thoughts

The general principle is that using a vector commitment an oracle can sign a commitment to an arbitrarily large set of data, and if this set of data can be easily verified, then users have a higher than normal assurance that the oracle cannot get away with cheating.
If BitVM bonds can be fashioned for these oracles, and if they could be made for other chains (I don’t see why not), it might enable an interesting class of bridges.
If it’s possible, it still requires careful consideration.
Bond values and funds secured can, without detection, become unbalanced to the point that good oracle behavior is no longer economically incentivized.

# Caveats

A reliable public publishing system is required to assume only one commitment per block per oracle.
In practice, something like publishing commitments on nostr should be sufficient.
Nevertheless, it is still possible for an oracle to attempt private cheating by collusion.
An oracle may collude with a smart contract actor to provide an invalid commitment only to that actor (so that they can defraud another smart contract actor).
Verifiers can still identify this kind of cheating with a bit more work, as long as the commitment ends up on chain.
Off-chain protocols using these oracles lose this protection entirely, and would need to verify the commitments themselves.

# LLM Disclosure

I'm not sure anyone cares on delving but just in case...
The original draft of this was 100% written by me, but I'm a pretty crummy writer (just look at my other posts).
ChatGPT helped me clean up the prose significantly, all edits were applied manually by me.
I tried to never take its suggestions verbatim, but there were times the suggestion was unambiguously better than anything I could come up with.
None of the concepts come from or are derived from ChatGPT output.
Even the overall structure is mostly from my first draft.

-------------------------

ademan | 2026-02-10 20:12:58 UTC | #2

(post deleted by author)

-------------------------

