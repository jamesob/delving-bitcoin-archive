# Combined CTV/APO into minimal TXHASH+CSFS

reardencode | 2023-08-23 17:50:13 UTC | #1

Here's the latest version of my proposal, somewhat updated since posting to bitcoin-dev mailing list. Not sure how serious this is as a proposal (partly depends on the response to it), but I do think that this write-up is useful in clarifying the similarities, differences, and trade-offs of CTV and APO.

https://gist.github.com/reardencode/2aa98700b720174598d21989dd46e781

(I apparently can't paste the text in here because I'm a new user and it contains links)

-------------------------

instagibbs | 2023-08-24 14:44:22 UTC | #2

The original TXHASH mail says:

> 
> For similar reasons, TXHASH is not amenable to extending the set of txflags
> at a later date.  In theory, one could have TXHASH abort-with-success when
> encountering an unknown set of flags.  However, this would make analyzing
> tapscript much more difficult. Tapscripts would then be able to abort with
> success or failure depending on the order script fragments are assembled
> and executed, and getting the order incorrect would be catastrophic.  This
> behavior is manifestly different from the current batch of OP_SUCCESS
> opcodes that abort-with-success just by their mere presence, whether they
> would be executed or not.
> 
> I believe the difficulties with upgrading TXHASH can be mitigated by
> designing a robust set of TXHASH flags from the start.  For example having
> bits to control whether (1) the version is covered; (2) the locktime is
> covered; (3) txids are covered; (4) sequence numbers are covered; (5) input
> amounts are covered; (6) input scriptpubkeys are covered; (7) number of
> inputs is covered; (8) output amounts are covered; (9) output scriptpubkeys
> are covered; (10) number of outputs is covered; (11) the tapbranch is
> covered; (12) the tapleaf is covered; (13) the opseparator value is
> covered; (14) whether all, one, or no inputs are covered; (15) whether all,
> one or no outputs are covered; (16) whether the one input position is
> covered; (17) whether the one output position is covered; (18) whether the
> sighash flags are covered or not (note: whether or not the sighash flags
> are or are not covered must itself be covered).  Possibly specifying which
> input or output position is covered in the single case and whether the
> position is relative to the input's position or is an absolute position.
> 
> That all said, even if other txhash flag modes are needed in the future,
> adding TXHASH2 always remains an option.


I think adding upgrade hooks, as he says, is pretty fraught, vs "just use another opcode". Currently if someone reorders it, it just becomes trivially true do the hash type has to be committed to in the output to avoid "anyonecanspend" style behavior.

I hadn't read this in a while, and the closing part was interesting:

> The usual way of building a covenant, which we will call "*additive *
> *covenants*", is to push all the parts of the transaction data you would
> like to fix onto the stack, hash it all together, and verify the resulting
> hash matches a fixed value.  Another way of building covenants, which we
> will call "*subtractive covenants*", is to push all the parts of the
> transaction data you would like to remain free onto the stack.  Then use
> rolling SHA256 opcodes starting from a fixed midstate that commits to a
> prefix of the transaction hash data. The free parts are hashed into that
> midstate.  Finally, the resulting hash value is verified to match a value
> returned by TXHASH.  The ability to nicely build subtractive covenants
> depends on the details of how the TXHASH hash value is constructed,
> something that I'm told CTV has given consideration to.

-------------------------

reardencode | 2023-08-24 17:54:26 UTC | #3

> I think adding upgrade hooks, as he says, is pretty fraught, vs “just use another opcode”. Currently if someone reorders it, it just becomes trivially true do the hash type has to be committed to in the output to avoid “anyonecanspend” style behavior.

