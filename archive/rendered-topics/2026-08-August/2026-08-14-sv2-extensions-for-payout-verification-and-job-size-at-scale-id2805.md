# SV2 Extensions for Payout Verification and Job Size at Scale

SweetHash | 2026-08-14 06:45:51 UTC | #1

Miner-side job declaration, where the miner builds their own block template, is standardized in Stratum V2 and has real mainnet precedent in DATUM. That closes one half of a non-custodial pool's problem: the miner already doesn't have to trust the pool's choice of transactions.

It doesn't close the other half. Stock SV2 gives a pool no way to make its payout split checkable before a block is found, and no way to keep a mining device's job the same size once payout structure grows past a handful of addresses. Both are still just "trust the pool," in two different places.

I've been working on Tessera, a suite of two Stratum V2 extensions aimed at that gap. Tessera is not a new protocol. It's two extensions, Pactum and Axiom, that plug into the Job Declaration hop SV2 already defines, negotiated through SV2's own extension-negotiation exchange (\`RequestExtensions\` / \`RequestExtensionsSuccess\`), the same mechanism any other SV2 extension would use.

**Pactum (0x4E43)**

What it is: a single message, \`SetPayoutCommitment\`. A pool signs a specific division of a future block's payout outputs, using BIP340 Schnorr over secp256k1, and binds that signature to the exact \`mining_job_token\` issued for that Job Declaration session.

The problem: in stock SV2, a miner has no way to check, before a block is actually found, that the pool will pay out the split it claims. The payout rule lives entirely in the pool's own database, unobserved from outside and unenforceable by anyone but the pool.

How it closes that: the commitment turns the payout rule into a signed, checkable artifact instead of an internal promise. Once a pool issues it for a session, it can't quietly change that session's split. A different split fails to verify against the signature already on record. Checking it needs the signature and the pool's public key, nothing else.

**Axiom (0x4E50)**

What it is: coinbase midstate compression. A 32-byte hash, computed once upstream of the mining device, standing in for the full structure of a coinbase transaction's payout outputs.

The problem: a pool paying many addresses produces a coinbase whose size grows with the number of outputs. Forwarding that whole structure down to resource-constrained ASIC firmware for it to verify or reconstruct doesn't scale with payout complexity. More contributors means a bigger job, on hardware that wasn't built to get bigger.

How it closes that: the midstate is computed once, upstream. The device's job stays a constant 32 bytes no matter how many payout outputs sit behind it, and resumes hashing from that midstate rather than re-deriving the coinbase structure itself.

**Where this actually stands**

I want to be precise about this rather than round it up. What exists today is a working prototype Job Declaration Server: real SV2 Noise handshake, real extension-negotiation flow, \`SetPayoutCommitment\` issued only after negotiation succeeds, not before.

I tested it on the wire against the real reference Stratum V2 Job Declaration Client (the SRI project's own JDC). The result: the reference JDC tolerates the Pactum extension without erroring, but doesn't act on it, because its own Job Declaration code has no extension-negotiation logic today. That's a real, measured finding, not a guess, and it's as much a statement about the reference implementation's current state as it is about this extension. If that's already a known gap, or already being worked on, I'd genuinely like to know.

What isn't true yet: the repository isn't public. Nothing here runs inside a production pool, including the one my own team operates. Both of those are things I want to fix, not things I'm claiming already happened.

**What I'm actually asking**

Does gating \`SetPayoutCommitment\` behind explicit extension negotiation seem like the right shape for this kind of addition to Job Declaration, or is there a cleaner way to express "the pool commits to a payout function, not just a set of outputs" inside SV2 as it stands today? And separately: is the reference JDC's lack of extension-negotiation support in Job Declaration a known limitation, or worth its own issue against the reference implementation?

I'll follow up here once the repository is public, with the actual code and the full interop result instead of a description of it.

DATUM is the real mainnet precedent for miner-side template construction, and the reason the other half of this problem was worth taking seriously. Tessera is an attempt at the pool-side half of the same idea, open and not tied to one pool's implementation.

-------------------------

SweetHash | 2026-08-14 23:58:38 UTC | #2

Here is the repo: [sweethashio/Tessera: Tessera SV2 Suite](https://github.com/sweethashio/Tessera)

-------------------------

