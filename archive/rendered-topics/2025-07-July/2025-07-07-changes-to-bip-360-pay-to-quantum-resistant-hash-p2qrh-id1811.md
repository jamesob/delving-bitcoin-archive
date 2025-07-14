# Changes to BIP-360 - Pay to Quantum Resistant Hash (P2QRH)

EthanHeilman | 2025-07-07 16:01:15 UTC | #1

We made the following changes to [BIP-360 (Pay to Quantum Resistant Hash) PR](https://github.com/bitcoin/bips/pull/1670):

* P2QRH (Pay to Quantum Resistant Hash) is now taproot (P2TR) but with the quantum vulnerable key-spend path removed.
* PQ signatures have been moved to a future BIP (coming soon).
* The plan for PQ signatures is to redefine OP_SUCCESSx opcodes: OP_CHECKMLSIG

Below we go into these changes one by one, see BIP-360 PR for full details ([BIP-360 mediawiki render as of 7/7/2025](https://github.com/bitcoin/bips/blob/77dbeee502b606fda2006ae53f4014cef612aacd/README.mediawiki)).

P2QRH is now script-spend only P2TR (taproot), i.e. no quantum vulnerable key-spend. P2QRH outputs commit directly to the tapleaf merkle root computed by taproot.

The scriptPubKey for a P2QRH output is:

OP_PUSHNUM_3 OP_PUSHBYTES_32 <tapleaf merkle root>

Advantages of this approach

1. We can reuse taproot code, but just skip taptweak steps.
2. Everyone who understands P2TR, already understands P2QRH.
3. By supporting tapscript and tapleaf, it supports everything that supports tapscript.
4. P2QRH protects tapscript outputs against long-exposure attacks. This is a big win because long-exposure attacks will be practical before short-exposure attacks. Note: protecting against short-exposure attacks requires PQ signatures.
5. P2QRH gives us similar functionality as the much discussed option of disabling key-spends in P2TR on Q-Day (when quantum attacks become practical), but with the added benefit that the ecosystem can upgrade well before Q-Day. This removes the risks of attempting a consensus change during an emergency or acting too late.

We moved PQ signatures specification out of BIP-360 so that P2QRH can be debated independently of the debate over PQ signature algorithms. This allows us to move forward on P2QRH without forcing a commitment to any particular algorithm.

BIP-360 includes a purely informational plan for adding PQ signature algorithms to tapscript. This plan to add tapscript PQ signature verification opcodes for ML-DSA (CRYSTALS-Dilithium) and SLH-DSA (SPHINCS+) via OP_SUCCESSx. This allows separate activation of PQ signature algorithms if desired and provides a pattern for adding new signature algorithms in the future. No new tapleaf version needed. The full specification will be given in a new BIP.

-------------------------

stevenroose | 2025-07-14 09:31:50 UTC | #2

If you're going the route of adding post-quantum crypto as just opcodes, then I would strongly suggest renaming the p2qrh soft-fork simply pay-to-tapscript or pay-to-mast. It's not quantum-specific and there is definitely demand from other parts of the ecosystem for something like p2ts.

-------------------------

EthanHeilman | 2025-07-14 15:05:35 UTC | #3

(post deleted by author)

-------------------------

EthanHeilman | 2025-07-14 15:09:06 UTC | #4

> there is definitely demand from other parts of the ecosystem for something like p2ts.

I agree with that BIP-360 (P2QRH) independent of its value to quantum resistance is a useful output type to have regardless.

> If you’re going the route of adding post-quantum crypto as just opcodes, then I would strongly suggest renaming the p2qrh soft-fork simply pay-to-tapscript or pay-to-mast.

Under other circumstances I would be in favor of naming this pay-to-tapscript (P2TS) or pay-to-tapleaf (P2TL), but under the threat of a post-quantum world there is a strong rationale for naming this P2QRH. In a post-quantum world, we do not want user's getting confused with P2TS (quantum safe output) and P2TR (quantum vulnerable output). P2QRH very clearly communicates that it is a quantum resistance output and distinguishes itself from P2TR.

This is also why we use Segwit version 3 rather than version 2. It makes the address contain an r,  `bc1(r)...`, so "r for resistant". This functions as a mnemonic and makes it easier for users to remember. Allowing users to, at a glance, determine if an output is Quantum-Resistant or not based it being bc1r... is helpful for avoiding accidents and letting users assess which if any of their addresses are vulnerable to long-exposure attacks. We need to help users not spend funds to to a bc1p... address if bc1p... addresses are vulnerable.

-------------------------

stevenroose | 2025-07-14 18:26:10 UTC | #5

Allow me to give some reasons of why I disagree here. Note that it's just naming though, arguably not the most important part of a technical proposal..

- The quantum threat might never materialize and a p2tr-like output type can still be useful without it. If this proposal is advertised as a solution for the quantum threat, it might not get any attention until that threat exists, which might be never.
- The name "pay-to-quantum-resistant-hash" is confusing as it breaks with the convention of output types, that describe what you're paying into. "pubkey-hash" being the hash of a pubkey, that can sign to spend, "script-hash" being the hash of a script, that should be fulfilled to spend. No "quantum resistant [thing]" is hashed here, so it sounds like you're just paying to a hash, as if spending would be done by simply providing the preimage.
- "quantum resistant hash" also sounds like you're using some better more quantum-resistant hash function, but from reading the BIP it seems like you're proposing to re-use SHA256 which in my limited QC understanding is quantum-safe? Can be seen as misleading.
- Addresses are to be read by the sender and the sender doesn't really care if the receiver's wallet is quantum-resistant. Using letters in the address to convey information also sets a bad UX precedent as they're not intended to be used that way.
- In the scenario where a quantum threat is imminent or already passed and we are in a post-quantum world, I hope we don't need address letters or output naming for developers to know what outputs they should be using.
- Also, in such a world, any additional new feature we add to bitcoin will hopefully also be quantum resistant and I hope we won't be naming every new output type as "pay to quantum resistant something". Even other new features might be added before. Will we add OP_QUANTUMRESISTANTTXHASH, OP_CHECKQUANTUMRESISTANTCONTRACTVERIFY, etc?

TL;DR: I think pay-to-tapscript is useful outside of the QC context and association with it might delay interest until QC seems more relevant.

-------------------------

EthanHeilman | 2025-07-14 19:35:52 UTC | #6

I like your objections, thanks for posting them. Let me respond to them.

>  [1] If this proposal is advertised as a solution for the quantum threat, it might not get any attention until that threat exists, which might be never.

I would not be working on it if there was not a pressing threat of quantum computers. Is it useful for other things, sure, but that is not the motivation behind this proposal.

> [2]  The name “pay-to-quantum-resistant-hash” is confusing as it breaks with the convention of output types, that describe what you’re paying into.

I don't disagree here. Technically following the naming convention it would be something like P2QRTLRH (pay-to-quantum-resistant-tapleaf-root-hash). That seems too long.

If everyone wants to activate P2QRH but they want a different name, I would not object. I do worry about the bikeshedding potential of arguing over the exact name before the PR is even merged into the BIPs repo.

> [3]  reading the BIP it seems like you’re proposing to re-use SHA256 which in my limited QC understanding is quantum-safe? Can be seen as misleading

SHA256 is quantum-resistant. If SHA256 pre-image security is broken, Bitcoin is in big trouble.

> [4] Addresses are to be read by the sender and the sender doesn’t really care if the receiver’s wallet is quantum-resistant. Using letters in the address to convey information also sets a bad UX precedent as they’re not intended to be used that way.

Senders often do care if the receiver has their funds stolen prior to receiving it. Users are also likely to examine their addresses to see if their funds are vulnerable long-exposure attacks, this makes it easier for them to do so.

Being able to determine the output type via the address is useful and we already do this with bc1... to tell someone it is a segwit (mainnet) address. I suspect this is a matter of personal taste and I'd be interested in seeing a larger discussion on this question to gauge preferences.

> [5] In the scenario where a quantum threat is imminent or already passed and we are in a post-quantum world, I hope we don’t need address letters or output naming for developers to know what outputs they should be using.

I hope the same thing, but wallets are sometimes slow to upgrade and people may be using steel wallets with the address written down. All things being equal, we should choose the option that reduces the chance of user error even if we aren't sure how big of a chance that is.

> [6] Also, in such a world, any additional new feature we add to bitcoin will hopefully also be quantum resistant and I hope we won’t be naming every new output type as “pay to quantum resistant something”. Even other new features might be added before. Will we add OP_QUANTUMRESISTANTTXHASH, OP_CHECKQUANTUMRESISTANTCONTRACTVERIFY, etc?

If we add a new quantum resistant output type after everyone has upgraded their outputs, then it doesn't really matter anymore. Everyone will have upgraded. We are past the point of danger. 

Any opcodes added via OP_SUCCESSx or tapleaf version would be compatible with P2QRH. I could be wrong, but I am not sure we will need a new output type for sometime.

What is the benefit of putting in the word QUANTUMRESISTANT in front of new opcodes as you are proposing? 

> TL;DR: I think pay-to-tapscript is useful outside of the QC context and association with it might delay interest until QC seems more relevant.

I'm curious what you want such an output for? If you don't care about quantum attacks, use P2TR with a NUMS point. Sure P2QRH would save you 32 bytes in transaction size. What is your use case here?

-------------------------

cryptoquick | 2025-07-14 20:30:49 UTC | #7

Honestly I'd be open to renaming it to Pay to Tap Script Hash, or P2TSH. It rhymes better with P2WSH that way.

I'll admit I've done worse things just to get an ACK on my BIP :joy_cat: 

Should I go ahead and make the PR?

-------------------------

