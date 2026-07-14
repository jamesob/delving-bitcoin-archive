# Post Quantum Recovery Of Hashed Addresses With No Confiscatory Risk

shinobi | 2026-07-14 16:55:27 UTC | #1

Currently the discussion around how to address the potential risk of a viable quantum computer centers around two primary issues: the quantum-safe signature schemes (or schemes) to integrate for users to migrate to and use if need be, and what (if anything) to do about any coins that remain secured by vulnerable ECC based scripts, specifically should these coins be frozen, and what mechanisms are available to allow legitimate owners to recover those coins if possible without an attacker having the ability to do so.

This second issue is (and always has been) a very socially contentious one, as the common understanding is it is impossible to guarantee with certainty that no user is being left in a position where they are incapable of generating a recovery proof, and therefore are forever prevented from accessing their coins.

My motivation for writing this is to solve this contention (at least partially) in a manner that can hopefully move discussions forward in a productive direction rather than lead to two opposing plans of action eventually colliding in the real world with live implementations of conflicting rules.

The current solution landscape as it stands to my understanding is:

BIP 32 hierarchical proofs, as proposed a decade or so ago by Adam Back and recently implemented as a proof-of-concept by roasbeef.
Non-deterministic stateful proofs constructed and timestamped before a deadline, showing a vulnerable ECC key signing off on a quantum-safe authentication mechanism to be used for spending after a post-quantum spending restriction is activated
Tim Ruffing/Tadge Dryja's idea (I apologize, but I forget who was the actual originator of the proposal) of a commit-reveal migration scheme where a transaction spending vulnerable ECC inputs must have an encrypted commitment to that exact transaction confirmed in a block with a pre-determined number of confirmations prior to the decrypted plaintext transaction's confirmation, or the decrypted transaction is invalid.

All three of these proposals leave some subset of coins uncovered. HD proofs are useless for users who generated their keys any other way than BIP 32, stateful timestamped proofs are useless for any inactive user or someone who for any reason does not create them before the creation deadline, and the commit-reveal migration is useless for anyone with an address type that isn't hashed because any attacker would have access to the material needed to create a valid pre-commitment for a spend.

However, by layering them, and allowing any single one of them to be used, complete coverage of any conceivable key generation method can be achieved for hashed address types. Coverage of non-hashed address types is fundamentally impossible, because the requirement to allow use of commit-reveal migration would inherently leave such address types still exposed to a quantum attacker.

Hierarchical proofs cover any BIP 32 user
Stateful timestamped proofs cover any active/observant non-BIP 32 user
Commit-reveal covers any non-BIP 32 user who is inactive/not observant

The only case in which I can see a restriction of hashed address type spends using this layered approach of recovery would leave a user unable to recover their coins is if they have lost access to their private keys. That case is completely outside of the scope of any of these proposals, and inherently impossible to cover.

-------------------------

