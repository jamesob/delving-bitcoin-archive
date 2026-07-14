# What does "post-quantum" actually mean for a Bitcoin L2 when settlement still happens on a non-PQ L1?

Hal_03 | 2026-07-14 19:01:51 UTC | #1

I'm working on a Post-Quantum State-Channel payment method on Bitcoin, and building it raised a question I think matters beyond my specific case.

You can make every off-chain component of a system post-quantum, signatures, identity, ledger state, proofs. But if settlement ultimately touches Bitcoin L1, you're still relying on ECDSA/Schnorr over secp256k1. So what does "post-quantum" actually mean here? I don't think it's about which algorithms run inside the system, I think it's about the boundary: what specifically crosses from the off-chain side to L1, and whether that crossing point can be minimized rather than just accepted.

This isn't specific to my project. Any BitVM-based bridge, Lightning, anything that does real cryptographic work off-chain and settles on a non-PQ L1, the same question applies, whether or not it's being asked yet.

**Where I've gotten so far**

Peg-out seems more tractable, it can potentially be reduced to a commitment-based claim, with the classical signature only appearing once, at the final moment of withdrawal, rather than being exposed continuously. Peg-in seems fundamentally harder, depositing requires a classical signature to construct the transaction itself, and I don't currently see a way around that without changing Bitcoin L1's own signature scheme.

Genuinely interested in whether this framing holds up, whether others have thought about this systematically, or whether I'm missing existing work on it.

-------------------------

