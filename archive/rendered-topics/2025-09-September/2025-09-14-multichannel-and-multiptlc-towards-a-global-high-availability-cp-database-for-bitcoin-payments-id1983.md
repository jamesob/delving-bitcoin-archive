# MultiChannel and MultiPTLC: Towards A Global High-Availability CP Database For Bitcoin Payments

ZmnSCPxj | 2025-09-14 21:43:32 UTC | #1

Title: Towards a High-Availability CP Database For Bitcoin Payments

# Introduction

The CAP Theorem is: You can have any two of these three:

1. Consistency
2. Availability
3. Partition-Tolerance

Now, because network is unreliable, what you actually *can* have
are any one of these two:

1. CP - Consistent and Partition-Tolerant
2. AP - Available and Partition-Tolerant

The Bitcoin blockchain is ultimately a database — the
blockchain itself is a write-ahead log that is never reset, and
transactions delete entries (UTXOs) and add new entries (updates
are effectively an atomic deletion followed by an insertion, thus
Bitcoin is actually a full ACID CRUD database).

The Bitcoin blockchain is an AP database.
Due to the thousands of fullnodes (some of which even have hashpower)
deployed worldwide, the Bitcoin blockchain is never going to go down.

However, it *does* allow for temporary inconsistency.
In particular, zero-conf transactions may later be reverted, and if
you read zero-conf transactions, you’ll be getting dirty reads
(There Ain’t No Such Thing As A Global Mempool problem).
Even confirmed transactions may theoretically still be later
reverted, though the probability of that grows very low with more
confirmations.
Thus, to ensure consistency, payments on the blockchain are slow,
in order to ensure that potential inconsistencies settle down into
confirmed-committed transactions.

But the gold standard for high-speed global financial applications
is a high-availability CP database (which prioritizes Consistency
over Availability, but does have mitigations to improve its
Availability in practice).
And the Bitcoin blockchain, as we have demonstrated, is not a CP
database, but an AP one (and not even a high-consistency AP
database).

# Lightning as a Global CP Database

Surprisingly, we have actually managed to build and deploy a
global CP database built on top of the AP database: Lightning
Network.

It is not a *high-availability* CP database, but it *is* a CP
database, and a globally CP database at that.
Because of its CP nature, financial payments on the Lightning
Network can be very fast, as there is no need to wait to
reduce the incidence of dirty reads due to inconsistencies.

The core of the Lightning Network being a *global* CP database
lies in two constructions:

1. The Channel, a *local* two-node CP database for financial
   transactions that can only safely hold money for the two
   nodes on it, with the ability to publish its final state
   at any time on the “layer 1 blockchain” aka the AP
   database we are building on.
2. The HTLC, a financial contract that can be used to ensure
   atomicity across multiple financial databases.

The Channel is local: it can only safely hold money for the
two nodes that are managing the database.
However, it is very CP, using a bespoke consistency protocol:
we have a hand-over-hand for updating to the next database
state (creating an intermediate metastate where both are
valid), then revoking the old state, leaving just the next
state as valid.
Revocation is thus the point at which consistency is assured,
as it prevents the other side from putting old state and thus
consistent with the latest state.
In case of a network partition (i.e. connection lost, then
reestablished) the two nodes show the signatures for the
latest state that the other side has provided as well as the
latest revocation they have, thus proving to the other node
what the latest state actually is.

We then build up these local CP databases into a *global*
CP database by using HTLCs to ensure consistent updates
across multiple, local CP databases.
The HTLC is effectively a row-level lock on some amount of
coins on the CP database, locking their use until either
the payment reaches the receiver, or some other failure
occurs.
Deadlock is prevented by the simple expedient of doing
locks from the order sender to receiver.
A payment is thus composed of a series of locks on multiple
local CP databases, and once the payment is received, the
locks are freed in reverse order.

## Lightning Flaw: Low-Availability

The Lightning Network is *not* high-availability, meaning
that in case of network partitions, sometimes payments are
impossible.

For example, if an HTLC has already propagated, and one of
the nodes along the path goes down, then the payment update
cannot finish at the sender.
In case of some other failure, the payment may also fail
before it reaches the receiver, and because the sender and
the receiver cannot be sure of what the status of the
HTLC-lock-chain, they are unable to do anything to complete
the payment: the sender cannot safely send out another
alternate payment without trusting that the receiver will not
spend both the original and the alernate payment, and the
receiver literally holds no money at this point and cannot
trust the sender claim that it has sent out the funds.

Thus, the Lightning Network availability drops whenever a
public routing node goes offline.

