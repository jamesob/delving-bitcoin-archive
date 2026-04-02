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