Thanks for bringing this up. As a rule, I think the `<argument> OP_TXHASH` should be treated as a single fragment, each of which has exactly one interpretation at each soft fork level. In cases where a script wants to accept a signature on several possible TXHASHes they should explicitly guard the opcode as I [now show](https://gist.github.com/reardencode/2aa98700b720174598d21989dd46e781#can-selection-of-op_txhash-mode-be-deferred-to-spend-time).

I do think that the benefit of upgradeable `OP_TXHASH` is more important than the ability to easily specify the hash type at spend time or compute it in script. My first thought here is that this is potentially a correct disjunction between the existing sigops and `OP_CSFS`: Existing sigops allow for safe spend time hash method specification because they combine the hashing and signature checking. Once the hashing operation and signature validation are separated the hash method should be specified in the output. This relates to the reason that bip118 settled on using a new key type. Existing key types hadn't pre-committed to being spendable with the new modes.

-------------------------

jamesob | 2023-08-25 19:10:26 UTC | #4

A few incidental remarks, sorry if too far afield.

[quote="instagibbs, post:2, topic:60"]
I think adding upgrade hooks, as he says, is pretty fraught, vs “just use another opcode”. 
[/quote]

I agree here that there's little practical difference between softforking to add a new opcode and softforking to enable new argument-based behavior. Clients still need to update, fork still needs to be deployed, and code savings on the Core side are pretty minor I think.

[quote="instagibbs, post:2, topic:60"]
I hadn’t read this in a while, and the closing part was interesting:
[roconnor quote about pushing stuff on the stack and hashing ...]
[/quote]

This is less a comment about roconnor's remarks per se and more about the notion of essentially assembling a sighash message on the stack, but:

I think it would be really unfortunate if we got to the point where we were using CAT and rolling SHA opcodes to slice and dice txn components into tailor made hashes. Not only am I skeptical that there are many uses that call for fundamentally different txn component combinations to be hashed, but it strikes me that doing this kind of stuff on the stack is more in the "computation" than "verification" camp, and so is a waste of space and time when it comes to L1. 

I think this explains my bias for "CISC" vs. "RISC"-style opcodes in Bitcoin. Of course the trade-off here is that CISC implies being able to anticipate things that people want to do, and I suppose there's no reason that both styles can't exist in script. Maybe this is worth elaborating on in another post. 

---

I think it would be helpful to have some kind of listing of uses that having CSFS enables (e.g. key delegation, although to be honest I can't explain this use much deeper than just saying "key delegation"; something-something federations maybe?). Because if the only use for CSFS that we can think of here is allowing a unification of BIP 118 and 119, I'd say it's more expeditious just to activate the two together as written.

In any case, great work @reardencode - there's some really useful stuff in your document, like the "what is hashed?" table. And in general, it's enjoyable to consider the novel approach you've introduced.

-------------------------

reardencode | 2023-08-27 13:37:10 UTC | #5

[quote="jamesob, post:4, topic:60"]
I agree here that there’s little practical difference between softforking to add a new opcode and softforking to enable new argument-based behavior. Clients still need to update, fork still needs to be deployed, and code savings on the Core side are pretty minor I think.
[/quote]

That's a good point. Since we have both Tapscript versions and quite a few upgradable opcodes, optimizing for avoiding using extra ops in upgrades is not a good general goal. In this case, it seems like defining an op `OP_TEMPLATEHASH` which takes a only numeric mode arg from the stack would make more sense than upgradable `OP_TXHASH`.

[quote="jamesob, post:4, topic:60"]
I think it would be really unfortunate if we got to the point where we were using CAT and rolling SHA opcodes to slice and dice txn components into tailor made hashes.
[/quote]

I couldn't agree more. We don't want to have mile-long difficult to reason about scripts. We want to have concrete verification functionality that can be composed to do useful things.

[quote="jamesob, post:4, topic:60"]
I think it would be helpful to have some kind of listing of uses that having CSFS enables (e.g. key delegation, although to be honest I can’t explain this use much deeper than just saying “key delegation”; something-something federations maybe?). Because if the only use for CSFS that we can think of here is allowing a unification of BIP 118 and 119, I’d say it’s more expeditious just to activate the two together as written.
[/quote]

Key delegation looks like this:
```
Lock: OP_2DUP OP_DROP <csfs_pubkey> OP_CSFSV OP_CHECKSIG
Unlock: <csfs_signature> <cs_pubkey> <cs_signature>
```

More usefully one might do:
```
Lock: <csfs_pubkey> OP_SWAP OP_IF OP_TOALTSTACK OP_OVER OP_FROMALTSTACK OP_CSFSV OP_ENDIF OP_CHECKSIG
Unlock0: OP_0 <csfs_signature> 
Unlock1: OP_1 <csfs_signature> <cs_pubkey> <cs_signature> 
```

I'd have to do some digging to learn about other usecases for CSFS.

--------------------------------

Thanks for your thoughts!

-------------------------

reardencode | 2023-08-27 14:02:22 UTC | #6

Summarizing some discussions I've had on X and elsewhere about this:

### Recursion, counters, Spookchains

Any covenant-capable hash/template proposal that lets the hash (or the signature on a hash) only appear in the unlock script enables some form of deleted key recursion. Deleted key constructions are obviously limited by how well the covenant creator can convince folks that they have actually deleted the key, but let's accept that if they are enabled they will likely be used.

Comparing the deleted key recursion enabled by the various possible hashes, it appears that `SIGHASH_SINGLE` type hashes enable potentially useful (or scary, depending on perspective) constructions (a la Spookchains / hashrate escrow). With `SIGHASH_ALL` type hashes, a new deleted key counter covenant would need to be created and trusted for each possible withdrawal, removing the TOFU security. In that case users would simply use federated withdrawals, skipping the whole countdown process.

I'm not sure if dropping the `SIGHASH_SINGLE` modes from this proposal (or APO by extension) would satisfy most or all of those concerned about spooky deleted key recursion.

### Helping Ark (or similar protocols)

There are clear use cases for a CTV/APO-like hash that hashes either all scripts other than the input being verified, all outpoints other than the input being verified, or both. This kind of "every other input" hashing creates quadratic hashing, so I've been hesitant to include it. @4moonsettler pointed out that for CTV-style execution this isn't a problem because there would be a hash loop making it impossible to include more than 1 such hash in a transaction. For `OP_TEMPLATEHASH`/`OP_TXHASH`-style execution, it becomes possible to create many megabytes of hashing in a consensus-valid transaction. 

One possible solution here would be to allow at most one hash-every-other mode hash in any given transaction. This of course would make reasoning about scripts more difficult - `OP_TEMPLATEHASH` might fail validation due to how it is combined with other inputs. 

Another solution would be to instead include hash-the-next modes - where the template hash commits to the next input's script/outpoint only. These would work fine for Ark, I think, but are a bit odd. https://twitter.com/brqgoo/status/1694090272009797687

-----------------------------

Responses have been quite interesting. I'm going to write down a version of `OP_TEMPLATEHASH` which does not include `SIGHASH_SINGLE`-like modes, and tries to cover as many variations on what input-related data is hashed as I can think of without introducing quadratic hashing.

-------------------------

reardencode | 2023-08-29 18:33:26 UTC | #7

Here's what I think `OP_TEMPLATEHASH` might look like, which could then be combined with `OP_CHECKSIGFROMSTACK(VERIFY)` to provide APO (but not APO|SINGLE) behavior.

-----------------

All `OP_TEMPLATEHASH` hashes commit to `<hash_mode> OP_TEMPLATEHASH` (i.e.
`0x0050`, `0x5150`, &c), the transaction's version, number of 
outputs, and outputs.

We define the following groups of input data:

* may be included for any input
    * `prevout`:
    * `prevscript`: output amount and script
    * `sequence`:
* only from the input being validated
    * `script`: control block, leaf script, and code separator position
    * `annex`: taproot spend type and annex
* other data
    * `locktime`:
    * `scriptsigs`: hash of all scriptSigs if any scriptSig is non-empty

We define the following paired hashing modes using a numeric `hash_mode` that 
can be represented by an opcode. Even modes do not hash `annex`, odd do.

 mode  | included data
 :---: | -----------------
 0,1   | -
 2,3   | locktime and this sequence (3 similar to bip118 0xc1)
 4,5   | "2" with this prevscript and script (5 similar to bip118 0x41)
 6,7   | nInputs
 8,9   | "6" with locktime and all sequences
 10,11 | "8" with scriptsigs (similar to bip119)
 12,13 | "10" with all prevscripts and script
 14,15 | "12" with all but input 0's prevout[^1]

[^1]: Modes 14,15 only work on input 0 which is a bit of an odd behavior, but
  enables a hash mode that constrains all other inputs' prevouts without 
  incurring quadratic hashing. Because such a mode could only ever be used on
  one input to a transaction, constraining it to input 0 seems a reasonable 
  solution.

-------------------------

