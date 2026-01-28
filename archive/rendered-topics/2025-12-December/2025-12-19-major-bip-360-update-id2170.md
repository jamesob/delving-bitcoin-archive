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

sipa | 2026-01-28 20:12:17 UTC | #8

I'm going to be blunt here. I realize it's petty to criticize just the name of a proposal, but I'm pretty annoyed here because I feel you're not understanding my comments.

BIP-360 as [written](https://github.com/cryptoquick/bips/blob/c7b11d7b3753bc6984c2e12b1ddf212610aa8947/bip-0360.mediawiki) in the open PR is essentially "all of BIP-341, but remove taproot". Why do you want the keep the name "tap"/"taproot" in there? It's exactly **not** taproot.

BIP-342 defines "tapscript": the specific variant of Bitcoin Script that is used in BIP-341 script tree leaves with leaf version 0xc0. My understanding is that BIP-360 intends(*) to inherit **all** semantics from the BIP-341 script tree - whether that's the BIP-342 semantics for 0xc0 leaves, or any future script semantics added separately. For that reason, I think invoking the name "tapscript" here is just **wrong**: it's not paying to "tapscript", it's paying to a tree of scripts which include any and all script types (even though tapscript is currently the only one defined). If your intent is to *only* allow paying to BIP-342 tapscript, that is not clear at all.

(*) I note that BIP-360 just copies the line

> Execute the script, according to the applicable script rules, using the witness stack elements excluding the script *s*, the control block *c*, and the annex *a* if present, as initial stack.

From BIP-341, but then never defines what the applicable script rules are. Since BIP-342 specifically only says it applies to BIP-341 output spending rules, and BIP-360 outputs are not that, technically there just aren't any applicable spending rules according to the current writing, and any leaf is an anyone-can-spend?

---

> The reason I chose Pay to Tapscript Hash was because it rhymes with Pay to Script Hash and Pay to Witness Script Hash.

Because it **rhymes**? You're seriously suggesting that's a justification for giving a misleading name?

> I also ran it by Stephen Roose in a past thread here on Delving and he gave me the thumbs up on it.

...

> Also, MAST is confusing because Merkelized Alternative Script Trees are different than prior MAST proposals, and the retronym makes it more difficult to disambiguate.

Fair enough.

> As for the suggestions, P2STMR, P2TTR, and P2MAST are not superior acronyms in my opinion, at least, from an aesthetic sense…

...

> And while naming is subjective and difficult and imperfect, I think P2TSH works well in the context of what’s currently in Bitcoin now.

I think it's highly misleading, and you're doing the ecosystem a disservice by letting aesthetics guide things.

-------------------------

