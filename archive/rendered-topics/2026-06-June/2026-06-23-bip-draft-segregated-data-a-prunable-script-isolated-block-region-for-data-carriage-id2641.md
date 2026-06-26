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

MrHash | 2026-06-24 22:42:09 UTC | #7

All full nodes need to process at the tip (288 blocks). That cost is CPU cycles (essentially validating the hash of each entry) offset by whatever the cost of the existing vector cost is.

SegData adds no additional network or storage as the data is moved not added.

Smaller nodes (or otherwise) which validate beyond the retention window can opt out of bandwidth/cpu/storage. This is a forward reduction in network/storage requirement.

I understand the concern is resources, my argument is that segdata is net resource cost negative for those opting out.

-------------------------

AntoineP | 2026-06-25 14:55:47 UTC | #8

[quote="MrHash, post:7, topic:2641"]
All full nodes need to process at the tip (288 blocks).
[/quote]

I don't understand what you mean. Every full node always processes all blocks content, whether it is right after the block was created (what i believe you call "at tip", am i correct?), when it catches up after some downtime, or when it performs IBD.

Adding a new data structure to consensus rules means every single full node will have to download and validate this data. (In this case as i understand it, download the SegData and verify the Merkle root commitment.)

Furthermore, this additional data needs to be served by some nodes. If most reachable nodes do not serve it, this would impose significant load and reliance on the few that do.

[quote="MrHash, post:7, topic:2641"]
SegData adds no additional network or storage as the data is moved not added.
[/quote]

I am unconvinced that you can simply assume so, but let's take it for granted. This is still a block size limit increase, and because there is unlimited demand for free replicated storage, i think we can expect that space to be filled if it's made available.

[quote="MrHash, post:7, topic:2641"]
Smaller nodes (or otherwise) which validate beyond the retention window can opt out of bandwidth/cpu/storage.
[/quote]

That comes back to my previous point, but no: every node that wants to fully validate the state of the system will have to process any additional data structure introduced.

-------------------------

murch | 2026-06-25 19:18:12 UTC | #9

Since the SegData would cost the same as witness stuffing, but would result in reduced data availability, why would it be attractive to users that want to embed data?
Given that most people aren't interested in improving support for data embedding, I doubt anyone else would consider this worth the effort of a softfork, but if it's not even attractive for the would-be users, what's the point?

-------------------------

MrHash | 2026-06-25 20:02:53 UTC | #10

Hi Murch

The rationale does explain this in the BIP but I'll reiterate. 

1. Availability is not guaranteed in Bitcoin anyway (pruning). 
2. The default mode is archival, retention is effectively guaranteed, just not forced to everyone.
3. Witness is now used because its cheaper than op_return. This is a side effect of pricing discount which segdata equals. There's no reason not to use it.
4. Historical precedent, op_return was successful in mitigating fake address usage. SegData can draw data out of op_return and witness.
5. There is no way for anyone putting data on the chain to not impose it on all nodes, this intent is not served at all at the moment. SegData allows proper structural treatment of data vs money.
6. Not moving to SegData reveals intent.
7. Having a dedicated data channel allows deprecation of existing vectors as happened with bare multisig.

So I think there are lots of reasons to take this proposal seriously.

-------------------------

MrHash | 2026-06-25 20:12:27 UTC | #11

Antoine, the node does NOT require the SegData extended serialization of the block to perform sync validation beyond the tip (new blocks), it is done on the base serialization (no data paylaod) and validation is complete and BC. This is explained in detail in the peer services bip as well as the construction of blocks. The segdata portion can be reconstructed/requested by syncing segdata peers and validated in addition, which rehashes the entries to validate the reference hashes.

It is NOT a block size limit increase, it conforms to existing block size and tx weight.

There is no IBD/validation requirement of the segdata portion, similar to assumevalid but only for the data portion.

-------------------------

MrHash | 2026-06-25 20:32:36 UTC | #12

Just to clarify a full node validates the whole chain, it's not a block or tx size increase, the extra cost is a few hashes. If that is too much then we have some bigger problems to address with script exec in bitcoin. An opting out node can skip IBD and storage of the segdata part, while validating the tip (again just a few hashes). The default mode is full validation and retention, there is no likely concern with availability pressure, whereas the benefit to a opting out node would be huge in this case.

-------------------------

sipa | 2026-06-25 21:06:46 UTC | #13

How does this differ from the existing witness? It's equally discounted, and like all the rest of the block data not accessed anymore after validation (except for its effect on the UTXO set, which persists).

-------------------------

MrHash | 2026-06-25 22:05:58 UTC | #14

Hi sipa, segdata differs from segwit in that segdata is unspendable, not required to be download at all during sync or to be retained if opted out of, without affecting block validity. Witness still imposes the IBD and unpruned storage burden. To quote the BIP:

> Where SegWit separated signatures (validation data not needed once the containing block is buried) from transaction identity, SegData separates application payloads (arbitrary data not needed once the containing block is buried) from both transaction and witness regions.

-------------------------

cguida | 2026-06-25 23:23:44 UTC | #15

What's the point if there's no requirement to store it? What bitcoin noderunner is going to store nonmonetary data having nothing to do with bitcoin, for free? Would you do that? I wouldn't. No one will store this data.

If all you want is to force nodes to store a *commitment* to some data, then that's already been possible for a long time and requires no changes. Just stick a hash into a tweaked taproot key, or in an opreturn. Done.

-------------------------

MrHash | 2026-06-26 00:05:10 UTC | #16

I don't think the broad assumption you're making is well considered, because we all store the data for free now, so yes lots of people will store the data, and it's also the default in this proposal. It gives the option to consent to sync/storage to those who can't or don't want it, in part or in full. That's a benefit to both decentralization as well as consent, two very good dimensions to consider the proposal on. Offchain data hashes don't change, all other arbitrary data can move out of block and scripting vectors.

-------------------------

cguida | 2026-06-26 00:15:20 UTC | #17

Humans tend not to do things there is no incentive for them to do. This is a very basic economic principle.

Communism is what results from thinking like yours, where everyone is expected to provide unlimited free services with nothing in return. Either the system falls apart because no one wants to provide services, or everyone is forced to provide them at the point of a gun (and then it falls apart anyway).

Bitcoin does not operate on such thinking. Bitcoin operates on sound economic ideas, like paying people if you want them to do something.

-------------------------

cmp_ancp | 2026-06-26 01:27:55 UTC | #19

(post deleted by author)

-------------------------

