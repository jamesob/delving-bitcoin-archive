# Draft BIP for OP_TXHASH and OP_CHECKTXHASHVERIFY

stevenroose | 2023-09-30 22:21:50 UTC | #1

CC from the mailing list, so that we can have discussion here as well.

--- 
Hey all


The idea of TXHASH has been around for a while, but AFAIK it was never formalized. After conversations with Russell, I worked on a specification and have been gathering some feedback in the last several weeks.

I think the draft is in a state where it's ready for wider feedback and at the same time I'm curious about the sentiment in the community about this idea.

The full BIP text can be found in the attachment as well as at the following link:
https://github.com/bitcoin/bips/pull/1500

I will summarize here in this writing.

**What does the BIP specify?**

* The concept of a TxFieldSelector, a serialized data structure for selecting data inside a transaction.
  * The following global fields are available:
    - version
    - locktime
    - number of inputs
    - number of outputs
    - current input index
    - current input control block
  * For each input, the following fields are available:
    - previous outpoint
    - sequence
    - scriptSig
    - scriptPubkey of spending UTXO
    - value of spending UTXO
    - taproot annex
  * For each output, the following fields are available:
    - scriptPubkey
    - value
  * There is support for selecting inputs and outputs as follows:
    - all in/outputs
    - a single in/output at the same index as the input being executed
    - any number of leading in/outputs up to 2^14 - 1 (16,383)
    - up to 64 individually selected in/outputs (up to 2^16 or 65,536)
    - The empty byte string is supported and functions as a default value which commits to everything except the previous outpoints and the scriptPubkeys of the spending UTXOs.

* An opcode OP_TXHASH, enabled only in tapscript, that takes a serialized TxFieldSelector from the stack and pushes on the stack a hash committing to all the data selected.

* An opcode OP_CHECKTXHASHVERIFY, enabled in all script contexts, that expects a single item on the stack, interpreted as a 32-byte hash value concatenated with (at the end) a serialized TxFieldSelector. Execution fails is the hash value in the data push doesn't equal the calculated hash value based on the TxFieldSelector.

* A consideration for resource usage trying to address concerns around quadratic hashing. A potential caching strategy is outlined so that users can't trigger excessive hashing.
  * Individual selection is limited to 64 items.
  * Selecting "all" in/outputs can mostly use the same caches as sighash calculations.
  * For prefix hashing, intermediate SHA256 contexts can be stored every N items so that at most N-1 items have to be hashed when called repeatedly.
  * In non-tapscript contexts, at least 32 witness bytes are required and because (given the lack of OP_CAT) subsequent calls can only re-enforce the same TxFieldSelector, no additional limitations are put in place.
  * In tapscript, because OP_TXHASH doesn't require 32 witness bytes and because of a potential addition of operations like OP_CAT, the validation budget is decreased by 10 for every OP_TXHASH or OP_CHECKTXHASHVERIFY operation.


**What does this achieve?**

* Since the default TxFieldSelector is functionally equivalent to OP_CHECKTEMPLATEVERIFY, with no extra bytes required, this proposal is a strict upgrade of BIP-119.

  * The flexibility of selecting transaction fields and in/output (ranges), makes this construction way more useful
    - when designing protocols where users want to be able to add fees to their transactions without breaking a transaction chain;
    - when designing protocols where users construct transactions together, each providing some of their own in- and outputs and wanting to enforce conditions only on these in/outputs.

* OP_TXHASH, together with OP_CHECKSIGFROMSTACK (and maybe OP_CAT*) could be used as a replacement for almost arbitrarily complex sighash constructions, like SIGHASH_ANYPREVOUT.

* Apart from being able to enforce specific fields in the transaction to have a pre-specified value, equality can also be enforced, which can f.e. replace the desire for opcodes like OP_IN_OUT_VALUE.*
* The same TxFieldSelector construction would function equally well with a hypothetical OP_TX opcode that directly pushes the selected fields on the stack to enable direct introspection.


