# Major BIP 360 Update

cryptoquick | 2025-12-19 20:46:40 UTC | #1

After reviewing community feedback, Ethan Heilman and I have enlisted the help of a third co-author, [Isabel Foxen Duke](https://x.com/isabelfoxenduke), in an editorial role to lead and execute a clean sheet rewrite of BIP 360.

Because previous revisions introduced meaningful technical changes, we determined that a full rewrite, rather than incremental edits, was warranted to improve clarity, internal coherence, and to better articulate our intentions for managing potential quantum-related risks.

Consistent with its previous version, this proposal does not introduce post-quantum signature schemes. Instead, BIP 360 proposes the addition of a new output type with the key path spend removed, which is thus protected from hypothetical breaks of Elliptic Curve Cryptography (ECC).

We have renamed this proposed output type **"Pay-to-Tapscript-Hash (P2TSH)**" for clarity, and believe its adoption is an important first step in protecting Bitcoin from potential threats to ECC, via quantum computers or any other cryptanalytic advancements.

Additionally, the proposal now includes test vectors in Python and Rust.

With gratitude, we hope you’ll review these changes in the [BIP Repo](https://github.com/bitcoin/bips/pull/1670) or at [BIP360.org](http://bip360.org). We look forward to ongoing community feedback, and new ideas in our efforts to Make Bitcoin Quantum Resistant.

Thank you for your time,
Hunter Beast

-------------------------

josh | 2025-12-20 17:45:05 UTC | #2

> We have renamed this proposed output type **"Pay-to-Tapscript-Hash (P2TSH)**" for clarity

Why did you settle on Pay-to-Tapscript-Hash? I worry this name could be confusing if a new tapleaf version is added that supports Simplicity.

-------------------------

simul | 2025-12-21 04:37:01 UTC | #3

This makes a lot of sense! if we also enabled TXHASH that would allow people to commit to a spending path that requires multi-step secret reveal. that allows spending under adversarial quantum conditions. In other words..  we don't technically need signatures at all.  This would allow people to vault their coins and spend them safely even if quantum came out tomorrow and we still hadn't gotten a signature scheme.

see: https://delvingbitcoin.org/t/a-quantum-resistance-script-only-using-op-ctv-op-txhash-and-no-new-signatures/2168

-------------------------

sipa | 2025-12-29 18:50:30 UTC | #4

Indeed, that seems like a confusing name.

I'd interpret it as a `scriptPubKey` which contains the hash of a BIP-342 tapscript, but it's (a) not the hash of a script but a Merkle root and (b) not restricted to tapscript.

I think these would be more descriptive:
* "pay to script tree Merkle root"
* "pay to taptree root"
* "pay to MAST"

-------------------------

cryptoquick | 2026-01-05 04:37:39 UTC | #5

The reason I chose Pay to Tapscript Hash was because it rhymes with Pay to Script Hash and Pay to Witness Script Hash.

I think it compares well in lists, too:

* P2SH
* P2WSH
* P2TR
* P2TSH 

I also ran it by Stephen Roose in a past thread here on Delving and he gave me the thumbs up on it.

Also, MAST is confusing because Merkelized Alternative Script Trees are different than prior MAST proposals, and the retronym makes it more difficult to disambiguate.

As for the suggestions, P2STMR, P2TTR, and P2MAST are not superior acronyms in my opinion, at least, from an aesthetic sense... And while naming is subjective and difficult and imperfect, I think P2TSH works well in the context of what’s currently in Bitcoin now.

Thank you for the feedback so far.

-------------------------

billymcbip | 2026-01-05 14:54:10 UTC | #6

How about Pay to Script Tree (P2ST)? Keep it simple.

-------------------------

cryptoquick | 2026-01-06 07:57:55 UTC | #7

I think it should end in the word Hash because it helps communicate quantum resistance. And of course a MAST merkle root is a technically just a hash like any other.

-------------------------

sipa | 2026-01-28 22:26:36 UTC | #8

I'm going to be blunt here. I realize it's petty to criticize just the name of a proposal, but I'm pretty annoyed here because I feel you're not understanding my comments.

BIP-360 as [written](https://github.com/cryptoquick/bips/blob/c7b11d7b3753bc6984c2e12b1ddf212610aa8947/bip-0360.mediawiki) in the open PR is essentially "all of BIP-341, but remove taproot". Why do you want the keep the name "tap"/"taproot" in there? It's exactly **not** taproot.

BIP-342 defines "tapscript": the specific variant of Bitcoin Script that is used in BIP-341 script tree leaves with leaf version 0xc0. My understanding is that BIP-360 intends(*) to inherit **all** semantics from the BIP-341 script tree - whether that's the BIP-342 semantics for 0xc0 leaves, or any future script semantics added separately. For that reason, I think invoking the name "tapscript" here is just **wrong**: it's not paying to "tapscript", it's paying to a tree of scripts which include any and all script types (even though tapscript is currently the only one defined). If your intent is to *only* allow paying to BIP-342 tapscript, that is not clear at all.

---

> The reason I chose Pay to Tapscript Hash was because it rhymes with Pay to Script Hash and Pay to Witness Script Hash.

Because it **rhymes**? You're seriously suggesting that's a justification for giving a misleading name?

> I also ran it by Stephen Roose in a past thread here on Delving and he gave me the thumbs up on it.

...

> Also, MAST is confusing because Merkelized Alternative Script Trees are different than prior MAST proposals, and the retronym makes it more difficult to disambiguate.

Fair enough.

> As for the suggestions, P2STMR, P2TTR, and P2MAST are not superior acronyms in my opinion, at least, from an aesthetic sense…

...

> I think it should end in the word Hash because it helps communicate quantum resistance.

Sigh, ok.

> And while naming is subjective and difficult and imperfect, I think P2TSH works well in the context of what’s currently in Bitcoin now.

I think it's highly misleading, and you're doing the ecosystem a disservice by letting aesthetics guide things.

If you insist on having "hash" at the end, at least "pay to script tree hash" wouldn't be misleading.

---

(*) I note that BIP-360 just copies the line

> Execute the script, according to the applicable script rules, using the witness stack elements excluding the script *s*, the control block *c*, and the annex *a* if present, as initial stack.

... from BIP-341, but then never defines what the applicable script rules are. Since BIP-342 specifically says it only applies when spending BIP-341 outputs, and BIP-360 outputs are not that, technically there just aren't any applicable spending rules according to the current writing, and any leaf is an anyone-can-spend. I assume that's not what you meant, but it's worth addressing too.

-------------------------

ajtowns | 2026-01-29 00:56:22 UTC | #9

[quote="sipa, post:8, topic:2170"]
If you insist on having “hash” at the end, at least “pay to script tree hash” wouldn’t be misleading.
[/quote]

"Pay to script tree hash" / p2sth seems pretty decent to me as a name for a new address class whose scriptPubKey holds the merkle root of a tree of scripts (and much better than the alternatives).

I reserve the right to pronounce "p2sth" as "pay-to-something", though :slight_smile:

-------------------------

josh | 2026-01-31 03:56:18 UTC | #10

> I reserve the right to pronounce “p2sth” as “pay-to-something”, though :slight_smile:

"Pay-to-something" is such a great name. That being said...

> I think it should end in the word Hash because it helps communicate quantum resistance.

Honestly, “hash” is redundant, if not misleading. To pay to a script tree is to pay to the tree’s root. There is no other way to pay to a tree of scripts.

On the other hand, to pay to a script tree hash implies you are hashing the root. That’s clearly not what is going on, and if it were, it would only make sense if you hashed the root with a different algorithm than what is used to construct the tree (for hypothetical security reasons).

Pay-to-Script-Tree (P2ST) would be more accurate in my opinion than P2STH. To include the word “hash” purely for marketing reasons is hardly justifiable, especially when the alternative is fewer syllables, easier to say, and technically more precise.

(Big fan of this work by the way @cryptoquick! I see no reason why semantics should get in the way of adoption. Hopefully we can reach consensus on naming and then move on to the technical merits)

-------------------------

ajtowns | 2026-02-01 03:09:37 UTC | #11

[quote="josh, post:10, topic:2170"]
Honestly, “hash” is redundant, if not misleading. To pay to a script tree is to pay to the tree’s root. There is no other way to pay to a tree of scripts.
[/quote]

If you didn't do hashing at all, you could pay to a bare scriptPubKey of all the leafs in your tree, separating them by "IF/ELSE" in some form. Taproot's merkle root is fundamentally just a hash generated from a particular priority ordering of a set of leaves.

-------------------------

cryptoquick | 2026-02-04 16:22:48 UTC | #12

I appreciate your concerns and your honesty, Pieter, as I have always appreciated your generosity when I’ve asked you questions on certain things in the past on libsecp256k1 and also on Bitcoin StackExchange.

I have consulted with my co-authors and we have come up with two names we would be happy with:

* P2MR - Pay-to-Merkle-Root
* P2QB - Pay-to-QuBit

Curious to hear your thoughts. I would also like to give the community a week to also chime in. As this will be the third name for the BIP, it might help to allow others to contribute their thoughts as well. Ultimately, the name doesn’t matter as much as in the grander scheme of things. What matters is that we can lay the foundation for spend script optionality, which was in part pioneered with your work, and for that we owe you a great deal.

One thing I would also like to clarify; is the only thing you dislike of the BIP just the name and the lack of clarity in spending rules? Happy to address all concerns.

-------------------------

alex | 2026-02-06 03:48:07 UTC | #13

Another name to toss in: **P2TR-** (Pay-to-Taproot-minus)

* Self-descriptive: it immediately tells you what the output type *is* - taproot with the key path removed.

* Sets correct expectations: the minus signals that it can do *most* of what P2TR can do, but not all of it.

* Zero learning curve for anyone already familiar with taproot.

-------------------------

sipa | 2026-02-06 20:06:28 UTC | #14

[quote="cryptoquick, post:12, topic:2170"]
I have consulted with my co-authors and we have come up with two names we would be happy with:
[/quote]

Of those two, I think P2MR is the more accurate one. P2QB may relate to the BIP's intent, but I don't think it describes the actual behavior, as BIP-360 (as written in the PR) doesn't outlaw EC opcodes inside.

[quote="cryptoquick, post:12, topic:2170"]
One thing I would also like to clarify; is the only thing you dislike of the BIP just the name and the lack of clarity in spending rules?
[/quote]

Those were the comments I had, as I think clarity is important for the ecosystem to help judge proposals. Neither is intended as an argument either for or against adoption of the BIP itself, which I'm not going to comment on.

-------------------------

billymcbip | 2026-02-06 22:21:14 UTC | #15

> Why do you want the keep the name “tap”/“taproot” in there? It’s exactly **not** taproot.

To be fair, BIP360 still uses the "TapLeaf" and "TapBranch" tags so including "tap" in the name isn't unreasonable. That said, I still prefer Pay-to-Script-Tree.

-------------------------

cryptoquick | 2026-02-09 21:15:06 UTC | #16

The problem with calling it Script Tree is that locks us into an implementation detail that might become more ambiguous someday if things like Simplicity do catch on.

But if sipa wills it, it is done. You have to understand, I have it on good authority that sipa is the sole S-tier influencer in the bitcoin governance process, and so if he wants us to change the name, we change the name!

-------------------------

sipa | 2026-02-09 21:26:03 UTC | #17

[quote="cryptoquick, post:16, topic:2170"]
The problem with calling it Script Tree is that locks us into an implementation detail that might become more ambiguous someday if things like Simplicity do catch on.
[/quote]

I don't understand this. If Simplicity were added as a new leaf version, there would still be a script tree above it (unless you imagine it as an independent output type, which isn't unreasonable, but then it's unrelated to BIP-360 or BIP-341 anyway).

[quote="cryptoquick, post:16, topic:2170"]
You have to understand, I have it on good authority that sipa is the sole S-tier influencer in the bitcoin governance process, and so if he wants us to change the name, we change the name!
[/quote]

And you have to understand that my goal is explicitly to not have this kind of influence, which is why I'm not commenting on whether the consensus change itself is desirable. But you also shouldn't change a name because I don't like it - you should pick the name that you believe is most clear to everyone, after weighing all comments about it (including mine, if it convinces you).

-------------------------

leishman | 2026-02-12 03:25:31 UTC | #18

> In other words, we believe users' fear of quantum computers may be worth addressing regardless of CRQC viability. Given these concerns, we think it's worth considering simple low risk changes that create options for using Bitcoin in a quantum-resistant way.

It’s possible that the convo has been had elsewhere but it’s unclear to me how this proposal does anything to actually address QC concerns. Nothing proposed here improves the quantum security posture for Bitcoin, it only “fixes” a potential weakness in taproot.

Specifically, I think the BIP fails to answer the question: Why go through the effort of a soft fork to make a change that won’t actually make bitcoin any more QC resistant? Why not instead focus on adding a QC secure signature op code?

-------------------------

ArmchairCryptologist | 2026-02-13 08:38:14 UTC | #19

[quote="leishman, post:18, topic:2170"]
It’s possible that the convo has been had elsewhere but it’s unclear to me how this proposal does anything to actually address QC concerns. Nothing proposed here improves the quantum security posture for Bitcoin, it only “fixes” a potential weakness in taproot.

[/quote]

It specifically fixes the low-hanging vulnerability where P2TR implicitly exposes a public key through the key-path spend, which makes it vulnerable to CRQCs due to long-term public key exposure. So it does make Bitcoin more quantum-resistant, for taproot specifically, which may be an important stop-gap while the addition of PQC signatures is worked out. Though it should probably be emphasized that the new address format is not *more* quantum-resistant than the other modern address types, and is still vulnerable to short-exposure attacks, should they become feasible.

-------------------------

leishman | 2026-02-13 18:20:41 UTC | #20

> It specifically fixes the low-hanging vulnerability where P2TR implicitly exposes a public key through the key-path spend

Yes I think the BIP does a good job at making that clear, but we already have a fix that doesn’t require any changes: You can just use P2WSH and get the same security guarantees with no change to the protocol. So this proposed change is specifically adding more complexity to solve the problem of “I want to use tapscript but am worried about quantum” and I personally don’t see why this is the problem to be focusing on (and it’s very possible I’m wrong).

The BIP website says this is “**a proposed first step in advancing Bitcoin quantum resistance”,** but the BIP provides no argument for why THIS, out of a multitude of alternatives, should be the first step in advancing Bitcoin’s quantum resistance. Is there an assumption that all future quantum resistant work (like adding hash-based signatures) will be done under this P2TSH umbrella?

My specific feedback for the author is: make the case clearer for why this specific change should be the first step we take towards quantum security.

-------------------------

murch | 2026-02-13 18:40:03 UTC | #21

My understanding is that the authors of BIP 360 anticipate that a post-quantum signature scheme will be added to Tapscript by redefining an OP_SUCCESS opcode next to the introduction of P2MR.

-------------------------

cmp_ancp | 2026-02-13 23:02:28 UTC | #22

I hardly think anyone would push to activate this BIP alone, but rather in a post quantum bundle. It's just matter of separation of concerns, we use a BIP to define the structure beneath and another defining the signature schemes per se.

-------------------------

