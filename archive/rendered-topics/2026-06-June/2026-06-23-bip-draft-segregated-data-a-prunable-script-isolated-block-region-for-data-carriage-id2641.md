# [BIP Draft] Segregated Data: a prunable, script-isolated block region for data carriage

MrHash | 2026-06-24 08:34:51 UTC | #1

Dear all,

I've drafted companion BIPs for **Segregated Data (SegData)**, a soft-fork block region for arbitrary data (data without an intent to transfer value). SegData *entries* are committed in-block through a separate Merkle root and validated at the tip by every node. Beyond the standard retention window a node may prune them individually or in bulk, so the storage cost falls on the operators who choose to keep the data rather than on everyone.

This is not an effort to promote data carriage, rather to give it a designed structural home. OP_RETURN mitigated UTXO bloat by drawing data out of fake outputs, but its bytes are still synced in full and stored permanently by every node. SegData offers to carry the same data in a prunable region at the witness discount, so a carrier has sensible reasons to migrate. The OP_RETURN precedent shows the approach works in principle. Altering the existing vectors is deliberately out of scope.

**Design.** It follows BIP-141 patterns. The commitment is a coinbase output over a separate Merkle root, blocks gain a base and an extended serialisation, the weight formula is extended, and *reference* outputs are witness v2, value-zero, unspendable, and excluded from the UTXO set. No new sighash is needed.

Two specific properties carry most of the weight:

* **Script isolation.** No opcode may read an entry's contents, now or in any future opcode. That is what keeps entries prunable, since consensus never needs to read them again, and what stops an entry gating a spend.

* **Depth-scoped validation.** Within the retention window every node enforces all rules. Beyond it a node may validate from the base serialisation and skip the region, so sync never depends on voluntary retention. The region's byte length is committed, so even a node that skips it still checks the block weight.

**Who migrates:**

* **Can.** Data no script needs to read, interpreted off-chain by indexers, such as application-layer assets, blobs, and content carried for timestamping or attestation. SegData carries the bytes, and a reference output binds them to the transaction. Current carriage is overwhelmingly in this category.

* **Cannot.** Data a script must evaluate at spend time, such as covenant-style constructs. Script isolation forbids opcode access, and the price is the same, so nothing pulls it across.

* **Unknown until measured.** Whatever stays on the existing vectors once an affordable consensual alternative exists. Its size is not knowable in advance, and making it observable is part of what SegData offers.

**Open questions I would value feedback on:**

* **Witness version and length slot.** v2 with a 36-byte program, marker- and length-disjoint from BIP-360 P2MR. Share v2 or take a dedicated version? This is allocation and coordination, not correctness.

* **Per-reference entry length.** Committing it would make per-transaction weight and feerate computable without holding the entries. The block-level region length is already committed. Is the larger reference worth it?

* **Scope boundary.** Restriction of existing vectors is left to future BIPs. Is that the right line?

* **Coverage-tier granularity** (peer services). Finer tiers help a node find a peer that retained a range, but shrink the anonymity set that protects retention depth from fingerprinting.

Consensus BIP: https://github.com/MrHash/bips/blob/4eeeb0afbb9d256d264225801e635d2df1cc875f/bip-segdata.md

Peer-services BIP: https://github.com/MrHash/bips/blob/4eeeb0afbb9d256d264225801e635d2df1cc875f/bip-segdata-peer-services.md

Also announced on bitcoin-dev ML, but not passed moderation yet.

Happy to take detailed discussion here. I'm sure there will be many more questions, i hope the rationale section covers most of the obvious questions.

Hash

X: @hashamadeus

Nostr: npub1tjfwajj3cfy25ujx02c7q3e7pzc27jasxakk9v0lsrkrewahpkesee5a0v

-------------------------

AntoineP | 2026-06-23 17:29:08 UTC | #2

[quote="MrHash, post:1, topic:2641"]
Beyond the standard retention window a node may prune them individually or in bulk, so the storage cost falls on the operators who choose to keep the data rather than on everyone.
[/quote]

I don't understand the goal here. Since you give it consensus meaning, any full node would have to process the new data structure and use up resources. After processing the data, the node would not have to keep it around, but this saves storage (the least expensive resource) at the expense of other more expensive resources.

Alternatively, you could simply have the commitment but not give it consensus meaning through a soft fork. Essentially commit a Merkle root in an `OP_RETURN`. This is already possible, and [done](https://opentimestamps.org/), today.

-------------------------

MrHash | 2026-06-23 18:14:32 UTC | #3

Hi Antoine,

Addressing the point about resources, SegData is designed to move data that would otherwise be in OP_RETURN, witness stuffed, etc. With the segregation, aside from the absolutely necessary tip validation, the segdata is not required either by IBD or storage, and validation is skipped, so node resources are potentially saved in multiple dimensions.

Addressing the other point, as you say anyone can timestamp, so this doesn't change that in principle. However we can't force people keep their data off-chain so this covers the case where people insist on putting it on chain, which is what is happening in the wild, but instead of OP_RETURN, creating a structural semantically deliberate region with intent which supports consensual retention.

I hope that answers the initial question. Please continue to press if there's something i'm missing.

-------------------------

cguida | 2026-06-23 19:31:06 UTC | #4

Why would bitcoin noderunners want to store nonmonetary data for free?

-------------------------

MrHash | 2026-06-23 22:09:06 UTC | #5

I'm not sure I understand your question. SegData gives node operators the choice not to store data. Existing vectors offer no such choice.

-------------------------

AntoineP | 2026-06-24 20:56:16 UTC | #6

[quote="MrHash, post:3, topic:2641"]
the segdata is not required either by IBD or storage, and validation is skipped
[/quote]

[quote="MrHash, post:3, topic:2641"]
I hope that answers the initial question. Please continue to press if there’s something i’m missing.
[/quote]

I believe there is. Either the data structure is part of consensus rules, and all full nodes need to process it, or it's not, and it's already possible today without a soft fork.

-------------------------

