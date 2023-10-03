# CATT: thoughts about an alternative covenant softfork proposal

stevenroose | 2023-10-02 00:40:20 UTC | #1

Since I just discovered this forum and some interesting conversations are going on here. I definitely saw James' proposal [here](https://delvingbitcoin.org/t/covenant-tools-softfork/98/5), not trying to undermine that, just trying to give an alternative perspective.

I've been thinking about this idea, which I dub "covenant all the things", which I guess could go a few ways. The idea is to not focus on specific use-cases (APO, VAULT), but provide more general tools that 

A very minimal version would be a combination of 

- OP_TXHASH
  - and OP_CHECKTXHASHVERIFY for templating
- OP_CHECKSIGFROMSTACK
  - together with TXHASH this is very powerful
  - easily emulates APO to build eltoo-style protocols

These two are already very powerful together. It basically introduces a "sighash on steroids" that allows signing off on anything you want. Plus a very flexible templating mechanism that supports bring-your-own-fee constructions.

So, on top of that, to allow real introspection, there are some possibilities:

- the full version would be
  - OP_CAT
    - who doesn't like CATs?
  - OP_TX
    - for full introspection, this works quite well with OP_TXHASH's TxFieldSelector
  - 64-bit arithmetic
    - for anything with amounts
  - OP_TWEAKADD
    - for doing taproot stuff

- a lesser version
  - OP_HASHMULTI (aka OP_CATSHA256) or something
  - OP_TWEAKADD (this seems kinda needed in any case)

