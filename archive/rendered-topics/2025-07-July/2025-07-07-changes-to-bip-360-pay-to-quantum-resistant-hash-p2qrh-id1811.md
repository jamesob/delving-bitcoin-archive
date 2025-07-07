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