**What are still open questions?**

* Does the proposal sufficiently address concerns around resource usage and quadratic hashing?

* *: Miraculously, once we came up with all possible fields that we might consider interesting, we filled exactly 16 spots. There is however one thing that I would have liked to be optionally available and I am unsure of which side to take in the proposal. This is including the TxFieldSelector as part of the hash. Doing so no longer makes the hash only represent the value being hashed, but also the field selector that was used; this would no longer make it possible to proof equality of fields. If a txhash as specified here would ever be used as a signature hash, it would definitely have to be included, but this could be done after the fact if OP_CAT was available. For signature hashes, the hash should ideally be somehow tagged, so we might need OP_CAT, or OP_CATSHA256 or something anyway.

  * A solution could be to take an additional bit from each of the two "in/output selector" bytes, and assign to this bit "commit to total number of in/outputs" (instead of having 2 bits for this in the first byte).
    * This would free up 2 bits in the first byte, one of which could be used for including the TxFieldSelector in the hash and the other one could be left free (OP_SUCCESS) to potentially revisit later-on.
    * This would limit the number of selectable leading in/outputs to 8,191 and the number of individually selectable in/outputs to 32, both of which seem reasonable or maybe even more desirable from a resource usage perspective.

* General feedback of how people feel towards a proposal like this, which could either be implemented in a softfork as is, like BIP-119 or be combined in a single softfork with OP_CHECKSIGFROMSTACK and perhaps OP_CAT, OP_TWEAKADD and/or a hypothetical OP_TX.


This work is just an attempt to make some of the ideas that have been floating around into a concrete proposal. If there is community interest, I would be willing to spend time to adequately formalize this BIP and to work on an implementation for Bitcoin Core.


Looking forward to your thoughts

Steven

-------------------------

ajtowns | 2023-10-02 09:31:12 UTC | #2

[quote="stevenroose, post:1, topic:121"]
Does the proposal sufficiently address concerns around resource usage and quadratic hashing?
[/quote]

