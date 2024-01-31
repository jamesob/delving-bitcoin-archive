# 0conf LN channels and v3 anchors

MattCorallo | 2024-01-29 22:29:37 UTC | #1

It was mentioned on the previous LN spec call that one concern with v3 transactions is that we need to support 0conf channels in some way. Specifically, imagine a lightning channel between two parties A and B. A trusts B to not double-spend themselves (ie trusts them for a 0conf transaction), but may not trust B to not go offline and never come back. B does not trust A at all.

In such a channel, let's assume that it opens, a payment or two happens, and then A or B go offline and the other wants their funds back. Now you have a unconstrained transaction, a v3 commitment transaction spending it, and a CPFP anchor spend hanging off of the commitment transaction. In the current v3 transaction proposal, as far as I understand it, this package would not be broadcastable - the v3 transaction is required to have no unconfirmed ancestors.

If the funding transaction confirms on its own (or via CPFP if B has a change output on it, though that's certainly not a requirement we'd like to require) then we're fine, the v3 commitment and any CPFP fee-bumps will broadcast just fine, but if fees start to rise right after the channel closes (or the funding transaction is generated with a deliberately low fee, which is common as there's generally no rush on them), we'd be totally stuck.

I'm not sure what the correct fix is for tweaking v3 to allow this, maybe the funding transaction could be restricted to no unconfirmed inputs and then we simply allow an A -> B -> {C, D, CPFP bump} topology.

-------------------------

t-bast | 2024-01-30 10:24:58 UTC | #2

[quote="MattCorallo, post:1, topic:494"]
I’m not sure what the correct fix is for tweaking v3 to allow this, maybe the funding transaction could be restricted to no unconfirmed inputs and then we simply allow an A → B → {C, D, CPFP bump} topology.
[/quote]

I'm not sure that's sufficient, because you usually want to allow having a chain of unconfirmed splices. The topologies that v3 would then need to support would probably be too complex to analyze before cluster mempool...

[quote="MattCorallo, post:1, topic:494"]
If the funding transaction confirms on its own (or via CPFP if B has a change output on it, though that’s certainly not a requirement we’d like to require) then we’re fine
[/quote]

My plan was to have a change output on the funding/splice transaction on B's side to ensure that it could be CPFP-ed. This is indeed far from ideal, but that's something we already need today if we want to protect against pinning, so v3 doesn't make it worse?

-------------------------

MattCorallo | 2024-01-30 20:10:14 UTC | #3

[quote="t-bast, post:2, topic:494"]
I’m not sure that’s sufficient, because you usually want to allow having a chain of unconfirmed splices.
[/quote]

Mmm, that's a good point, though that is still a relatively simpler chain that we could still analyze assuming no additional unconfirmed inputs, no?

[quote="t-bast, post:2, topic:494"]
The topologies that v3 would then need to support would probably be too complex to analyze before cluster mempool…
[/quote]

All of the v3 discussion assumes cluster mempool :)

[quote="t-bast, post:2, topic:494"]
My plan was to have a change output on the funding/splice transaction on B’s side to ensure that it could be CPFP-ed.
[/quote]

I think we should strive, if at all possible, to have splice bumps be RBF, not CPFP. Seems nuts to prefer CPFP, no?

[quote="t-bast, post:2, topic:494"]
This is indeed far from ideal, but that’s something we already need today if we want to protect against pinning, so v3 doesn’t make it worse?
[/quote]

Sure, but if we're rethinking how we do mempool policy, we should probably think through these things so that we try to get as close as possible and don't end up with a v4 :)

-------------------------

morehouse | 2024-01-30 20:48:10 UTC | #4

[quote="MattCorallo, post:1, topic:494"]
I’m not sure what the correct fix is for tweaking v3 to allow this, maybe the funding transaction could be restricted to no unconfirmed inputs and then we simply allow an A → B → {C, D, CPFP bump} topology.
[/quote]

Without complicating the current v3 proposal, we could achieve similar results by making 0-conf funding transactions v3 and adding a shared anchor to them.  Then either party can CPFP in an emergency.

-------------------------

instagibbs | 2024-01-30 21:13:29 UTC | #5

[quote="MattCorallo, post:3, topic:494"]
All of the v3 discussion assumes cluster mempool :slight_smile:
[/quote]

V3 pre-dates cluster mempool, we can likely do significantly better afterwards.

-------------------------

t-bast | 2024-01-31 07:30:44 UTC | #6

[quote="MattCorallo, post:3, topic:494"]
Mmm, that’s a good point, though that is still a relatively simpler chain that we could still analyze assuming no additional unconfirmed inputs, no?
[/quote]

I'm not sure it is, because each splice may have a change output, so you end up with arbitrary trees of transactions...

[quote="MattCorallo, post:3, topic:494"]
All of the v3 discussion assumes cluster mempool :slight_smile:
[/quote]

That's not the way I understood it, the plan is currently to deploy v3 before cluster mempool to have a good enough solution to mitigate most pinning vectors. And then v3 can become more powerful once cluster mempool happens.

[quote="MattCorallo, post:3, topic:494"]
I think we should strive, if at all possible, to have splice bumps be RBF, not CPFP. Seems nuts to prefer CPFP, no?
[/quote]

But we don't have a choice, if we use 0-conf, we just can't use RBF safely, we have to use CPFP unfortunately. Otherwise there are trivial attacks that lets the other peer steal funds.

[quote="MattCorallo, post:3, topic:494"]
Sure, but if we’re rethinking how we do mempool policy, we should probably think through these things so that we try to get as close as possible and don’t end up with a v4 :slight_smile:
[/quote]

That's true! But I don't think we'd need any kind of v4, if v3 starts with a restrictive package topology and later relaxes it (while still using v3), that's ok? That's why I'd lean towards a first deployment of v3 for very simple packages that makes our lives easier sooner rather than later, and gets even better over time.

[quote="morehouse, post:4, topic:494"]
Without complicating the current v3 proposal, we could achieve similar results by making 0-conf funding transactions v3 and adding a shared anchor to them. Then either party can CPFP in an emergency.
[/quote]

The issue with this is that we cannot create chains of unconfirmed splices anymore (because v3 can have only one unconfirmed child right now), which in practice is quite limiting.

-------------------------

