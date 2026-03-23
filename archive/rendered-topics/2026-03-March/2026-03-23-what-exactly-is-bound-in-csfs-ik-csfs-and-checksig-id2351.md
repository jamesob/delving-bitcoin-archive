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

