# Segwit commitment to post-quantum witness data?

sipa | 2026-07-12 16:27:07 UTC | #1

This is a shower thought I had after seeing the idea of a post-quantum witness:

[quote="ajtowns, post:23, topic:2603"]
Hmm, if we increased the block size because of the larger size of PQ signatures by adding an extension block area for PQ signature data, we could also put the extra overhead for ECC spends of P2MR or P2MR+PKR in that same extension block space, and arbitrarily reduce the cost of that overhead; having the first 32B or 64B per-input be free would be manageable, I think, and might be sufficient to make P2MR ECC paths economically competitive with p2wpkh and p2tr, even without abandoning batching.
[/quote]

Adding another witness area is certainly technically doable, and might be inevitable if/when large-scale usage of PQC schemes is needed, depending on how large the scheme's signature/keys are. However, if done in a similar way to the segwit witness area, it adds some complications that may pose a burder to adoption, which may be more time-sensitive now:
* New pqwtxid for transactions that includes also the pqdata in addition to the segwit data.
* New block commitment in coinbase to the pqwtxids, requiring mining infrastructure changes.
* Possibly a pqwtxidrelay relay feature, matching the [BIP339](https://github.com/bitcoin/bips/blob/master/bip-0339.mediawiki) wtxidrelay feature.
* Several places inside P2P logic that now need to track transactions by 3 identifiers instead of 2, for compatibility with older nodes/protocols that use wtxids.

I think it may be possible to avoid some of that headache for a future post-quantum witness area: **by placing a commitment to the pqdata of an input inside that same input's own segwit area**. That would make it so that wtxids still commit to all data relevant to a transaction's validity, allowing it to remain usable as the sole transaction identifier, and avoiding some of the complications listed above.

There are two downsides, but both are addressable I think:
* **Extra storage/bandwidth**: this obviously adds 32 bytes in a naive serialization format. However, since it's just a hash of other data, it could be omitted from these formats. More interestingly, this can be done as an independent optimization, local to the node implementation, or negotiated as a P2P protocol extension, so it doesn't burden deployment of the pqdata feature itself.
* **Impact on weight/cost**: the same 32 bytes in the segwit witness (i.e., 8 vbytes) are inevitable as pre-softfork nodes see them. However, since this is being discussed in the context of a new output type, it's also possible to just move **all** witness data to the pqdata area for spends of such outputs, allowing it to be costed arbitrarily, and it can start off with a cost of -8 vbytes to compensate for the pqdata commitment in the witness (of course, rounding up to 0 if it somehow remained negative still).

Without having thought too long about this, it seems like an obvious improvement to me over the approach used with segwit (note that using this approach there would have meant a minimum of 33 vbytes per witness, much more than the 8.5 vbytes used by a taproot key path spends now).

Thoughts?

-------------------------

garlonicon | 2026-07-12 19:48:34 UTC | #2

I think ECDSA signature can simply commit to the quantum one inside R-value. And then, the quantum space can be just sigops-based: we allow up to N quantum signatures, which would have some constant size.

-------------------------

sipa | 2026-07-13 03:10:32 UTC | #3

A more worked out version of this idea.

* Abstractly, every transaction gains a per-txin witness style number (0 = segwit, 1 = pqdata, 2+ = future stuff).
* Every style is associated with a P2P protocol extension, so nodes can opt into receiving specific styles or not. A full node would be expected to receive all styles known to its consensus rules. The reason for having multiple styles is to allow each to have its own discount/costing function, as those might need to change over time depending on signature schemes that are adopted. To not impose a bound on future ones, they must be optional to fetch for those that don't have their cost rules known.
* Serialization:
  * Use flag byte 0x02 after the 0x00 marker (instead of flag 0x01 used by segwit) if at least one style>0 witness is present in the transaction.
  * The encoding of witness data remains the same, except there is an additional style byte for every txin.
* For relay to nodes that don't support certain styles, witnesses can be "collapsed" to style 0, by replacing them with a commitment to the full data. Specifically, with a witness stack consisting of a single element of 34 bytes, of the form 0xff + style_byte + tagged_hash(style_byte + serialized non-collapsed witness data).
* The wtxid and txid of a transaction are defined as the normal segwit ones, after collapsing all styles > 0.
* The weight of a transaction is defined as the that of the normal segwit one as well, plus any additions due to style-specific rules.
* All existing witness output types (P2TR, P2WPKH, P2WSH) get an "input has style=0" consensus rule.
* Transaction inputs with a single-element style=0 witness of 34 bytes whose first byte is 0xff, followed by a known style number, are outlawed. This makes it possible to recognize "this is a downgraded witness for which I should have been given the full data instead", without needing access to the UTXO being spent, or performing full script validation. Violations of this kind are a "witness stripped" type of error, so they don't cause the block/tx it's in to be marked (permanently) invalid.

So instead of having a "naive" serialization with both the full witness data, and a commitment to it in the normal witness, this just treats the abstract transactions as having at most one of the two. The backward compatibility is achieved by adding a "collapse" operation instead of (the equivalent of) witness stripping.

---

@garlonicon I think that works in theory, but it needs a rather ugly layer violation in that the p2p/serialization logic needs to pierce the script down to the opcode level, to understand whether/where/how many pqdatas are permitted.

-------------------------

ajtowns | 2026-07-13 19:33:38 UTC | #4

[quote="sipa, post:3, topic:2702"]
* Abstractly, every transaction gains a per-txin witness style number (0 = segwit, 1 = pqdata, 2+ = future stuff).
[/quote]

Conceptually, I think this means "every input has authorisation data that gets a particular weight formula applied to it -- legacy has the original 1:1 weighting, segwit has the 1:4 weighting, pqdata and future things have new weightings".

In particular, this implies that spending an input uses only one authorisation type; you don't have a taproot spend with a bip340 signature in segwit witness data and also an additional post-quantum signature via pqdata, eg. That prohibits the non-hard-fork "rescue protocol" method suggested in [this list post](https://gnusha.org/pi/bitcoindev/xXllZpuSNUfmizNVhO9lt8q7Wi-l5-7RsHBHnO2FmenEj52K8FF2hhoW1fg_UMRMkhYzzrXS9sGDsKaYfKxviiaQ3mIuesm-bfEII79EI8g=@proton.me/).

For composite approaches like TRv2, I wonder if that approach is really what we want? If you spend a TRv2 coin via an bip340 path, should you get any "pqdata"-esque discounts? If you had a script that requires you to reveal 10kB worth of preimage data and also provide a signature, and provide (a) a 64 byte bip340 signature, or (b) a 3kB post-quantum signature, should you get pqdata-esque discounts at all in case (a)? Should you get a discount on the full 13kB in case (b) or just the 3kB?

[quote="sipa, post:3, topic:2702"]
* The encoding of witness data remains the same, except there is an additional style byte for every txin.
[/quote]

Seems like this might as well be a minimal CompactSize?

[quote="sipa, post:3, topic:2702"]
* ... witnesses can be “collapsed” to style 0, by replacing them with a commitment to the full data. Specifically, with a witness stack consisting of a single element of 34 bytes, of the form 0xff + style_byte + tagged_hash(style_byte + serialized non-collapsed witness data).
[/quote]

I wonder if this would be better specified as an annex entry?

If we consider the annex as a place where we can extend the tx with additional "nSequence" and "nLockTime" like behaviours (eg, per-input locktimes, larger range relative locktimes, or block at height X has hash Y assertions), then it would probably be good to have those assertions be available independently of new transaction authorisation encodings/weightings, ie have the "collapsed" encoding includes the annex assertions (regarding locktimes, etc) still available, despite the other data being absent.

If it was desirable to have witness data in multiple styles (ie 1:4 ratio for hash preimages and 1:6 ratio for pqdata used for post-quantum signatures), having an annex commitment could be a reasonable way to signal that, eg:

 * tx encoding: `version / 0002 / inputs / outputs / witness-per-input / styled-witness-per-input`
 * annex encoding: "50" then a sequence of length-tag-value entries, ordered by tag, with no duplicate tags; tag 0 is the witness-style commitment, the value is an array of style/sha256 hash pairs
 * rules:
   * every input with a styled-witness, must include an annex commitment of that styled-witness
   * commitment of the styled-witness must
   * non-segwit-inputs and p2wpkh ad p2wsh can't have a styled witness
   * style-2 witness data can only be used for p2trv2 inputs, and can only be used for data that's read via CHECKPQSIG op

So a 41-byte annex of `[50 22 00 02 [hash] 04 01 40420f]` could represent a pqdata commitment plus a per-input lockheight of 1,000,000 (tag 1, little-endian encoding), eg. The "22 00 02 [hash]" component is derivable from the `styled-witness-per-input` data, but the bandwidth saving probably wouldn't be worth the complexity, I guess?

[quote="sipa, post:3, topic:2702"]
* For relay to nodes that don’t support certain styles, witnesses can be “collapsed” to style 0, ...
[/quote]

It's necessary to do things this way because if you don't support the style, you not only don't know how to interpret it, you also don't know how much data you can accept in a block -- is 4MB okay? 28MB? 500PB? So you have to discard unknown styles entirely and just store the commitment.

That also means that if anyone mines transactions with future/undefined witness styles, everyone will discard the associated data, so if a new style=3 pqdata2 is introduced one day, any blocks prior to the activation of that feature that has style=3 txs will be collapsed for everyone, not just nodes that haven't upgraded to support pqdata2.

-------------------------

sipa | 2026-07-14 22:40:16 UTC | #5

[quote="ajtowns, post:4, topic:2702"]
Conceptually, I think this means “every input has authorisation data that gets a particular weight formula applied to it – legacy has the original 1:1 weighting, segwit has the 1:4 weighting, pqdata and future things have new weightings”.
[/quote]

That's certainly the easiest approach, but it's not the only one. For example, the witness stack could be "typed" and different types of data can have different discounts, even within a single witness type. It's annoying because that may mean a need to track the types throughout the script execution logic too, but it's not infeasible. If more invasive changes to the scripting language are considered, you could have a design where the witness stack pretty much contains a sequence of *(pubkey,signature)* pairs (or just *signature* if the pubkey can be recovered from it) that are checked up front, and the script then contains assertions of the form "pubkey(hash) X signed".

But maybe taking a step back: how should cost accounting occur, if we have the freedom to redesign it in abstract? I think it should be a per-txin monotonic function of (I/O costs, storage/bandwidth costs, CPU costs). The first is a constant (UTXO lookup). The second is proportional to the serialized size of the witness(es), and objectively speaking this is independent of how that data is used. The third is a function of executed opcodes.

I could imagine a design where the witness stack consists of just (a) a declared computation budget number (used by the opcodes in the script) and (b) script input data. Each witness style is then a formula for mapping the (computation budget, script input size) to WU. Newly introduced opcodes can have new/different computation cost from existing one without needing a new witness style. It is only when the formula changes, so practically when the cost per script input byte changes, that a new style is needed. Perhaps with such a design, a single style per txin suffices?

[quote="ajtowns, post:4, topic:2702"]
Seems like this might as well be a minimal CompactSize?
[/quote]

Sure, why not.

[quote="ajtowns, post:4, topic:2702"]
I wonder if this would be better specified as an annex entry?

If we consider the annex as a place where we can extend the tx with additional “nSequence” and “nLockTime” like behaviours (eg, per-input locktimes, larger range relative locktimes, or block at height X has hash Y assertions), then it would probably be good to have those assertions be available independently of new transaction authorisation encodings/weightings, ie have the “collapsed” encoding includes the annex assertions (regarding locktimes, etc) still available, despite the other data being absent.
[/quote]

That seems reasonable. I think we could also simplify things by just permitting multiple annexes per txin, avoiding the complexity of encoding it into a single witness. Witness style commitments could then be an annex, but they could also be just something else, very much like annexes, but with a different prefix byte. Maybe that's just a aesthetical difference; witness style commitments feel much more like a P2P/serialization thing, while annexes a script thing.

[quote="ajtowns, post:4, topic:2702"]
That also means that if anyone mines transactions with future/undefined witness styles, everyone will discard the associated data, so if a new style=3 pqdata2 is introduced one day, any blocks prior to the activation of that feature that has style=3 txs will be collapsed for everyone, not just nodes that haven’t upgraded to support pqdata2.
[/quote]

I don't think that's the case. The consensus rule that no style=0 commitments to a style=3 witness are allowed itself would activate along with the consensus change that gives meaning to style=3 data. Before that point, they'd just be style=0 witness data that happens to look like a commitment, but be relayed as style=0, and everyone would accept it without giving it special meaning.

-------------------------

ajtowns | 2026-07-15 00:11:33 UTC | #6

[quote="sipa, post:5, topic:2702"]
For example, the witness stack could be “typed” and different types of data can have different discounts, even within a single witness type.
[/quote]

I think it would be a lot easier to deal with if different weighting rules were separated structurally? In that case the "type" is just the "style".

[quote="sipa, post:5, topic:2702"]
But maybe taking a step back: how should cost accounting occur, if we have the freedom to redesign it in abstract? I think it should be a per-txin monotonic function of (I/O costs, storage/bandwidth costs, CPU costs). The first is a constant (UTXO lookup).
[/quote]

I think you could have non-constant I/O costs by allowing multiple "UTXO" lookups per-input; eg looking up 7be22151bc96041932830d79c2af4472e40e9dedaa29018fff6757501e7855c6:0 could give you up to 10kB of script data in a pubkey, which if it were reused across multiple transactions might be more efficient than repeating the same snippet in multiple transactions.

Perhaps it's better to think of this as a way of dealing with storage costs reducing over time, that doesn't involve (a) hard forks, (b) predicting the future and locking in a schedule in advance, (c) having regular soft forks to put in place a new temporary limit, where the risk is that if some soft fork fails, we don't have any reasonable limit at all?

[quote="sipa, post:5, topic:2702"]
The second is proportional to the serialized size of the witness(es), and objectively speaking this is independent of how that data is used.
[/quote]

It's not quite clear to me if "independent of how that data is used" is the right call here. It might be that the decision matrix is for the next decade should be something like:

 * ideal block size if Q-day doesn't happen is 3.23MB -- that maximises decentralisation
 * ideal block size if Q-day does happen is 12.76MB -- that trades off some losses in decentralisation, versus avoiding a significant decrease in transaction capacity

If that's the case, then it might make sense to provide ~10MB of capacity in a "pqdata" area, but require as consensus that the pqdata area is only used for post-quantum signatures. That way if Q-day doesn't happen, people don't use the pqdata area and blocks stay closers to 3.23MB target (because a post-quantum signature uses a higher percentage of 10MB than an ECC signature does of 4MB), but if Q-day does happen, the additional capacity is already available.

[quote="sipa, post:5, topic:2702"]
Perhaps with such a design, a single style per txin suffices?
[/quote]

I think that's true provided you don't want to make tradeoffs like the above; but my impression is we probably do want to make tradeoffs like the above? That is, I think the ideal block size for TRv2 spends (eg) with current technology (node hardware, and post-quantum cryptosystems) is different before / after Q-day. I might be wrong ofc.

[quote="sipa, post:5, topic:2702"]
I don’t think that’s the case. The consensus rule that no style=0 commitments to a style=3 witness are allowed itself would activate along with the consensus change that gives meaning to style=3 data. Before that point, they’d just be style=0 witness data that happens to look like a commitment, but be relayed as style=0, and everyone would accept it without giving it special meaning.
[/quote]

I think your description there is just a different way of saying what I intended.

-------------------------

sipa | 2026-07-16 18:25:18 UTC | #7

[quote="ajtowns, post:6, topic:2702"]
[quote="sipa, post:5, topic:2702"]
I don’t think that’s the case. The consensus rule that no style=0 commitments to a style=3 witness are allowed itself would activate along with the consensus change that gives meaning to style=3 data. Before that point, they’d just be style=0 witness data that happens to look like a commitment, but be relayed as style=0, and everyone would accept it without giving it special meaning.

[/quote]

I think your description there is just a different way of saying what I intended.
[/quote]

Right, I see what you mean now.

I think the way to look at it is just that witness styles just don't "exist" unless there are consensus rules (or any rules) that need to observe them. Before a softfork that introduces style=3, there is just no way to hand style=3 witness data to a node, because it either doesn't know about it (pre-softfork software), or because it knows it doesn't need it (pre-activation).

-------------------------

ajtowns | 2026-07-16 23:08:14 UTC | #8

I think that means each new "style" introduction is a combination soft-fork, block storage and p2p upgrade much like segwit was, with the benefit that all the logic is pre-written with the first new style, so enabling new styles is just "define a new weighting rule, and the activation trigger". For nodes that upgrade to the new logic late, you'd still need to re-download blocks after activation to get the additional data. That could theoretically involve "invalidating" post-activation blocks that were previously valid if you're not able to obtain new style data that matched the commitment you already had.

So in theory just as complicated as segwit's deployment, but in practice much simpler because it's just repeating the same logic multiple times, I think.

-------------------------

