# [Lightning] Batched Splicing Considered Risky

ZmnSCPxj | 2023-11-08 14:45:57 UTC | #1

Good morning delvingbitcoin,

I believe at least part of the effort towards a splicing protocol
also includes a sub-protocol by which it is possible to add
arbitrary inputs and outputs to the splicing transaction.

The intent of this sub-protocol is to allow batching of additional
onchain operations to the splice.

For instance, suppose I have a channel with peers B and C.
Suppose I have no channel with a potential peer D.

Now suppose I wish to splice out from the channel with B, and
also splice out from the channel with C, in order to fund a
new channel with D, which is intended to be a 0-conf channel
for a JIT channel operation.
And suppose that the splicing protocol, as well as my awesome
code, is sophisticated enough to be able to actually batch
these operations in a single large transaction.

Now, the transaction would have inputs:

* channel-with-B (pre-splice).
* channel-with-C (pre-splice).

And would have outputs:

* channel-with-B (post-splice).
* channel-with-C (post-splice).
* channel-with-D.

Suppose B actually is run by a competitor that wishes to
damage my interactions with my other peers in order to
convince my peers that I suck.
What can B do?

B can actually double-spend channel-with-B (pre-splice)
using a revoked commitment transaction.

Now of course, if B does that, then B will be punished
and will lose its funds in the channel-with-B.

But consider: why would ***I*** splice out from
channel-with-B?
Obviously I would do that if I have too much funds in
channel-with-B, which implies that it is very likely
that B has little funds in channel-with-B.
Thus, if B wants to hurt me, the cost of the attack
may very well be worth it, even with 1% reserve
(woe betide me if I allowed 0% reserve with B, or if the
channel was new and funded by me and thus B has no reserves
yet).

Now remember, I said:

> Now suppose I wish to splice out from the channel with B, and
> also splice out from the channel with C, in order to fund a
> new channel with D, which is intended to be a 0-conf channel
> for a JIT channel operation.

D, having seen me broadcast the batched transaction that includes
the open of the channel-with-D, has now, naively, sent the
preimage to the HTLC for the JIT channel payment.

But because B double-spent the entire batched transaction out
from under me (with help from a miner), it also double-spent
the JIT channel funding of channel-with-D.
Thus, I have inadvertently attacked peer D, caused by B
attacking me.
This is a bad security breach escalation; it is bad enough
for me to be attacked, but now I am induced to also attack
others.

This implies that B (and C!) can cause disruption to my other
peers, because I batched splicing of channels with other
onchain operations, in this case, a 0-conf channel open.

If B is a competitor, then the fact that B can induce me to
perform an attack on my client D would let them redirect D
towards them.
Thus, there is a strong incentive to attack me, as it allows
B to move my hands and commit the crime.

Note as well that this is splice-out and not splice-in.
We already know that splice-in has even worse problems, with
the splicing-in peer also being able to double-spend this
out from under you using unilaterally-controlled funds, with
no risk of punishment.

A similar attack is also possible with batching dual-funded
openv2.
If we dual-fund with one peer, that peer can also disrupt
the channel openings of other peers in the same batch.

Of note is that even if I "just" batch splicings, the attack
remains a massive nuisance.
One good reason I might have to splice out is in order to
splice in to a different peer.
Ideally, both operations should be batched in a single
large transaction, but that leaves me vulnerable to attacks
from both peers.

For example, suppose I want to splice out from peer B and
splice in the same amount to peer C, and there is **no**
peer D involved in a JIT channel.
If I batch both splices in a single transaction, and B
disrupts the batch, I need to be able to tell C "forget
about the splice, let us go back to our original channel
funding output", without having to close the channel-with-C,
in order to try again with a different batched splice that
no longer involves the peer B, or to splice in to C with my
onchain funds instead of from the channel-with-B.

(Instead of telling C it would be better if C monitored
the chain for double-spending of the splice in transaction,
and then automatically negotiate with me to disregard the
splice, but that increases complexity of splice code even
more;
because of timeliness concerns it would be best if this was
not a timeout but rather an actual detailed check of the
txins of the splice in transaction.
In particular, this may make batched splicing non-viable
for usecases where users need to use SPV.)

In current Lightning Network without splice, in order to
move my funds from a channel with a peer, to a different
peer, I have to do a circular routing rebalance.
In that case, I will end up paying directly to the peer I
am moving liquidity away from, via routing fees.
Thus, that peer has a disincentive against disrupting my
circular routing, as they would lose the routing fee if
they disrupted it.

But with splicing, there is no routing fee to pay to the
peer that I am moving liquidity away from.
Thus, there is no disincentive against disrupting a batched
splice.
(other than loss of the reserve, of course, but there may
be situations where I splice out of a channel where the
peer has no reserve yet, and that is likely if I have put
too many funds into that channel and that peer has not
been receiving;
this is one reason for me to do a splice out, but the
peer now has the ability to hostage those funds by
disrupting a splice out that is batched with a splice
in to a different peer.
Thus, batched splicing considered risky.)

It has been proposed elsewhere that the splice transaction
can be re-created periodically, such as once a block at
least.
However, we should note that a miner can select *any* of
the still-valid splice transactions, including the
non-latest splice transaction.
This may happen by sheer accident, such as the winning
miner having a slow mempool update from your own mempool.
Every new splice transaction would thus require a new
set of commitment transactions per update, because
*any* of the splice transactions could confirm.
i.e. if there is a splice transaction that does not
confirm, and me and my "friends" make a new one, then
every update of every channel in the splice must by
necessity include signatures for commitment transactions
spending the per-splice, the first splice, and the second
splice transaction output (3 sets of signatures!) and
would require exchanging another set of signatures for each
additional splice transaction.
This solution thus greatly increases complexity and may
very well have more losses due to bugs than the problems
it tries to fix.

The above *can* be ameliorated with `SIGHASH_ANYPREVOUT`.
However, this may make 0-conf channel use-cases worse, as
`SIGHASH_ANYPREVOUT` inherently allows RBF (by adding new
outputs that pay fees) and thus the client code now has to
rely on a particular address and output amount appearing
on some confirmed transaction, instead of a specific
transaction ID getting confirmed, i.e. JIT channel
specifications that want to reduce trust in the LSP need
to be modified to consider the address and output amount
instead of the transaction ID, but only if
`SIGHASH_ANYPREVOUT` actually gets added to consensus.
Prior to `SIGHASH_ANYPREVOUT`, the address and output
amount is insufficient and the client needs to check for
the specific transaction ID, as the signatures it holds
are for the specific transaction ID.

Regards,
ZmnSCPxj

-------------------------

