# Is there a social approach to the quantum problem?

josh | 2025-03-24 23:47:15 UTC | #1

I saw [this tweet](https://x.com/lynaldencontact/status/1904262504638783832?s=46&t=5QZh3FLO-zdzeZNNX9xX0w) suggesting “quantum” was the greatest threat to Bitcoin, from an investment risk perspective.

While Bitcoin generates no cash flows, it is valued like an investment, whether we like it or not. And investments are discounted by the perceived riskiness of the investment.

Therefore, it is reasonable to conclude that market participants withhold capital allocation to Bitcoin on the basis of this perceived risk. This depresses Bitcoin’s price, not in USD terms but in terms of its real purchasing power.

Now, I am not someone who believes quantum is a threat to Bitcoin. I understand that hashed public keys are quantum safe and that Bitcoin is a social experiment. Not a technical one.

But do others? I think not. Most approach “Bitcoin” no different from “blockchain.” All they see is the tech.

So I ask, should the community present a social solution, to the quantum problem?

Perhaps there is no immediate need for a quantum soft fork. People don’t need that. What they need is certainty their coins will be safe. If the worst happens and a breakthrough out of left field happens tomorrow.

How might the community proceed, if that happened?

One way would be to give up, and start again from a quantum-resistant beginning.

But why throw away the existing ledger? Why not publicly release a transition plan, that any user can opt into?

This brings me to my point. Why not start a movement around signing quantum-resistant public keys?

Why not ask users to provide proof of ownership?

No one has to, and there is no enforcement mechanism, but if there existed a large repository of timestamped public key signatures, using something like [OpenTimestamps](https://opentimestamps.org) and [BIP322](https://github.com/bitcoin/bips/blob/02ad0e01c2a9189124e05a52afe97ef90a3b7f1f/bip-0322.mediawiki) proofs of ownership, it would be trivial to reconstruct the ledger, if the worst happened.*

Is this needed? Technically, probably not. Owners of UTXOs that do not expose the public key can probably safely move to a quantum-proof UTXO if they need to, by sending their coin through a private miner who has a legal obligation to not try to steal their coin.

But the point isn’t to fix the quantum problem. It’s to reassure users, from the smallest village to the biggest bank, that Bitcoin is safe to seriously adopt.

Has anyone pursued this approach before? Would there be interest / support from the community?

*the proofs themselves wouldn’t even need to be published, unless a quantum breakthrough occurs

-------------------------

garlonicon | 2025-03-25 06:58:42 UTC | #2

> using something like OpenTimestamps

They put their proofs inside OP_RETURN. New protocols should probably commit things directly into ECDSA signature instead, by tweaking R-value of the signature. Then, transaction size will be smaller, and it will be harder to detect, if someone attached any commitment or not.

-------------------------

josh | 2025-03-25 17:24:08 UTC | #3

Fascinating!

You got me thinking. There's probably a better approach. At least for taproot addresses, users could conceivably tweak their Schnorr public key with the hash of some quantum-resistant public key.

This would minimize the on-chain footprint and the off-chain backup requirements, while providing a seamless way to backup the ledger.

-------------------------

garlonicon | 2025-03-26 06:46:51 UTC | #4

> At least for taproot addresses, users could conceivably tweak their Schnorr public key with the hash of some quantum-resistant public key.

Tweaking keys works on all address types, which use OP_CHECKSIG in that way or another. You can even tweak some DER signature for P2PK, if you really want. Later, when the private key will be compromised, people could always make new signatures, but couldn't go back in the chain, and change previous commitments.

And I think the main problem is not related to technically doing it, but rather to standardizing it correctly, so when the community will want to upgrade the rules, the new code will handle all commitments correctly. Because I think it is quite likely, that there will be more than one way to commit to things, and different people may use different schemes.

-------------------------

josh | 2025-03-27 02:49:13 UTC | #5

> Tweaking keys works on all address types, which use OP_CHECKSIG in that way or another. You can even tweak some DER signature for P2PK, if you really want. Later, when the private key will be compromised, people could always make new signatures, but couldn’t go back in the chain, and change previous commitments.

I see. Just to clarify, you're talking about tweaking signatures in the input(s), correct? Not data embedded in the UTXO itself.

My understanding is that a P2TR UTXO is the only type of UTXO where a commitment can be made *in the output script*. It sounds like you're suggesting users instead commit to quantum-resistant public keys by tweaking signatures in the inputs, for backing up outputs created in the same transaction, or outputs that exist elsewhere.

That's an interesting idea. It could work. It doesn't matter where the commitment exists, as long as the earliest provable commitment is used as a UTXO's proof of ownership.

> I think the main problem is not related to technically doing it, but rather to standardizing it correctly, so when the community will want to upgrade the rules, the new code will handle all commitments correctly.

I agree 100%. This is a social problem, not a technical one. If there existed a "standard" way to commit to a quantum-resistant backup public key, for any existing or newly created UTXO, users and investors would probably feel more comfortable with the "quantum risk," since a fallback plan exists.

Whether we like it or not, perception matters. If Bitcoin is perceived as less risky, its purchasing power in real terms will naturally rise.

-------------------------

garlonicon | 2025-03-27 05:19:09 UTC | #6

> Just to clarify, you’re talking about tweaking signatures in the input(s), correct?

You can tweak things in two places: in your output, or in your input. I think it is better to tweak inputs, because changing your output will change your address, which may confuse some users. And it is harder to handle commitments in outputs, because you have to always carry your commitment in your output descriptor, to properly derive your address.

> My understanding is that a P2TR UTXO is the only type of UTXO where a commitment can be made in the output script.

No, you can commit data to any public key, no matter where it is located, and in which address type it is wrapped. No matter if you pick Taproot or not, you should not push your committed data on-chain now, because it is not enforced by the current consensus (just like you should not push witness data on-chain before 2017).

> for backing up outputs created in the same transaction, or outputs that exist elsewhere.

First, the whole way of committing to any data should be standardized. And then, we can decide, how to interpret committed data, and how they should be revealed (and should they be stripped before sending to non-upgraded nodes, like we handle witness data).

Today, if you encounter OP_CHECKSIG anywhere, you can see any valid signature, and then it is accepted. In the new consensus, when it will be possible to go from public to private key, not all signatures will be accepted. You will need a regular signature, as it is today, and some commitment to prove, that you are the true owner, and not someone, who only recovered the private key.

Which means, that after a future soft-fork, you will be able to move P2PK in the same way as today, but additional commitment will be required, and attached in a similar way, like witness is attached today. And then, non-upgraded nodes will just see everything in the current format, but upgraded ones will accept a new format, and a valid commitment data for each OP_CHECKSIG call.

Which also means, that old address types would not be "burned", like "removed from the UTXO set". Instead, they would be just "trapped", so you would need more conditions to move them (signature + commitment). And decades later, commitments may also become unsafe, so we will upgrade into (signature + commitment + something). And so on, and so forth.

And I guess hash functions can be upgraded in a similar way: if SHA-256 will ever be broken, then the new hash function should produce identical hashes for all old data, but new 256-bit values for some broken edge cases, like it is done today with hardened SHA-1.

-------------------------

