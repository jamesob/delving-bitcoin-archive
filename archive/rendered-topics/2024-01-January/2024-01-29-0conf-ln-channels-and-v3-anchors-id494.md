# 0conf LN channels and v3 anchors

MattCorallo | 2024-01-29 22:29:37 UTC | #1

It was mentioned on the previous LN spec call that one concern with v3 transactions is that we need to support 0conf channels in some way. Specifically, imagine a lightning channel between two parties A and B. A trusts B to not double-spend themselves (ie trusts them for a 0conf transaction), but may not trust B to not go offline and never come back. B does not trust A at all.

In such a channel, let's assume that it opens, a payment or two happens, and then A or B go offline and the other wants their funds back. Now you have a unconstrained transaction, a v3 commitment transaction spending it, and a CPFP anchor spend hanging off of the commitment transaction. In the current v3 transaction proposal, as far as I understand it, this package would not be broadcastable - the v3 transaction is required to have no unconfirmed ancestors.

If the funding transaction confirms on its own (or via CPFP if B has a change output on it, though that's certainly not a requirement we'd like to require) then we're fine, the v3 commitment and any CPFP fee-bumps will broadcast just fine, but if fees start to rise right after the channel closes (or the funding transaction is generated with a deliberately low fee, which is common as there's generally no rush on them), we'd be totally stuck.

I'm not sure what the correct fix is for tweaking v3 to allow this, maybe the funding transaction could be restricted to no unconfirmed inputs and then we simply allow an A -> B -> {C, D, CPFP bump} topology.

-------------------------

