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