The reason why Lightning Network is *not* high-availability
CP lies precisely in the Channel:

> The Channel, a local *two-node* CP database for financial
> transactions

Having only two nodes means that if one of the nodes is down,
there is simply no way for the remaining node to update the
database state.
In order to get a *high-availability* CP database, we need at
least *three* nodes, not two, and the rule that if one of
them goes down, the remaining two can keep updating the
database.

The more general rule is that if at least 50%, plus 1, are
available, they can update the database state.

The problem with the typical high-availability CP database
rule above (“three can keep updating if one of them is
dead”) is that it is in conflict with an important principle
of self-custody:

> ***NOT YOUR KEYS, NOT YOUR COINS***

On the underlying Bitcoin AP database, deletions of UTXOs
are authorized by signatures.
And thus, for the Lightning Channel two-node local CP
database, the authorizers are *both* nodes, and both must
provide signatures whenever they are updating the local
CP database.

Obviously, if a node is down, it *cannot* provide a
signature.
If we were to naively follow the high-availability CP
database rule of “three can keep updating if one of them
is dead” then that implies that only two nodes need to
provide signatures for a three-node construction.
But that means that the dead node does not have its key
used to spend the money: the fund is spendable without
its signature, i.e. it can be spent with
***NOT ITS KEY***.
Thus, that node has to assume that its funds are
potentially ***NOT ITS COIN*** if the remaining two nodes
collude to steal the coin.

# High-Availability CP With Your Keys, Your Coins For End Users

We have established two facts:

* For high-availability CP, we need at least 3 nodes, and
  allow 2 to update if 1 is dead.
* ***NOT YOUR KEYS, NOT YOUR COINS*** requires consensus,
  i.e. everyone has to be around to sign.

On the face of it, the two facts imply that we cannot have
high-availability CP in combination with self-custody, at
least with current schemes available in Bitcoin.

However, we can instead degrade the strong self-custody
requirement as follows:

* The public forwarding nodes have significant resources
  to check the trustworthiness of other public forwarding
  nodes.
  They also have a public reputation, which, if destroyed,
  also potentially destroys their income stream (routing
  fees).
* The end user does *not* have the resources to check the
  trustworthiness of *any* public forwarding nodes.
  They also do not have a public reputation, and are
  therefore less trustworthy-by-default than public
  forwarding nodes.
* Therefore:
  * The end user ***MUST NOT*** trust any public forwarding
    nodes, because they might not have the resources to
    check if the public forwarding nodes are trustworthy.
  * The public forwarding nodes ***MUST NOT*** trust the
    end user, because the end user is literally some random
    Internet person.
  * The public forwarding nodes *can* trust other public
    forwarding nodes, by doing things like know-your-business,
    service level agreements, legal contracts, being owned by
    the same company, and so on.

What I propose, then, is:

* Have a 2-of-2:
  * One is the end user.
  * The other is a 2-of-3 of 3 different public forwarding
    nodes.
    (Or, more generally, some `k`-of-`n` of public
    forwarding nodes where `n > 3` and `k >= 1 + floor(n / 2)`).
    * This implies that the each of the public forwarding nodes
      have to trust that a quorum of the other public forwarding
      nodes does not steal funds.
      This is trivially achievable if the public forwarding
      nodes are run by the same corporation; they might do
      this precisely so that they can keep high availability,
      such as doing rolling updates of their nodes.
      Alternately, public forwarding nodes may have legal
      guards to protect against misbehavior of other public
      forwarding nodes, without requiring the same legal
      guards against end users (or for the end users to
      require legal guards against any or all of the public
      forwarding nodes).

> ***Digression*** I mean — if you are pushing Spark or Ark
> or some other statechain-based mechanism or federated mechanism
> where end users have to trust a quorum of big corporate service
> providers, then you are making an assertion that big corporate
> service providers are trustworthy.
> If so, and you are a big corporate service provider, put your
> money where your mouth is: trust a quorum of those other big
> corporate service providers yourself, instead of forcing end
> users to trust them.
> If you will not do that, then that reveals your actual
> assessment of the risk of trusting big corporate service
> providers.
> Yes, you are putting more funds into this than an end user,
> but the end user is also much poorer than a big corporation
> and the loss of those funds would cut just as deeply for
> them.
> Hey at least you do not have to trust end users with this
> mechanism, unlike SuperScalar where you trust that they will
> actually come online to help you move liquidity where it is
> needed, or to cooperatively exit before the tree times out,
> or else you pay more onchain fees.

(Note: as there is currently no proof that FROST-in-MuSig2 is
safe, we should use explicit SCRIPT-based checking of two
signatures, one of which is the end user signature, the other
being the FROST of the public forwarding nodes.)