One of the nice properties of all of this is that almost all of these opcodes already have production implementations in Elements (Liquid), except for the OP_TXHASH and OP_TX which are closely related. I formalized a potential OP_TXHASH BIP [here](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-September/021975.html) (discussion here on Delving [here](https://delvingbitcoin.org/t/draft-bip-for-op-txhash-and-op-checktxhashverify/121/1)), and OP_TX could be implemented by re-using the TxFieldSelector specified in that BIP.

-------------------------

ajtowns | 2023-10-02 09:32:28 UTC | #2

You're certainly welcome to cross-port OP_CAT and OP_CHECKSIGFROMSTACK to [inquisition](https://github.com/bitcoin-inquisition/bitcoin/pulls); I'm skeptical that it'll be reasonably feasible to do anything useful with them in script (as opposed to handwaving on a blog / mailing list), but I'd love to be proved wrong. Could specify the behaviour on the wiki (eg https://github.com/bitcoin-inquisition/bitcoin/wiki/Relay-Policies) if you didn't want to write up a proper BIP yet.

-------------------------

stevenroose | 2023-10-02 10:33:05 UTC | #3

[quote="ajtowns, post:2, topic:125, full:true"]
You're certainly welcome to cross-port OP_CAT and OP_CHECKSIGFROMSTACK to [inquisition](https://github.com/bitcoin-inquisition/bitcoin/pulls); I'm skeptical that it'll be reasonably feasible to do anything useful with them in script (as opposed to handwaving on a blog / mailing list), but I'd love to be proved wrong. Could specify the behaviour on the wiki (eg https://github.com/bitcoin-inquisition/bitcoin/wiki/Relay-Policies) if you didn't want to write up a proper BIP yet.
[/quote]

CHECKSIGFROMSTACK only really makes sense with something like TXHASH so that you have something to sign, I guess. Building a covenant with just CAT+CSFS is indeed not reasonably feasible, I agree with that. Though look at [what the BitMatrix guys have built](https://beta.bitmatrix.app/), it's pretty impressive, though obv it requires arithmetic too.

-------------------------

reardencode | 2023-10-02 16:47:30 UTC | #4

> CHECKSIGFROMSTACK only really makes sense with something like TXHASH so that you have something to sign, I guess. Building a covenant with just CAT+CSFS is indeed not reasonably feasible, I agree with that. Though look at [what the BitMatrix guys have built ](https://beta.bitmatrix.app/), it’s pretty impressive, though obv it requires arithmetic too.

I mentioned this over on Telegram, but think it's worth saying here as well: I think it makes more sense to take something like TXHASH or [Template Key](https://github.com/reardencode/bips/blob/ccda3646d82fe589103c472d5c2e4e0627c85cd7/bip-template-key.mediawiki) and make an APO-ish new Tapscript key version with a variant that works well with signatures rather than adding CSFS. _If_ we get to the point that we're adding full introspection then that changes, but until then there are differences in how we should hash things between equality/script usage and signature usage.

-------------------------

stevenroose | 2023-10-02 17:02:27 UTC | #5

Why exactly do you think that?

I see the benefits of introducing a key version (simple keyspends mostly), but I don't see the drawbacks of CSFS as an alternative. A few more bytes in the witness?

What I personally like about TXHASH is that it gives both templating and a sighash with just CSFS extra. Defining a key version seems more work than adding the opcode.

There might actually be semantic differences. With TXHASH+CSFS you can specify exactly what message each key has to sign. With sighashtypes (AFAIU), the sighashtype is added to the signature, so it can't be enforced by the script but is provided at sign-time by the signer.

I have to confess I didn't do much thinking about these differences and which semantics are more desirable.

-------------------------

reardencode | 2023-10-02 17:21:19 UTC | #6

> I see the benefits of introducing a key version (simple keyspends mostly)

Sadly, APO (and other such ideas for new Tapscript key versions) don't enable key path spends because that would split the anonymity set of Taproot outputs :cry:

> There might actually be semantic differences. With TXHASH+CSFS you can specify exactly what message each key has to sign. With sighashtypes (AFAIU), the sighashtype is added to the signature, so it can’t be enforced by the script but is provided at sign-time by the signer.

Exactly, it is semantically different. Which is more likely when a hash is going to be used with a signature: The signer may specify at signing time what parts of the transaction they want to hash and is the party best suited to make the decision on what hashing modes to use; or the creator of the output script knows exactly the single hashing mode appropriate for the signer to use at script creation time. Until we are enabling ~full introspection, I would argue that the former is much more useful and matches the expected semantics of bitcoin script better.

-------------------------

stevenroose | 2023-10-02 23:57:02 UTC | #7

But with TXHASH and CSFS, both options are possible, while with a sighashtype only the at-sign-time option.. To emulate a sighashtype, the user would pass a TxFieldSelector (or whatever you call the input to TXHASH) and a signature and then the script would generate the hash value using the user's input and then validate the signature.

One could actually argue that this is a more pure implementation of OP_CHECKSIG in Bitcoin Script..

-------------------------

reardencode | 2023-10-03 04:08:50 UTC | #8

Sure, you can get the same result with TXHASH + CSFS, but now you have hash modes (tagging with the selected mode, annex hashing) that are _only_ used with CSFS where those could be omitted if the same general hash structure was available separately in the separate contexts. (If you haven't already, reading [Template Key](https://github.com/reardencode/bips/blob/ccda3646d82fe589103c472d5c2e4e0627c85cd7/bip-template-key.mediawiki) might make this more clear)

-------------------------

stevenroose | 2023-10-03 09:49:04 UTC | #9

Hmm, yeah so I think I read the part where you specify the template hashtypes for CTV. So this document adds them as sighash types basically.

In theory we could do the exact same thing with TXHASH: specify a key type and use the TxFieldSelector construction as a "sighash flags on steroids" where you can use that to select parts of the tx to hash.

Your proposal a huge difference in flexibility with TXHASH, though. For example the special treatment of only the first input is very limiting in many ways.

I think I'm at the point where all resource usage concerns can be elevated from TXHASH so I don't see why we would need a system that pre-sets different modes of operation instead of allowing users to set their own. Of course the result is very likely that only a handful of TxFieldSelector values will take up 99% of usage, but it does allow protocol developers to construct more complicated systems.

Actually this makes me think twice about the default for TXHASH. Currently the default is "all that's non-recursive", so CTV-style. But in fact for a sighash, the regular "ALL" makes more sense.

We do have one special case value we can assign: the 0x00 byte, because "not including anything" is currently not allowed. So we could play with two different "shortcut values": `0x` and `0x00`.

-------------------------

