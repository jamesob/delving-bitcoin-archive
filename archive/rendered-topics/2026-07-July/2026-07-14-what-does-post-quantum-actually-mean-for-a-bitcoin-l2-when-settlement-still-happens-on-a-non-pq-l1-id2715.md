# What does "post-quantum" actually mean for a Bitcoin L2 when settlement still happens on a non-PQ L1?

Hal_03 | 2026-07-14 19:01:51 UTC | #1

I'm working on a Post-Quantum State-Channel payment method on Bitcoin, and building it raised a question I think matters beyond my specific case.

You can make every off-chain component of a system post-quantum, signatures, identity, ledger state, proofs. But if settlement ultimately touches Bitcoin L1, you're still relying on ECDSA/Schnorr over secp256k1. So what does "post-quantum" actually mean here? I don't think it's about which algorithms run inside the system, I think it's about the boundary: what specifically crosses from the off-chain side to L1, and whether that crossing point can be minimized rather than just accepted.

This isn't specific to my project. Any BitVM-based bridge, Lightning, anything that does real cryptographic work off-chain and settles on a non-PQ L1, the same question applies, whether or not it's being asked yet.

**Where I've gotten so far**

Peg-out seems more tractable, it can potentially be reduced to a commitment-based claim, with the classical signature only appearing once, at the final moment of withdrawal, rather than being exposed continuously. Peg-in seems fundamentally harder, depositing requires a classical signature to construct the transaction itself, and I don't currently see a way around that without changing Bitcoin L1's own signature scheme.

Genuinely interested in whether this framing holds up, whether others have thought about this systematically, or whether I'm missing existing work on it.

-------------------------

ZmnSCPxj | 2026-09-30 12:23:49 UTC | #2

It seems to me that systematically, any non-PQ L1 simply cannot host a PQ L2 that is worthy of the name "L2".

If any entity can derive L1 private keys from L1 public keys that they semantically "should not" control, then they can create an L1 transaction that anchors (pegs in, whatever) any funds controlled by that key into any L2, and then the current entity who "should have" controlled that key would be unable to prevent the loss.  It is thus immaterial whether the L2 uses PQ for the purposes of fund security.

-------------------------

Hal_03 | 2026-09-30 13:51:16 UTC | #3

Thanks for the reply; after three months, I had given up hope of hearing back from anyone :rofl:

In principle I agree with you, but on the point that a PQ L2 is useless, no. Actually, the more I think about it, the more the argument turns against itself.

Because the L2 doesn't add the vulnerability, it inherits it. If a quantum computer can derive the private key from an L1 public key, it can steal those funds directly on-chain, with or without an L2. A peg-in with a classical signature isn't a flaw of the layer, it's the cost of spending a UTXO on Bitcoin, full stop. So an L2 like this is as secure as the L1 against the quantum threat: if your reasoning made the L2 useless, it would make Bitcoin itself useless too. Immaterial for fund security, as you say, but fund security isn't the only thing a payment system has to offer.

Then there's the role. If the L2 were created to replace the L1, then yes, it would be useless, and on that I agree with you. But the role of a layer like this is to make Bitcoin easier to use for payments (the reason Bitcoin was created), and there the post-quantum properties matter a lot. Privacy, for example: the L1 doesn't offer it at all. An adversary who intercepts and stores encrypted traffic today will decrypt it tomorrow, once a quantum computer exists. PQ signatures and encryption off-chain protect today's payments from tomorrow's attack, and that value doesn't depend in any way on whether the L1 is PQ.

On custody you're right, and in my case even more than you say: a custodial channel concentrates many users' funds under a few keys, a bigger target than average. That's exactly why the peg-out is designed as a commitment-based claim, with the classical signature reduced to a single moment, the final one. Minimizing the number and the window of exposure is the only engineering answer possible as long as the L1 stays as it is. It doesn't make it perfect, it makes it as little exposed as possible.

About preparing. When the L1 adopts something like P2QRH (BIP-360), QuBit or Lamport signatures in Script, an off-chain layer that is already PQ can migrate the peg the same day without redesigning anything and without unlocking user's founds. Today's work is the prerequisite for tomorrow's migration.

In the end, the point of my original post was exactly this: what does "post-quantum" actually mean for an L2 when settlement happens on a non-PQ L1? I don't think the answer is definitive, it depends on the phase we're in. Right now Bitcoin's L1 is not yet post-quantum, so a post-quantum L2 has the function, beyond the one it was created for, of reducing as much as possible the sensitive information that can be collected today and decrypted tomorrow. If one day Bitcoin itself becomes PQ resistant, then a PQ L2 will take on a different definition. I don't claim it's already perfect, but the fact that it already exists means someone is thinking and acting to face this challenge, instead of doing everything at the last second.

What do you think? Have you worked hands-on on anything in this space, or is this a concern you've looked at from the outside?

-------------------------

ZmnSCPxj | 2026-09-30 15:23:34 UTC | #4

[quote="Hal_03, post:3, topic:2715"]
So an L2 like this is as secure as the L1 against the quantum threat: if your reasoning made the L2 useless, it would make Bitcoin itself useless too. Immaterial for fund security, as you say, but fund security isn’t the only thing a payment system has to offer.
[/quote]

I will stop you right here and point out that fund security is an absolute requirement for this system, otherwise you might as well just go back to the legacy financial system.  A payment system that cannot offer fund security is no different from the legacy financial system and any other features it has is irrelevant.

-------------------------

