# PSBT/MuSig2 coordination over Nostr relays: transport invariants for nonce safety under at-least-once delivery

rafaelturon | 2026-09-01 10:04:52 UTC | #1

Most multi-party custody arrangements (2-of-3 multisig, MuSig2 Taproot aggregates, covenant-free time-locked vaults) still move their PSBTs and signing rounds through a coordinator run by the wallet vendor. The on-chain construction distributes authority; the coordination path re-concentrates observation and liveness in one operator. Joinstr, Munstr, Smart Vaults and Nunchuk have each shown that Nostr relays can carry parts of this traffic. What the lineage has not produced is a treatment of the transport itself: its invariants, its threat model, its failure modes at serious stakes.

We published a working note in August as an attempt at that treatment: PSBT (BIP-174) and MuSig2 (BIP-327) coordination over a small mesh of independent relays, borrowing the remote-signer pattern of NIP-46 and composing with BSMS (BIP-129). Design goal: no relay holds key material, decrypts fragments, or learns the spending graph; the only capability an adversarial relay retains is denial of service. 

**Links**
 - Working Note (v1.0): [PSBT Coordination over Nostr Relays]( https://intelligence.custodyagents.com/articles/psbt-over-nostr ) 
 - Reference PDF, co-published yesterday with BitVault (v1.0): [Vault Construction and Coordination Transport]( https://intelligence.custodyagents.com/papers/two-orthogonal-layers/ ), SHA-256 `0e54a55c6cbb846a7da505d9260623b6403d37adbec7fc09d371ae92a0a72eea`, digest file at `/papers/two-orthogonal-layers-v1.0.pdf.sha256` for `sha256sum -c`


The note names five open problems. I want to lead with the sharpest, because it is the one an aggregate keypath makes concrete rather than theoretical.

**Nonce safety under adversarial delivery.** BIP-327's proof assumes a signer never reuses a secret nonce and assumes orderly communication. A relay substrate delivers at-least-once: retries, duplicates, reordering. The transport therefore inherits obligations (single-consumption nonce state, per-session sequencing, idempotent message identity), and property-based testing over that state space finds counterexamples in naive clients but does not prove the invariants sufficient.

Three questions I would value answers to, in decreasing order of how much they would change the design:

1. Under at-least-once delivery, is per-session monotonic sequencing enforced client-side sufficient for secnonce uniqueness, or does the aggnonce-via-relay path in PartialSigVerify open an identifiability or replay gap that requires a stronger transport commitment?
2. Policy-blind transport: a relay that validates what it forwards becomes a censor; one that forwards anything becomes a spam vector. Can garbage resistance sit anywhere other than fully client-side, and if not, what is the minimal client-side structure that keeps the relay blind?
3. Accountability without a global observer: is there a per-message cryptographic structure (signed transcript hashes, chained acknowledgements) that lets honest parties prove relay or peer misbehaviour without reintroducing a party that logs everything?

Criticism of the architecture as a whole is equally welcome; the note is a sketch with named gaps, not a shipped protocol.

-------------------------

Anzus_GemWallet | 2026-09-02 15:02:45 UTC | #2

From a wallet user’s point of view, it would help to see a simple failure example: what happens if one relay goes offline or sends the same message twice? Does the wallet recover automatically, or does the signing session need to restart? Clear status and recovery steps would matter before users trust this with larger amounts.

-------------------------

rafaelturon | 2026-09-03 12:55:40 UTC | #3

Good question, and it splits by what is observable rather than by which component failed.

**Relay offline.** A dropped connection is observable: the client fails over to the remaining relays in the set fixed at setup and the session continues, since every message is published to all relays and delivery needs only one common live path. A live relay that silently withholds is not observable; from the receiver's side it is indistinguishable from a slow relay or an offline peer. So recovery cannot depend on diagnosing the cause, only on local session state plus a timeout.

**Duplicate delivery.** The rule that keeps this safe is local to the signer, not a property of the relays: the secnonce is consumed once, inside `Sign` (libsecp256k1's musig module zeroes it on use), and every round output is cached. A retry means resending the same bytes, which is harmless, never recomputing. The hazard is the recompute path: relay delivers aggnonce A, signer signs, completion is withheld, client times out, A′ arrives "for the same session." A client that re-runs `Sign` with a retained secnonce leaks its key. The transport obligations in the note (idempotent message identity bound to session, round and signer; per-session sequencing) exist so the client can recognize repeats and reject stale sessions, not so it can trust delivery.

By state:

* Waiting for nonces: resend own pubnonce until acknowledged; on timeout, abort and open a fresh session with a new id and fresh nonces. Costs time, nothing else.
* Partial signature sent: resend the cached partial signature until acknowledged; on timeout, abort. Never re-sign within the session. An abandoned session cannot be revived because the secnonce no longer exists.
* Conflicting values for the same (session, round, signer): abort.

So: automatic recovery for connection loss and for lost or duplicated messages; explicit restart with fresh nonces on timeout; no path where the wallet silently retries a signature. For a user that reduces to three states: reconnecting, waiting on peer, session expired. Agreed this needs to be specified precisely before anyone trusts it at size; that is the substance of Problem 4.

-------------------------

Anzus_GemWallet | 2026-09-04 14:02:32 UTC | #4

That three-state explanation makes this much easier to understand. One practical detail may also be worth documenting: can users safely close and reopen the wallet while it is reconnecting or waiting on a peer, or should they keep the session open? That seems like a situation ordinary users could easily encounter.

-------------------------

