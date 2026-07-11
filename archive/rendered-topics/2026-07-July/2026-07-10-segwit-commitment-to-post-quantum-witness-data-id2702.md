# Segwit commitment to post-quantum witness data?

sipa | 2026-07-10 23:13:25 UTC | #1

This is a shower thought I had after seeing the idea of a post-quantum witness:

[quote="ajtowns, post:23, topic:2603"]
Hmm, if we increased the block size because of the larger size of PQ signatures by adding an extension block area for PQ signature data, we could also put the extra overhead for ECC spends of P2MR or P2MR+PKR in that same extension block space, and arbitrarily reduce the cost of that overhead; having the first 32B or 64B per-input be free would be manageable, I think, and might be sufficient to make P2MR ECC paths economically competitive with p2wpkh and p2tr, even without abandoning batching.
[/quote]

Adding another witness area is certainly technically doable, and might be inevitable if/when large-scale usage of PQC schemes is needed, depending on how large the scheme's signature/keys are. However, if done in a similar way to the segwit witness area, it adds some complications that may pose a burder to adoption, which may be more time-sensitive now:
* New pqwtxid for transactions that includes also the pqdata in addition to the segwit data.
* New block commitment in coinbase to the pqwtxids, requiring mining infrastructure changes.
* Possibly a pqwtxidrelay relay feature, matching the [BIP339](https://github.com/bitcoin/bips/blob/c021a5f51ae9d3e71a41eac3dda6dc060fead35d/bip-0339.mediawiki) wtxid relay (something whose need was only discovered after segwit was deployed).
* Several places inside P2P logic that now need to track transactions by 3 identifiers instead of 2, for compatibility with older nodes/protocols that use wtxids.

I think it may be possible to avoid some of that headache for a future post-quantum witness area: **by placing a commitment to the pqdata of an input inside that same input's own segwit area**. That would make it so that wtxids still commit to all data relevant to a transaction's validity, allowing it to remain usable as the sole transaction identifier, and avoiding some of the complications listed above.

There are two downsides, but both are addressable I think:
* **Extra storage/bandwidth**: this obviously adds 32 bytes in a naive serialization format. However, since it's just a hash of other data, it could be omitted from these formats. More interestingly, this can be done as an independent optimization, local to the node implementation, or negotiated as a P2P protocol extension, so it doesn't burden deployment of the pqdata feature itself.
* **Impact on weight/cost**: the same 32 bytes in the segwit witness (i.e., 8 vbytes) are inevitable as pre-softfork nodes see them. However, since this is being discussed in the context of a new output type, it's also possible to just move **all** witness data to the pqdata area for spends of such outputs, allowing it to be costed arbitrarily, and it can start off with a cost of -8 vbytes to compensate for the pqdata commitment in the witness (of course, rounding up to 0 if it somehow remained negative still).

Without having thought too long about this, it seems like an obvious improvement to me over the approach used with segwit (though, the same approach couldn't have worked as well there, as witnesses under 128 bytes - which are common now - would have been counted as 32 vbytes still).

Thoughts?

-------------------------