The reason for using 2-of-3 is as noted above:

> “three can do an update if one of them is dead”

This is then where we upgrade the low-availability CP
database of the plain Channel to a high-availability CP
database of this novel construction, which I will now dub
the MultiChannel.

(Unfortunately, MultiChannels can only use PTLCs, not
HTLCs, as I will show below.)

## MultiChannel Sending

First, let me introduce the PTLC.
Like the HTLC, the PTLC is effectively a row-level lock on
some funds, which can then be either moved to the receiver,
or returned to the sender.
Like the HTLC, the PTLC can be used for locking across
multiple local CP databases in order to allow for global
updates that let senders send money to distant receivers.

The advantage of the PTLC is that it allows certain kinds of
math to be used.

The reason for introducing PTLCs here is that we want the
end user to send out just *one* PTLC on its MultiChannel
with the public network, and then for the public forwarding
nodes to be able to send out PTLCs in parallel, on behalf
of the sender, until one of them is able to have the
payment reach the receiver.
Then, we must ensure that the receiver is able to claim
only one of the payments, even if multiple arrive at the
receiver.

Now, “send out multiple PTLCs in parallel, of which only
one should be claimed at the receiver” is precisely the
scheme of [stuckless payments](https://diyhpl.us/~bryan/irc/bitcoin/bitcoin-dev/linuxfoundation-pipermail/lightning-dev/2019-June/002029.txt).
The stuckless payments proposal, however, requires PTLCs.

Briefly, a stuckless payment involves two secret scalars
per payment attempt:

* One is known by the receiver, and its revelation to
  the sender is “proof of payment” because the sender
  cannot know it unless the payment is received.
  * All payment attempts use the same proof-of-payment
    secret.
* One is known by the sender, and its revelation to the
  receiver allows the receiver to claim this attempt.
  * Each payment attempt has a unique receiver-can-claim
    secret.

The reason for the PTLC is that the PTLC can be arranged
to be claimable only if the receiver knows both the
proof-of-payment secret *and* the receiver-can-claim
secret: this is done by having the PTLC require a single
scalar that is the known sum of the two secrets (this
requirement can be imposed without actual knowledge of
either secret, only the knowledge of the two points
that hide the secret).
In practice, we also want a third secret. the “delta”,
that is used to obfuscate the actual proof-of-payment
secret from forwarding nodes, and is changed at each
hop to reduce correlatable data if a surveillor has
control of multiple public forwarding nodes on the
network.

The primary difference between stuckless payments via
PTLCs versus HTLCs is this:

* HTLCs lock rows in multiple databases, and allows the
  receiver to claim *any* or *all* of the funds that
  reach it with the same hash of the HTLC.
  Thus, it is unsafe to have multiple HTLC-lock-chains
  in flight that total to more than the payment amount,
  because the receiver can commit the transfer for all
  those funds.
* PTLCs allow the sender to also be required in order
  to commit any PTLC-lock-chains.
  Thus, the sender can ensure that only one
  PTLC-lock-chain is committed (or more generally, that
  the amounts locked by PTLC-lock-chains it allows to
  commit sum exactly to the agreed payment amount), and
  the other PTLC-lock-chain are forced to roll back.
  This allows the sender to make multiple parallel
  PTLC-lock-chains terminating at the receiver, because
  the receiver can only commit any PTLC-lock-chain (and
  finalize the transfer) if the sender allows it.

Suppose an end user, Ursula, with a MultiChannel to 3
forwarding nodes, Alice, Bob, and Carol, wants to send
out a PTLC-based payment to some remote receiver.
The end-user first maps out multiple paths to the
receiver.

Source routing has the drawback that the sender has
little information about remote channel state, but has
the major advantage of not revealing the sender or
receiver to intermediate forwarding nodes.
By simply preparing multiple possible paths beforehand
and using a novel construction I call the MultiPTLC,
Ursula the user can delegate retrying to the forwarding
nodes and go offline afterwards, while still preserving
good privacy of payment receiver.

Once Ursula has mapped out multiple possible paths to
the destination, it creates a MultiPTLC, with a branch
for each of its mapped-out paths.

### The MultiPTLC

The MultiPTLC is simply a contract with multiple P
branches and a single TL branch.
If you know what a PTLC looks like, then you know what a
MultiPTLC looks like from that sentence.

Basically, each of the P branches is a payment attempt.
That branch pays out to *one* of the forwarding nodes,
Alice, Bob, or Carol, depending on whether that payment
attempt starts at that node or not.
The scalar demanded by each one is different, because
as noted above, in the stuckless payments scheme, each
attempt has a different receiver-can-claim secret, and
each P branch does demands a different sum of
proof-of-payment secret plus receiver-can-claim secret.

As noted, each P branch is one payment attempt.
There are likely more payment attempts than there are
actual forwarding nodes, because there are likely
multiple possible paths for each forwarding node.
Thus, there are multiple P branches that go to Alice,
one for each payment attempt that starts at Alice,
and so on for Bob and Carol as well.

In actual implementation, what happens is that the
output at the Poon-Dryja layer is another 2-of-2,
where one of them is Ursula the user and the other is a
k-of-n of the public forwarding nodes Alice, Bob, and
Carol.
An alternate branch of the 2-of-2 is the timelock going
back to Ursula.
Then there are multiple transactions spending that
output, pre-signed by Ursula, that go to each of the P
branches, plus another revert-to-Ursula timelock: in
effect, each of the branches is just a plain PTLC.

***The advantage of the MultiPTLC is that Ursula the
end-user sender is only required to lock exactly the
amount they want to pay plus routing fees.***
This is a major difference from the plain stuckless
payment model, where the ultimate sender has to lock
multiple times their payment amount, one for each
parallel attempt.

This advantage is important:

* We expect Ursula the end-user to not have a lot of
  liquidity on the MultiChannel.
* We expect Alice, Bob, and Carol the public forwarding
  nodes to have a lot of liquidity on the public network,
  and can thus afford to lock more of that liquidity in
  parallel outgoing PTLCs to the receiver in a race to
  reach the receiver first.

Suppose that Bob is the first to get to the receiver,
and then the receiver asks Bob for the receiver-can-claim
for that attempt.

Actually, back when Ursula was giving over the MultiPTLC
and its signatures and the onions for each attempt, it
was also giving over the receiver-can-claim secrets.
Ursula gives them, encrypted to the respective public
forwarding node for that attempt, to the forwarding nodes.
After giving them all that data, Ursula can then go offline
and just let Alice, Bob, and Carol attempt the payment on
all the paths that Ursula pre-found.

Giving the receiver-can-claim secrets to the public forwarding
nodes is safe for Ursula: Ursula is only committing the exact
payment amount (plus fees).
The onus is then on the consistency protocol of Alice, Bob,
and Carol to ensure that only one of them sends the
receiver-can-claim secret that allows one of the
PTLC-lock-chains to commit.

Then, when the receiver asks Bob for the receiver-can-claim
secret, Bob asks Alice and Carol to sign for the P branch
that goes to that attempt for Bob (Ursula is not needed,
as Ursula has already signed for all the P branches).
As the remaining signature to enable the P branch requires
k-of-n, even if either Alice or Carol is down, it is enough
for Bob to create the signature with the other surviving
public routing node, thus keeping high-availability CP.

Then Bob can sign the P branch itself, which then lets,
via Magic Math, Ursula learn the sum of proof-of-payment
plus receiver-can-claim plus delta, once Bob has presented
it to Ursula either onchain or offchain.

With that assurance, Bob can send the correct
receiver-can-claim secret to the receiver, and once the
receiver claims the payment, Bob now knows the sum of
proof-of-payment plus receiver-can-claim plus delta, but
does not know the delta (Ursula knows the delta and
receiver-can-claim secrets, so it can learn the
proof-of-payment by subtracting both).
Bob can now use that secret to claim the funds from the
enabled P branch of the MultiPTLC.

Now you might ask: what if both Alice and Carol refuse to
sign for the P branch going to Bob?
Well, then Bob cannot earn routing fees, because it cannot
safely send the receiver-can-claim secret and the receiver
will then try one of the other attempts that reach them,
which could then go to Alice or Carol, who are clearly
colluding to steal the routing fee from Bob.
Is that a problem?
Sucks for Bob, but remember earlier?

> * This implies that each of the public forwarding nodes have
>   to trust that a quorum of the other public forwarding nodes
>   does not steal funds.

Routing fees are just funds, thus stealing the routing fees
is just stealing funds, and we already gave the assumption
that the public forwarding nodes have to trust that a quorum
of the other public forwarding nodes does not steal funds (and
therefore routing fees).

If Bob has already gotten the signature for the P branch for
the first-place payment attempt, it’s still possible for
another payment attempt to reach the receiver.
Suppose that Bob got a signature from Alice to enable the P
branch.
Then Carol gets the second-place payment attempt, requesting
for the receiver-can-claim secret.
Bob and Alice can then collude to keep the fact that the P
branch was already enabled for Bob, and sign for the P branch
going to Carol, but again, see above.

Now, who is “first place” and “second place”, if the race is
very close, and we are in a relativistic universe where
“global order” is a delusion of merely human minds?
Alice, Bob, and Carol can use standard Raft consistency
protocols (or any other quorum consistency protocol) to decide
on who wins the race; the assertion “Bob wins” or “Carol wins”
is simply two conflicting transactions to the database.
Note that Ursula is not involved here; it can be completely
offline once it has sent out the send-payment request with all
the necessary data and signatures to a quorum of the public
forwarding nodes.

Note that even inside a plain Channel, a MultiPTLC still has
some of the same benefits: the public routing node can
perform the payment attempts, in parallel, after the user
Ursula has given all the necessary data (and in particular,
Ursula can go offline while the public routing node is
attempting payments, and Ursula only needs to lock the
exact amount it wants to send).
However, the issue with using a MultiPTLC inside a plain
Channel is that if the *single* public routing node goes down,
Ursula cannot make payments or determine if the payment went
through or not, again, because of the lack of a sufficient
quorum to safely update state; with a MultiChannel, the
availability of the CP database is much higher.

The drawback of the MultiPTLC is that it requires that
pretty much the entire network upgrade to use PTLCs instead
of HTLCs, as well as to support a stuckless payments protocol
that is compatible with getting the receiver-can-claim secret
from the LSP instead of the ultimate sender.
Good luck with that.

Nothing really prevents the use of HTLCs within a MultiChannel.
However, the advantage of having high availability at the
MultiChannel hop is lost because Ursula can only safely make
one payment attempt at a time, meaning that if there are
further availability issues beyond the MultiChannel, the payment
remains stuck and Ursula suffers low availability.

So actually yes, we can still use HTLCs on this MultiChannel
scheme, but it still does not achieve global high-availability
CP database semantics, only local high-availability CP database,
which is insufficient for actual payments.

In effect:

* MultiChannel protects against *local* availability issues,
  i.e. where one of Alice, Bob, or Carol are down.
  * MultiChannel hosting plain HTLCs still suffer from
    availability issues if a remote forwarding node goes down,
    and Ursula needs to be online to perform alternate payment
    attempts if the current payment attempt fails.
* MultiPTLC (actually stuckless payments, but MultiPTLC allows
  for less funds to be locked at the sending end user hop than
  plain PTLC-only stuckless payments) protects against *global*
  availability issues, where some remote node goes down while
  it holds PTLC locks that are associated with the lock on
  Ursula funds on the direct (Multi)Channel.
  * MultiPTLC in a plain Channel with a single forwarding node
    still suffer from local availability issues if that single
    forwarding node goes down.

Both are needed for a *global* high-availability CP payments
database.

Both the MultiPTLC and the MultiChannel, allow the funds that
Ursula has, to be pointed at *one of multiple nodes*, instead
of just being pointed at one node which can have availability
issues.

## MultiChannel Receiving

This simply uses a plain PTLC funded from the funds of
whichever one of the public forwarding nodes got the payment.
Once Ursula the user comes online, a quorum of the public
forwarding nodes can update the state and forward the funds
to Ursula.

Clearly, each of the public forwarding nodes has to have
some amount of liquidity towards Ursula, and in the actual
channel state, there would be separate outputs for those.

This is left as an exercise to the reader.

It should be a well-known fact by now that senders get all the
payment failure data, and receivers simply get no payment
failure data.
Thus, the interesting part is getting senders to have higher
probability of successfully paying, because there is simply
***nothing*** the receiver can do if none of the lock-chains
even reach it; it cannot even know about the inability to get
paid.

## Poon-Dryja MultiChannel

With Bitcoin ossified, there are only a few viable offchain
mechanisms.
Poon-Dryja is one of them, and is what is currently deployed
on the Lightning Network.

Largely, Ursula the user talks with a quorum of Alice, Bob,
and Carol, and they talk almost entirely using the Poon-Dryja
consistency protocol.
This lets Ursula rely on standard Poon-Dryja assurances when
assessing security risks from the public forwarding nodes, and
vice versa, the public forwarding nodes can rely on standard
Poon-Dryja assurances to assess security risks from Ursula the
user.

However, the issue is with the revocation secret.
We require that a quorum of Alice, Bob, and Carol is needed
to release the revocation secret to Ursula.
Otherwise, if all of the public forwarding nodes know the
same secreet, all it takes is for one of them to collude
with Ursula to reveal the secret “early”, for the latest
state, and then trigger a closure of the MultiChannel at
the latest state, which is now revocable by Ursula.

Rather than a revocation secret, we have a revocation
transaction; the normal claim path is locked with a
relative timelock (as is normal for Poon-Dryja) while
the revocation transaction has no timelock, and can be
claimed.
The revocation transaction then has to be signed by a
quorum of Alice, Bob, and Carol — standard k-of-n
scheme — and puts the funds to Ursula.
Thus, to update the state, a minimum quorum of Alice,
Bob, and Carol is needed to provide revocations to
Ursula, a necessary part of the Poon-Dryja consistency
protocol.
However, *only* a quorum is needed, so even if one of
Alice, Bob, or Carol is down, the remaining are still
enough to update the CP database and achieve local
high availability.

Of course, a quorum of Alice, Bob, and Carol can still
collude with Ursula to steal funds by giving the
revocation “early” (i.e. giving revocation for the latest
state instead of only older states), but again:

> * This implies that each of the public forwarding nodes have
>   to trust that a quorum of the other public forwarding nodes
>   does not steal funds.

The 2-of-2 layer of the nested 2-of-2 where one is a k-of-n
scheme means that the user Ursula has the same security posture
as with plain 2-of-2 Poon-Dryja Channels: Ursula needs to pay
attention to the blockchain and watch for theft attempts.

The problem now is that our current Poon-Dryja
implementations use a “shachain” construct to store
revocation secrets.
The “shachain” is an O(1) space, O(log N) insertion,
O(log N) lookup structure for a sequence of revocation
secrets.
When we switch to providing signatures for revocation
transactions instead of revocation preimages like our current
Poon-Dryja implementations, the O(1) space shachain is not
useable anymore.

In short, the drawback is that Ursula needs O(n) storage
for each open MultiChannel.

The advantage is that Ursula only needs to maintain one
MultiChannel, as that single consturct can be used to pay
via multiple forwarding nodes without requiring additional
liquidity in a hot wallet, so this may be a better tradeoff
in practice than having to maintain multiple channels to
multiple public forwarding nodes.

Alternatively, we can use the Decker-Wattenhofer
decrementing-`nSequence` mechanism instead.
This does not require storing O(n) revocations, but instead
gets larger, variable timelocks that would also create
minimum timeouts for the MultiPTLCs in-flight inside the
MultiChannel.
We also have absolutely no experience in using these in
production, so there may be hidden edge-case gotchas
that we are not aware of, and may become future CVEs.
Another issue is that if you then embed MultiChannels inside a
SuperScalar, and if the MultiChannel is a Decker-Wattenhofer,
the SuperScalar also adds a few Decker-Wattenhofer layers,
adding even more variable delays that impact the minimum
timelocks for hosted MultiPTLCs.

Other multi-participant constructs are also available in an
ossified Bitcoin, but require significant amounts of funds to
be locked into bonds, and thus I normally do not bother
analyzing those; my model is that Ursula the user really wants
to have only a small amount of funds at risk on a hot wallet,
thus any bond-requiring construction is simply a no-go for
anything that is intended for last-mile connection to end users.
Analyses of the use of those constructs for a MultiChannel
is thus left as an exercise to the reader.

-------------------------

harding | 2025-09-15 20:15:08 UTC | #2

[quote="ZmnSCPxj, post:1, topic:1983"]
The advantage of the MultiPTLC \[over sender-created stuckless payments\] is that Ursula the end-user sender is only required to lock exactly the amount they want to pay plus routing fees.

[/quote]

Would you be able to explain more about why you think that’s an advantage worth the additional complexity?

[quote="ZmnSCPxj, post:1, topic:1983"]
We expect Ursula the end-user to not have a lot of liquidity on the MultiChannel.

[/quote]

What is the basis for that expectation?  Almost all of my payments with physical cash, LN sats, and bank credit are for less than half of my available liquidity (respectively: cash-in-wallet, sats-on-my-side-of-the-channel, and available credit); I would guess most of my payments are for less than 1/10th my liquidity.  User-initiated stuckless payments should be convenient under those conditions, with no need for additional third party liquidity.

In the rare cases where I make payments that approach the limit of my liquidity—conditions where stuckless payments might not function well or at all—I don’t usually mind waiting longer.  For example, I don’t want to wait more than a few seconds to complete a purchase of a $5 hot beverage but I don’t mind much that it often takes over a minute to complete a $2,000 purchase of airplane tickets on my current credit card (the bank usually requiring me to verify the purchase via text message).

-------------------------

cryptoquick | 2025-09-16 00:25:18 UTC | #3

Hmm

1. PTLCs aren't quantum resistant, so it would be a good idea to evaluate the risk in that context
2. I'm a fan of databases too and am a fellow CAP appreciator… But maybe consider not calling it a “global CP database”, especially given current discussions on CSAM

-------------------------

ZmnSCPxj | 2025-09-16 18:48:02 UTC | #4

[quote="harding, post:2, topic:1983"]
[quote="ZmnSCPxj, post:1, topic:1983"]
The advantage of the MultiPTLC \[over sender-created stuckless payments\] is that Ursula the end-user sender is only required to lock exactly the amount they want to pay plus routing fees.

[/quote]

Would you be able to explain more about why you think that’s an advantage worth the additional complexity?

[/quote]

As mentioned, my model is that Ursula has a small amount of funds.

In particular, as noted elsewhere, MultiPTLC really protects against ***global*** high-availability, i.e. when a ***remote*** routing node goes down while it is in the middle of a PTLC-lock-chain.  Suppose, as you mention, Ursula makes 10 stuckless payments using plain PTLCs.  Suppose one goes through, but one of the other 9 does not get a `update_fail_ptlc` message.  That fund remains locked, potentially for the entire lock time, which, since Ursula is the sender, is at its largest.  This can happen if one of the routing nodes along that path goes down while it has a PTLC on one or both of the channels it has on the route.

So tell me: how do I explain to  Ursula “Yes, the 1000 sats payment got through and I deducted it from your account, but now I have to hold ***another*** 1000 sats of your money because the Lightning Network had problems.”?  Sure, I could hide that fact and hope that Ursula never actually needs to zero out their wallet in the next two weeks, but if Ursula ***does*** want to drain their wallet, the jig is up and now I have just doubled down on my little white UX lie.

I mean… “stuckless” is really a lie.  It can still get stuck, it is just that we can now let them get stuck ***in parallel*** instead of having a strict required serialization where I have to wait for the current attempt to get unstuck before making a different attempt.  It is not really stuckless, it is actually “parallel stuckness is not a money loss risk”.  The locks still exists on the money. and the issue is that, because we do not have control over the ***rest*** of the network, we cannot ensure that the money will not get locked for two weeks due to failures of the remote nodes that we do not control.

The fact that MultiPTLC only locks ***one*** unit of the payment on the side of Ursula is nearer to how the mind of a human-implemented Ursula works: they have allocated ***one*** unit of the payment for this receiver, so it getting locked until the network is able to deliver it or not is what Ursula expects.  Ursula ***does not expect*** that multiples of their designated amounts get lockedd.

---

That is just ***one*** advantage of MultiPTLC.  Another is that it is significantly better for mobile devices with low or spurious connectivity.

By having just ***one*** unit of the payment locked on the side of Ursula, Ursula can outright hand over the receiver-can-claim scalars to the LSPs.  Then, Ursula device can calculate all the necessities — signatures, encrypted blobs, paths, onions — without touching the network, then send out all of those to at least two LSPs (or whatever minimum quorum) over a few dozen IP packets.  Then Ursula can go offline once again.  Ursula only needs those few dozen IP packets, and the rest of the payment flow is now handled by the LSPs.

Compare this to the sender-created “stuckless” payment, where Ursula needs to be constantly online, waiting for “request for receiver-can-claim scalar”.  Because Ursula is putting out multiples of the payment amount, it cannot hand over the receiver-can-claim scalars to the LSPs.  If Ursula loses connectivity after sending out the payment request, then Ursula cannot give the receiver-can-claim scalar and the receiver might decide to free up inbound liquidity early by giving up after a short timeout, leading to total payment failure.

The LSPs are better positioned to have high connectivity and high uptime.  Let us delegate the payment re-attempt flow and the receiver-can-claim scalar sending to them, not on Ursula, who could be low-connectivity and low uptime.

[quote="harding, post:2, topic:1983"]
[quote="ZmnSCPxj, post:1, topic:1983"]

[/quote]

[quote="ZmnSCPxj, post:1, topic:1983"]
We expect Ursula the end-user to not have a lot of liquidity on the MultiChannel.

[/quote]

What is the basis for that expectation?  Almost all of my payments with physical cash, LN sats, and bank credit are for less than half of my available liquidity (respectively: cash-in-wallet, sats-on-my-side-of-the-channel, and available credit); I would guess most of my payments are for less than 1/10th my liquidity.  User-initiated stuckless payments should be convenient under those conditions, with no need for additional third party liquidity.

In the rare cases where I make payments that approach the limit of my liquidity—conditions where stuckless payments might not function well or at all—I don’t usually mind waiting longer.  For example, I don’t want to wait more than a few seconds to complete a purchase of a $5 hot beverage but I don’t mind much that it often takes over a minute to complete a $2,000 purchase of airplane tickets on my current credit card (the bank usually requiring me to verify the purchase via text message).

[/quote]

As noted earlier, the problem is not just that Ursula might not have the liquidity, it is simply that it does not match the expectations of a naive human Ursula that multiple units of their payment amount get locked, and potentially can be locked for two weeks if bad things happen beyond the control of Ursula, the wallet developer, or the LSPs.  It is already bad enough that, with MultiPTLC, if ***all*** the parallel attempts by the quorum of LSPs fail but at least one of them gets stuck, the MultiPTLC is still stuck (the LSPs cannot safely revert it while at least one of their outgoing PTLCs are not reverted or commited, since Ursula might be a pseudonym of ZmnSCPxj the notorious hacker who can directly hand over the receiver-can-claim scalar to the receiver by sneakernet).

-------------------------

ZmnSCPxj | 2025-09-16 20:10:15 UTC | #5

Come to think of it, it is significantly better if it is only the LSPs know the receiver-can-claim scalar, as it protects against Ursula (who could be notorious hacker ZmnSCPxj in disguise) giving that receiver-can-claim scalar to the receiver.  Instead, the receiver-can-claim scalar is generated by the LSP for each attempt the Ursula wants to have in parallel (and during this stage, the LSP can indicate how many parallel attempts it allows Ursula to have at its node, by giving only a specific number of receiver-can-claim points when Ursula asks for it).

This worsens the mobile case a little, as now Ursula has to query the LSPs for receiver-can-claim points ***before*** it can do the offline preparation of routes and signatures and revocation keys and etc etc.  But now the MultiPTLC is now a multi-headed PTLC where all the points are `(delta + proof-of-payment) ` *`*`*`G`*,* and the point of the plain PTLC on the payment attempt on the LSP onwards is `(delta + proof-of-payment + receiver-can-claim) * G`.  At the receiver, the point has the `delta` (which is the sum of the hiding scalar at each hop) already cancelled out, and the point is now `(proof-of-payment + receiver-can-claim) * G`, and the LSP is the one who authorizes whether to give the receiver-can-claim scalar.

This allows the LSPs to rollback the MultiPTLC ***even if it has a PTLC-chain out there that is locked due to global availability problems***.  Once the LSPs rollback the MultiPTLC lock, they simply never reveal the receiver-can-claim scalars ever, and they can ensure this because they are the only ones that know the receiver-can-claim scalar (2-of-3 of the LSPs can secretly rollback the MultiPTLC lock to Ursula and never tell the third, but that is again covered by “LSP trusts a quorum of other LSPs” disclaimer).  With this scheme, ***global*** availability problems (i.e. remote nodes going offline while holding up a PTLC-chain involving you) ***do not affect Ursula at all***.  Only the LSP is affected by ***global*** low-availability issues, and it can afford that more readily than Ursula (and pass on the amortized cost via routing fees) simply because it has more funds in public channels than Ursula has on their dinky little MultiChannel.

The receiver-can-claim scalar can use the cryptographic trick that LDK uses for “stateless” invoices.  When Ursula asks for receiver-can-claim points, the LSP hands over a random 256-bit number and the receiver-can-claim point.  The LSP derives the receiver-can-claim scalar via `HMAC(lsp_secret, random-number-I-will-give-to-Ursula)`.  The `lsp_secret` here can be a separate key from the node key or onchain wallet key, because even if this key is lost, all it means is that the LSP cannot earn routing funds if it wins payment attempts, which is ***tiny*** compared to channel funds or onchain funds. That way, the LSP does not have to store this state until Ursula later gives it back in the MultiPTLC package together with the crypto stuff needed to lock the MultiPTLC on the MultiChannel, at which point Ursula has already staked an amount and the cost of storing the underlying receiver-can-claim scalar on the LSP can be justified (otherwise Ursula can DoS the LSP by asking for receiver-can-claim points and never using them, wasting storage at the LSP).

---

I ***REALLY REALLY*** want to emphasize that ***Lightning is a global CP database*** simply because of ***two*** things:

* The 2-node Poon-Dryja consistency algorithm which gives CP ***locally***.
  * Low availability because only 2 nodes.
* The HTLC lock-chain which upgrades the local CP to ***globally*** CP.
  * Low availability due to serial locking.

In order to mitigate the existing low-availability issues of both the Channel and the HTLC, we need ***two*** novel constructions that provide high availability, the MultiChannel and the MultiPTLC.

-------------------------