There's two good, general ways of addressing quadratic hashing with this sort of opcode that come to my mind:
1. (a) break it up into two phases, the first where you do a constant number of hashes over each byte of the tx and cache the result which adds O(txsize) computation and either O(1) or O(txsize) storage for the cache; then (b) combine a constant number of those cached hashes at runtime to implement the new opcode, so that it's both O(1) time and quite fast. That's what CTV does, and there was a bit of concern about how costly the first step was, and, if doing it on-demand, how complicated it is to share the just-in-time cached data across different threads (when the cache needs to be shared across different inputs without being recalculated, and those inputs may or may not be be being handled by different threads).
2. Allow the tx to waste time hashing things, but add that as additional "virtual weight" via the taproot annex as [envisaged in bip 342](https://github.com/bitcoin/bips/blob/e918b50731397872ad2922a1b08a5a4cd1d6d546/bip-0342.mediawiki#cite_note-13).

I think either of those could be made to work fine.

I think what you actually propose is more along the lines of "ensure that OP_TXHASH only ever hashes a constant data size", but I don't think that quite works here -- you're allowing a hash of random combinations of inputs' scriptSigs, which can be as large as a block, and I don't think (as specced) there's a decent way to cache and reuse those results.

[quote="stevenroose, post:1, topic:121"]
The flexibility of selecting transaction fields and in/output (ranges), makes this construction way more useful
[/quote]

Do you have specific/concrete examples of uses that this extra flexibility enables? Would you be up for doing a rough demo of what they'd look like in action something like https://github.com/jamesob/opvault-demo/ ?

[quote="stevenroose, post:1, topic:121"]
the other one could be left free (OP_SUCCESS) to potentially revisit later-on
[/quote]

I thought someone had made a post about this sort of thing being somewhat annoying as far as analysing miniscript goes, if this feature were to be included in miniscript, but I can't find the reference now. In any event, it seems pretty risky to have a potentially user-provided stack element be able to turn a script into "always succeed" behaviour. I think it would be better for the upgrade path to just be "if we want more txhash-y things, we'll introduce OP_TXHASH2".

Note that this differs from the checksig upgradability behaviour -- there we have `<sig> <unknownpubkey> CHECKSIG` be equivalent to `<sig> "" OP_NOTEQUAL`, and soft-fork in immediate script failure on invalid sig once we decide what "invalid" means for `<unknownpubkey>`.

-------------------------

stevenroose | 2023-10-02 10:30:09 UTC | #3

[quote="ajtowns, post:2, topic:121, full:true"]
I think what you actually propose is more along the lines of "ensure that OP_TXHASH only ever hashes a constant data size", but I don't think that quite works here -- you're allowing a hash of random combinations of inputs' scriptSigs, which can be as large as a block, and I don't think (as spec'ed) there's a decent way to cache and reuse those results.
[/quote]

I'm trying to do both, actually. I'm also charging a fixed validation weight.

For the prefix mode, the cache strategy would be to define some constant interval N (say 10) so that every N in/outputs the hash context for each field can be kept, making storage requirement O(#in/out) for the cache, while making the amount of data needed to hash on each occurrence be maximally N items.

I didn't think about the scriptSigs specifically. They can indeed be of arbitrary size and that can be a problem. Are there any other fields that can realistically be set at arbitrary size within policy limits? For those, we could have a field-specific cache. For scriptSigs, this would mean we'd have to store an extra 32-byte hash for each input, which isn't too bad. As most of scriptSigs are empty nowadays, the hash of the empty string can be cached so this would be free for anything segwitv0 and taproot.

[quote="ajtowns, post:2, topic:121, full:true"]
I thought someone had made a post about this sort of thing being somewhat annoying as far as analysing miniscript goes, if this feature were to be included in miniscript, but I can't find the reference now. In any event, it seems pretty risky to have a potentially user-provided stack element be able to turn a script into "always succeed" behaviour. I think it would be better for the upgrade path to just be "if we want more txhash-y things, we'll introduce OP_TXHASH2".
[/quote]

Yeah I also read this shortly after sending my e-mail. That would probably not be a good idea. But like Russell also replied in that thread, it could be remedied by just thinking long enough about what fields to expose so we don't need to redo them. This means that we might have one bit left in the current design to cover (or break up) something.

[quote="ajtowns, post:2, topic:121, full:true"]
Do you have specific/concrete examples of uses that this extra flexibility enables? Would you be up for doing a rough demo of what they'd look like in action something like https://github.com/jamesob/opvault-demo/ ?
[/quote]

I could do that, yeah. I'm thinking mostly of Ark atm, but almost anything CTV does, TXHASH can do with added flexibility to add fees. I can also do a version of [doubletake](https://github.com/stevenroose/doubletake) using OP_CHECKTXHASHVERIFY (that use case could be moved entirely to Bitcoin if we had CAT+CSFS).

-------------------------

ajtowns | 2023-10-02 10:55:27 UTC | #4

[quote="stevenroose, post:3, topic:121"]
Are there any other fields that can realistically be set at arbitrary size within policy limits?
[/quote]

Policy limits don't help here: quadratic hashing in a block is still a problem. In a block, the output scriptPubKey can also be arbitrarily large.

If you're not prehashing outputs, I think the caching doesn't work well if input ranges can overlap: a hash of inputs (1-3) and a hash of inputs (2-4) can't share a prefix-cache for any of their inputs. That's why [SIGHASH_GROUP](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2021-July/019243.html) avoided allowing overlapping ranges. That's much worse if you allow selecting arbitrary sets of inputs via a bitfield, rather than just a range; but even ranges give you $n(n+1)/2$ possible prefixes across the $n$ inputs?

-------------------------

stevenroose | 2023-10-02 12:59:17 UTC | #5

Prefixes are prefixes, like leading in/outputs. So only like "first N". Which can always share a cache as outlined in the draft BIP. Individual mode would be the problem, where you can pick any in/outs up to 64 currently, but maybe up to 32 in the alternative scheme.

I think consensus is less of an issue because you pay the validation weight in that case. The issue is that invalid txs don't pay the weight and they could exhaust resources. For free.

I think having a cached hash value for any field that can be over 32-bytes would be reasonable.

-------------------------

ajtowns | 2023-10-02 14:12:53 UTC | #6

I don't think there's much concern about invalid txs -- you can trivially replace an individual invalid tx consuming X^2 units of CPU with X invalid txs each consuming X units of CPU, for the same total consumption?

-------------------------

stevenroose | 2023-10-06 17:26:25 UTC | #7

I added a reference implementation to the BIP and also added an example implementation of a caching strategy in rust-bitcoin:

https://github.com/stevenroose/rust-bitcoin/blob/c54a09cb2dd334d6cdeec23dda55c6b6e7154d03/bitcoin/src/blockdata/script/txhash.rs

All large fields are cached individually and there are caches for the leading caches at intervals and caches for "all". So, like you said, this means that after every large piece of information in the tx has been hashed once, the number of hashed bytes per invocation of TXHASH has clear bounds.

-------------------------

reardencode | 2023-10-09 20:14:58 UTC | #8

This is super cool @stevenroose! It's nice to see a concrete full TXHASH proposal as something to think about.

A few comments (some of which we've already discussed offline, but I wanted to put into a public forum as well):

1. Given that BIP119 has working code and is already ready already, why not make OP_CHECKTXHASHVERIFY an upgrade path for non-32-byte hashes on OP_CHECKTEMPLATEVERIFY?
2. I wonder whether this OP_TXHASH + OP_CHECKSIGFROMSTACKVERIFY is more likely to get merged (and/or more useful) than adding these same OP_TXHASH modes as the sighash for a new Tapscript key type/version (a la APO).
3. With the above in mind, it seems like these modes are _mostly_ useful with at least 2 bytes of flags, so perhaps it would be worthwhile to reserve 1-byte flags for some pre-determined modes (possibly similar to Template Key). Then we would have an OP_TXHASH with 1-byte modes or multi-byte flags, OP_CHECKTEMPLATEVERIFY with 32, 33, or > 33-byte hashes, and Tapscript 33-byte keys with 64, 65 or > 66-byte signatures all sharing the same basic set of hash modes and flags.

Thanks!

-------------------------

stevenroose | 2023-10-10 06:31:52 UTC | #9

I can re-hash my thoughts in those points here, yeah.

About 1., that seems like a reasonable idea, in the scenario where CTV gets activated.

About 2., the semantics of TXHASH+CSFS and a sighash are different. The latter _only_ allows the sighash to be specified at sign time by the signer. The former also supports that, but also supports an alternative mode where the sighash properties are set by the policy beforehand. This could be useful, I really have to start listing use cases for these somewhere because I keep forgetting 😅

About 3., hmm, it might make sense. Though I'm always a bit weary about pre-defined modes. It might turn out that the most useful modes aren't included because we can't sufficiently foresee what people will want to do with this. I'm thinking of adding one extra special case "0x00" that is ALL, which is useful for a sighash while the default "empty" is a CTV (non-recursive ALL) mode.

-------------------------

reardencode | 2023-10-11 16:28:01 UTC | #10

Re. 2., absolutely. They are different. Part of what is provided by TXHASH+CSFS is provided by TxhashKey, and it may be more likely to reach consensus due to not adding _other_ uses of hashes for which all technical objections may not have been addressed. I admit that I'm not well versed in the objections to CSFS, but I'm confident that the technical objections to TxhashKey can be addressed.

Re. 3., I definitely understand your hesitation around pre-defined modes (we can see the challenges with such in my own objections to APO/APOAS). That said, I struggle to see the downside to providing some best effort pre-defined modes along with the complete flag set.

-------------------------

stevenroose | 2023-12-11 09:06:51 UTC | #11

The BIP is now no longer in draft state and there is a Bitcoin Core implementation as well:
https://github.com/bitcoin/bitcoin/pull/29050

-------------------------

reardencode | 2023-12-12 05:26:10 UTC | #12

Copying some bits of our chat on Telegram to this more accessible place:

### Does non-verify TXHASH make sense without CSFS or CAT too?

Yes, it can be used to validate equality between specific fields of the transaction (e.g. two input amounts are equal, or input and output scriptpubkey are equal).

Also, CSFS (obviously?) composes beautifully with TXHASH, and CAT plus TXHASH opens up an extremely broad design space.

### Why are input values in `TXFS_SPECIAL_TEMPLATE` (not in CTV)?

With CTV, the lack of input values allows protocol designers to awkwardly add fees via a specifically sized UTXO at spend time. With TXHASH, a more customized TxFieldSelector can be chosen to enable adding fees.

Including the values in this template mode helps remove the footgun of a specific hash being only usable with a specific UTXO size, and more clearly sets this template mode to hash everything possible that doesn't lead to a hash cycle.

### On the difference between adding CSFS and adding APO-style keys

There really isn't much difference. APO keys end up being 1-byte larger than taproot x-only keys, which is the same as the 1 extra opcode for TXHASH+CSFS. One benefit of APO-style keys is they would work with CHECKSIGADD as well.

In either case, we should consider whether adding "magic" keys for the Taproot internal key and/or external key makes sense (APO proposes <1> to represent the internal key); or whether opcodes to put those keys on the stack makes more sense (i.e. `OP_INTERNAL_KEY` and/or `OP_EXTERNAL_KEY`).

**edit**: Thinking more about this:

`OP_TXHASH` + `OP_CHECKSIGFROMSTACK` does open up some interesting possibilities in scripts compared to adding APO-style keys.

For example, a script requiring signatures from 2 keys on the same hash:

```
<sig> <sig> <txfieldselector>
OP_TXHASH OP_TUCK <pubkey> OP_CSFSV <pubkey> OP_CSFS
```

Also, I think OP_CHECKSIGADD can be emulated without undue pain. For 2-of-3, one of these `<sig> <txfieldselector>` pairs should be a pair of empties:
```
<sig> <txfieldselector> <sig> <txfieldselector> <sig> <txfieldselector>
OP_TXHASH <pubkey> OP_CSFS OP_TOALTSTACK OP_TXHASH <pubkey> OP_CSFS OP_FROMALTSTACK OP_ADD OP_TOALTSTACK <pubkey> OP_CSFS OP_FROMALTSTACK OP_ADD 2 OP_EQUAL
```
This is ugly, but only 1 vByte more than `OP_CHECKSIGADD` per key.

## Additional comments

* It seems like it might be worthwhile to pick a default mode for `TXFS_INPUTS` and `TXFS_OUTPUTS` (`TXFS_INOUT_NUMBER|TXFS_INOUT_SELECTION_ALL`?) where bytes 3 and 4 are not required. This makes the logic a tiny bit fiddly: if both inputs and outputs are selected either both or neither can use the default mode.
* It might be worth adding to the BIP some discussion of why committing to the control block is chosen over committing to the tapleaf_hash. At first I thought we might need both, but upon reflection the control block commitment implicitly commits to the leaf hash via BIP341’s script validation rules.

**edit**: Hmm, I think that if this TxFieldSelector mechanism is to be potentially useful in a future introspection opcode, it'll need some way to select the current input's script code. Could repurpose `TXFS_CONTROL`, but might be worth adding discussion of this to the BIP.

-------------------------

reardencode | 2023-12-12 05:22:09 UTC | #13

(post deleted by author)

-------------------------

