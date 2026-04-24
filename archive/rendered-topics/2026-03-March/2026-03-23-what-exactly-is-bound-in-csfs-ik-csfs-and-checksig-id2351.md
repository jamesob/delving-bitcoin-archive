# What exactly is bound in CSFS, IK+CSFS, and CHECKSIG?

AaronZhang | 2026-03-23 01:41:09 UTC | #1

I’ve been running a small set of experiments to make the
differences between three Tapscript constructions more
observable.

| Pattern | Binding target | Replayable? |
|----|----|----|
| `OP_CHECKSIGFROMSTACK` | Message (stack-supplied) | Yes |
| `OP_INTERNALKEY` + `OP_CHECKSIGFROMSTACK` | Identity (internal key) | Yes, same key |
| `OP_CHECKSIG` | Transaction sighash | No |

A tentative framing: IK+CSFS changes *where authorization
is sourced from*, but not what the signature commits to —
same opcode family, different binding surface.

Repo (offline harness + Signet anchors + checked-in outputs):

https://github.com/aaron-recompile/bitcoin-signature-binding

Does this framing match how others think about it?

-------------------------

AaronZhang | 2026-04-02 19:42:07 UTC | #2

Follow-up after running some APO experiments:

Adding `SIGHASH_ANYPREVOUT` to the table:

| Pattern | Binding target | Replayable? |
|----|----|----|
| OP_CHECKSIGFROMSTACK | Message (stack-supplied) | Yes |
| OP_INTERNALKEY + OP_CHECKSIGFROMSTACK | Identity (internal key) | Yes, same key |
| OP_CHECKSIG | Transaction sighash | No |
| SIGHASH_ANYPREVOUT (BIP118) | Transaction sighash minus outpoint | Yes, same script + amount |

APO seems to sit somewhere between CHECKSIG and CSFS in terms of what gets bound. It removes the outpoint commitment but keeps the rest — amount, script, outputs. So the same signature can be reused across different prevouts, as long as those match. Not “unbound,” just selectively unbound on the input side.

One thing this made concrete for me: APO + CTV split the commitment problem across two surfaces — APO relaxes the input side (no outpoint), CTV fixes the output side (template). Neither is sufficient on its own, but together they form a workable constraint system.

This came up directly while building a Braidpool-style covenant demo — APO handles Eltoo-style RCA updates and CTV commits the UHPO payout template per round.

Maybe the question is **not just what is bound,** but **what is fixed vs what is left open — and on which side.** The table above is one attempt to make that more explicit.

See the construction here:：[https://delvingbitcoin.org/t/challenge-covenants-for-braidpool/1370/2](https://Challenge:%20Covenants%20for%20Braidpool%20\(post%20#2\))

-------------------------

AaronZhang | 2026-04-09 20:39:36 UTC | #3

Building on the framing above, one additional angle that became clearer to me:

Ordinals/Brc20 look like message/data publication, but the reveal is not message-bound — it is ownership-bound.

The data is fully exposed in the witness, but a third party still cannot front-run the reveal, because they cannot produce a valid signature for the committed UTXO.

Also, the data envelope is not executed, but it is still committed via the TapLeaf. Changing even one byte breaks the reveal.

So it seems to separate:

• CHECKSIG → who can spend

• TapLeaf → what is committed

This may also explain why CSFS is not used here. CSFS would bind a message, But it seems these data-embedding constructions need to bind the right to publish the data, not just the data itself.

One way I’m starting to think about this is as a spectrum:

CSFS → message-bound (replayable)

IK+CSFS → identity-influenced

CHECKSIG → transaction-bound (not replayable)

There’s a concrete Ordinals example here that helped me reason about this:

[Ordinals and BRC-20 — Data Envelopes in Single-Leaf Script Paths](https://github.com/aaron-recompile/mastering-taproot/blob/e9067dbeebb60d3ac054ce503d1dfc0168adbba5/book/manuscript/Chapter%2009%20Ordinals%20and%20BRC-20%20-%20Data%20Envelopes%20in%20Single-Leaf%20Script%20Paths.md)

-------------------------

AaronZhang | 2026-04-24 18:40:38 UTC | #4

Adding a fifth row to the binding framework, now that I have
on-chain results for both ladder rebinding and APO rebinding side-by-side.

| Pattern | Binding target | Cross-prevout reusable? | Witness scaling |
|----|----|----|----|
| OP_CHECKSIGFROMSTACK | Message (stack-supplied) | Yes | O(1) |
| OP_INTERNALKEY + OP_CHECKSIGFROMSTACK | Identity (internal key) | Yes (same key) | O(1) |
| OP_CHECKSIG (default sighash) | Full transaction sighash | No | O(1) |
| OP_CHECKSIG with APO (BIP-118) | Sighash minus prevouts | Yes (same script + amount) | O(1) |
| CSFS rekey ladder (Post #5) | Delegated key chain | No (per-channel) | O(n) in state count |

A few observations from the empirical side, just from staring at TxIDs:

1. APO and the CSFS ladder both achieve eltoo-style state replacement,
   but along different binding axes. APO loosens what the signature
   commits to; the ladder threads authority through fresh keys derived
   per state.
2. The witness-scaling difference matters in practice. APO’s O(1)
   keeps a long-running channel cheap regardless of state count; the
   ladder’s O(n) makes it unattractive past roughly ten states. So the
   ladder is a “primitives-first” alternative for short state chains,
   not a structural replacement for APO in long-lived protocols.
3. Cross-prevout reuse — APO’s distinguishing feature — is what lets
   the same signature spend a fee-bumped funding tx variant without
   re-signing. The ladder cannot do this because each ladder is anchored
   to one channel’s funding outpoint at script creation time.
   So if anything, the ladder strengthens the case for APO in the
   “long-running, fee-volatile channel” use case, while showing that
   short-state-chain eltoo can run today on already-activated opcodes.

References:
Post :APO+CTV eltoo, 6 txs:

https://delvingbitcoin.org/t/eltoo-state-chain-on-signet-three-rounds-six-transactions-apo-ctv/2413

Post: CSFS+CTV ladder, 2 txs:

https://delvingbitcoin.org/t/eltoo-state-chain-on-signet-again-three-rounds-two-transactions-csfs-ctv-with-rekey-and-ladder-no-cat/2430

-------------------------

