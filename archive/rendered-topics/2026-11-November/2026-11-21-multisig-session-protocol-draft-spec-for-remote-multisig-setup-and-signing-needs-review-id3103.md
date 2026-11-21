# Multisig Session Protocol: draft spec for remote multisig setup and signing needs review

onedigitmoney | 2026-11-21 17:23:33 UTC | #1

BIP-129 defines the multisig setup file, but no protocol defines how remote cosigners exchange it or PSBTs. That's usually email, chat, or a vendor's server. In that scenario the coordinator becomes the only source of the other cosigners' keys: the substitution problem BitBox described in 2020.

I've written a protocol for that exchange as a specification, so wallets can implement it without my tool (Keylay, a browser app, MIT; see https://keylay.org). The draft, explainer, test vectors with a checker, and my own defects report are here: https://github.com/KeylayBTC/msp

Cosigners share a short code out of band and use it to authenticate ephemeral X25519 keys over a relay treated as hostile. Before anything else is sent, each side reveals a nonce it committed to in its hello and the two people compare a 12-character string over both keys and both nonces, by voice or over a verified messenger; a mismatch aborts with nothing disclosed. For setup, each participant then sends its key to the other, assembles the descriptor itself by a byte-exact function, and sends the hash; any mismatch aborts, and before funding each reads the first address off its signing device and the two compare it. No one distributes a descriptor, so there is no coordinator to compromise. Signers refuse PSBTs that don't spend from a wallet both parties hold, sign SIGHASH_ALL only, and approve destinations, amounts, and fee on the device.

Limits: two clients at a time in v0.1, so setup makes two-key wallets (signing works for any wallet both parties hold, so a 2-of-3 made elsewhere is signed by two of its cosigners); everyone online; no external cryptographic review; no post-quantum protection. Keylay does not yet conform; §14 lists every difference. An external design review of the previous draft was received and closed on November 18; this is the revision that came out of it.

Two constructions I'd like reviewed (issues #1 and #3; my current answers are in `spec/review-questions.md`):

1. **The SAS commitment (§5.3, §6.8).** The October draft hashed the session keys with nothing committed, so an attacker on the path holding the code could search for a collision at about 2^30 work per side. The current draft commits to a fresh nonce in the hello (`cm = H(pk ‖ r)`, under the code MAC), reveals the nonces in a dedicated first message once both hellos are held, and hashes keys and nonces into the string; the descriptor hash is deliberately *out* of it, because anything the attacker can influence after the reveals reopens the search. I believe the bound is about 2^-59 per session with no offline work, by the commit-then-open structure of Vaudenay / Laur–Pasini / Bluetooth numeric comparison. Is the two-party argument sound in this schedule, and what must a group profile add so it holds for n parties, given that each participant has to know the expected set before it reveals?

2. **The join is not a PAKE (§5.3).** It authenticates with an HMAC under a PBKDF2-stretched code, so `channel_id` and every hello MAC are offline oracles on the code. For a generated 10-character code (about 49.5 bits) the margin is large. For typed codes it isn't, and the fixed salt lets one search cover every logged session. Proposed fix: hash only a short public rendezvous segment into `channel_id`, and use a separate secret segment solely as the CPace password. Does that eliminate offline guessing, and does the rendezvous segment leak anything?

One more I'd take pushback on: symmetric assembly costs n(n−1) key messages and a normative assembly function (§6.6a). I haven't seen it done for BSMS-style setup. Is there a reason?

Related: Blind Relay (@scarlin90) covers relay confidentiality without key agreement or descriptor binding. @Turon's transport invariants are why message identity here is deterministic; I'm replying on his thread.

-------------------------

