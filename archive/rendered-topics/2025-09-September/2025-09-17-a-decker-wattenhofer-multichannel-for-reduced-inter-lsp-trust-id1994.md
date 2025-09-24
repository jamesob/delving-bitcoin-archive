# A Decker-Wattenhofer MultiChannel For Reduced Inter-LSP Trust

ZmnSCPxj | 2025-09-17 20:26:20 UTC | #1

Title: A Decker-Wattenhofer MultiChannel For Reduced Inter-LSP Trust

# Introduction

The [MultiChannel and MultiPTLC](https://delvingbitcoin.org/t/multichannel-and-multiptlc-towards-a-global-high-availability-consistent-partition-tolerant-database-for-bitcoin-payments/1983) constructions are novel cryptocurrency
systems that provide high-Availability, Consistent, and Partition-Tolerant
access to the Lightning Network for end-users.

Sadly, in order to achieve the high-availability goal, it degrades the
security of the LSPs, and requires that the LSPs trust their funds to a
quorum of the other LSPs in the same MultiChannel.

* The user Ursula ***does not trust*** the LSPs for ***fund safety***:
  not all of them together, not a quorum, not a single one, there is no
  trust necessary ***at all*** in any of them or in all of them.
  * The user Ursula ***does trust*** a quorum of LSPs to provide them
    with the ability to send and receive payments.
  * The user Ursula ***does trust*** a quorum of LSPs to not lock the
    Ursula funds for two weeks (or so) due to unilateral exit when
    Ursula is willing to cooperatively exit.
* The LSPs Alice, Bob, and Carol ***do not trust*** Ursula for
  ***fund safety***.
  * The LSPs Alice, Bob, and Carol ***do trust*** Ursula will not jam
    their public channels.
  * The LSPs Alice, Bob, and Carol ***do trust*** Ursula will actually
    utilize any inbound liquidity they provide to Ursula.
    This trust requirement can be removed if Ursula bought that
    inbound liquidity.
  * The LPSs Alice, Bob, and Carol ***do trust*** Ursula to not lock
    LSP funds for two weeks (or so) due to unilateral exit when a
    quorum of LSPs is willing to cooperatively exit.
* The LSPs Alice, Bob, and Carol ***do trust*** a quorum of their
  partner LSPs for ***fund safety***.
  * A quorum of LSPs can outright steal funds from an LSP with no
    recourse or theft punishment mechanism.
    * Even if this happens, fund safety is still assured for Ursula
      (and the theft would either require the cooperation of Ursula,
      or will nominally entirely benefit Ursula only).

In this writeup, I will present a variant of MultiChannel based on the
nested Decker-Wattenhofer construction, which *reduces* but *does not
completely eliminate* the trust that the LSPs have to have with a
quorum of their partner LSPs.

As a quick refresher, the full Decker-Wattenhofer Duplex
Micropayment Channel is a nested construction:

* One or more nested layers of decrementing-`nSequence`
  constructions.
* An innermost layer composed of two `nSequence`-variant Spilman
  unidirectional channels, one in each direction (the “duplex”).

As a bonus, this construction also achieves the 0.5-roundtrip of
“fast forwards” (and in fact, one can argue that the “fast forward”
scheme is just the stealth use of Spilman inside Poon-Dryja, which
is what actually achieves the 0.5-roundtrip).

# Motivation

Let us review our base onlineness assumptions of the participants:

* Ursula is ***usually offline***.
  * Ursula only comes online ***when she wants to pay*** or
    ***when she wants to check if she received***.
  * Ursula ***does not want to wait***; if she has to pay right
    now, she needs the Lightning connectivity service
    ***right now***.
* The LSPs are ***chronically online***.
  * However, because of the large amounts of money in hot wallets,
    the LSPs ***must take security seriously***.
  * This means that, at random times, when a security-critical
    patch is needed, ***the LSP MUST go offline to patch***.

The classic Channel construction does not serve Ursula well:

* Suppose at a random time, a security bug forces the LSP to
  go offline and patch the bug.
* Suppose while the LSP is offline, Ursula has to pay ***right
  now***.
  * Users do not care that 99.5% of the time the website loads
    in 0.05s, they care about that 0.5% of the time the
    website took 2.0s to load.
    Similarly, Ursula does not care that 99.5% of the timae,
    the LSP was online and could have facilitated their
    payment, Ursula only cares about that 0.5% of the time
    when Ursula needed to pay ***right now*** but the LSP was
    offline for “routine maintenance” BS.
  * Ursula *could* maintain multiple separate Channels to
    multiple separate LSPs.
    Unfortunately, that increases the liquidity requirements
    and increases onchain resource usage, which Ursula has
    to pay for, directly or indirectly.

With a MultiChannel, Ursula does not have to maintain separate
Channels to separate LSPs.
Ursula can create a single MultiChannel pointed at multiple
LSPs, and as long as at least a quorum of LSPs is online,
(e.g. 2 of 3, 3 of 5) then Ursula can still make a payment.

For example, even if the 3 LSPs Alice, Bob, and Carol are run
by a single corporation, when a critical security update is
needed, that single corporation can use a “rolling update”
scheme where it only takes down one LSP, updates it, brings
it back up, monitors it for a while to check for unexpected
issues, then takes down the next LSP, etc., until all the
LSPs are updated.
With a MultiChannel, the rolling updates ensure that Ursula
still has continuous service;
compared to a “multiple plain Channels” scheme, Ursula gets:

* In a multiple plain Channels scheme, if one of the LSPs is
  down, the liquidity in the plain Channel with that LSP is
  unuseable while that LSP is offline.
  With a MultiChannel, all the liquidity in the single
  MultiChannel remains useable with the still-live LSPs.
  * With MultiPTLC, Ursula can give a MultiPTLC package to a
    quorum of the LSPs, and herself go offline, then when the
    dead LSP comes online, it can start participating in the
    “stuckless a.k.a. ‘parallel stuck is fine actually’” payment
    protocol seamlessly without Ursula having to come back online
    to authorize it.
* In a multiple plain Channels scheme, one UTXO is needed for
  each LSP that Ursula has, increasing onchain costs.
  With a MultiChannel, only a single UTXO is needed for the
  single MultiChannel to multiple LSPs.

The drawback of the MultiChannel is:

> * The LSPs Alice, Bob, and Carol ***do trust*** a quorum of their
>   partner LSPs for ***fund safety***.

Thus, while a single corporation might willingly run multiple
LSPs and allow a MultiChannel from end-users to their LSPs,
multiple corporations may be unwilling to risk their funds
to form MultiChannels with each other.

This is problematic because:

> * The user Ursula ***does trust*** a quorum of LSPs to provide them
>   with the ability to send and receive payments.

A single corporation running all the LSPS of Ursula can
trivially form a quorum of the LSPs of Ursula, and deny
Ursula the ability to send and receive payments.
Ursula can still bring their funds onchain (this is the
***funds safety*** part) but this is expensive compared
to Lightning, and can remain effective as a censorship
mechanism.

However, if we are able to reduce the trust requirements
of LSPs to each other, this may be enough for multiple
corporations to allow their LSPs to run on the same
MultiChannel, banking on reduced risk.
If the risks can be lowered, then it may become simply
“the cost of doing business” and acceptable to provide
better service to end-users.
If so, no single corporation can outright deny Ursula
service, and the other corporations can still provide
service to Ursula, reducing the censorship risk.

# Concepts

Before continuing with the actual construction, I will
present some concepts that are vital to the
constructions.

First, let us refresh the onlineness of our participants:

> * Ursula is ***usually offline***.
> * The LSPs are ***chronically online***.

Now let me present the concepts:

* ***Update*** is an operation that requires
  Ursula to be online, plus at least a quorum (in our
  example, 2 of 3) of Alice, Bob, and Carol.
  * Every payment (send or receive) is an update.
  * Failed payments are also updates.
  * As only a quorum of Alice, Bob, and Carol is
    neededd, it provides high availability to Ursula
    for her sends and receives.
  * Every update “dirties” the MultiChannel a little
    bit.
    With enough dirtiness, the MultiChannel becomes
    unuseable, but a Cleanup (described below) will
    remove all the dirtiness.
* ***Cleanup*** is an operation that requires *all*
  participants (Ursula, Alice, Bob, and Carol) to be
  online.
  It *has* to happen or else the channel slowly becomes
  unuseable.
  * However, it is only needed periodically, e.g. at
    most once a day, and can generally be run at
    any time.
  * Cleanup only matters if Ursula is actively doing
    updates to the MultiChannel state.
    If Ursula does some updates (which dirties the
    MultiChannel) while only a quorum of LSPs is
    available, then goes offline for weeks or months
    without sending or receiving, it is OK to leave
    the MultiChannel in a “dirty” state until Ursula
    happens to come online while all the LSPs
    are also online.
  * Very rarely (at most once or twice a year)
    Cleanup will create an onchain transaction,
    which is a simple one-input, one-output
    transaction.
    This is an ***Onchain Cleanup***, compared to
    normal Cleanups that happen completely offchain.
    * In case of an onchain splice (splice-in or
      splice-out) the Onchain Cleanup can be
      coalesced with the splice.
      So, if Ursula or the LSPs initiate any splice
      operation, this will also count as a Cleanup
      and also defer the need for a future Onchain
      Cleanup.

Presenting these concepts separately allows us to
consider their impacts to the UX first before we
proceed to the details of the channel.

While Cleanup requires a full consensus (i.e. all
LSPs plus Ursula) to come online, it is a rare
operation; if Ursula hardly ever uses the wallet,
then it is okay to not do Cleanups, too.
Cleanups really only become necessary if Ursula is
doing a lot of offchain receives and offchain sends
alternately.

An Onchain Cleanup is necessary after a few hundred
Cleanup operations.
So, if Cleanups are rare, an Onchain Cleanup is
even rarer.

Further, an Onchain Cleanup is also coalesced with
any splice operation.
An Onchain Cleanup also counts as a Cleanup, so
that also enables in practice some amount of receives
and sends.

For example, if Ursula receives their monthly salary
onchain, and then puts some of it into their Lightning
wallet via splice-in, then that is an Onchain Cleanup
as well.

Alternately, if Ursula mostly receives money into
their wallet, at some point the liquidity from the
LSPs to the MultiChannel depletes.
The LSPs already have strong evidence that Ursula
will tend to receive funds, so they can justify
splicing in more liquidity towards Ursula.
Again, that splice-in is also coalesced with an Onchain
Cleanup.

Thus, this construction would work fine for
mostly-send and mostly-receive users, where Cleanup
operations in the form of splices in and out of the
MultiChannel would be enough to prevent the buildup
of Update dirt.
It is somewhat more problematic for users that have a
lot of activity that is mostly balanced between sends
and receives, especially when alternating between them.

Nevertheless, for many users, it would seamlessly
allow them to make payments ***right now***, even if
some less-than-quorum number of LSPs are down.
Cleanups do require the consensus set of LSPs, however,
we expect that most of the time, all the LSPs are
chronically online anyway, and the Cleanup operations
can be done at almost any time when Ursula happens to
be online to spend or to check a received payment.

# The Construction

I now present the jxPCSnmZ-Decker-Wattenhofer Complex
Micropayment MultiChannel construction, a novel nested
construction that achieves the characteristics of the
original MultiChannel concept, while providing
improved trust-reduction for the LSPs.

* One or more nested layers of decrementing-`nSequence`
  constructions.
  * Signers are all n-of-n participants.
* An innermost layer composed of multiple `nSequence`-variant
  Spilman unidirectional channels (the “complex”), one for
  each participant.
  * The Ursula Spilman channel is slightly different: it is
    from Ursula, to a k-of-n of the LSPs.
    * The signer is a nested 2-of-2, where one side is Ursula
      alone, and the other side is a k-of-n of the LSPs.
  * The LSP Spilman channels are standard two-party
    unidirectional channels, from LSP to Ursula.

Immediately, the improved trust-reduction is apparent:

* The LSP Spilman channels follow ***NOT YOUR KEYS, NOT YOUR
  COINS***: the construction is an n-of-n with the LSP and
  Ursula, and every outer construction in the nested
  construction are *also* n-of-n, which includes the LSP and
  Ursula.
  Thus, to spend from it, requires **YOUR KEYS**, thus the
  coins inside it are **YOUR COINS**.
  * Thus, money held by the LSP and Ursula on the LSP Spilman
    channels is completely safe (assuming adequate onchain
    monitoring after a unilateral close).
* The only thing with a degraded, trust-requiring k-of-n is
  the Ursula Spilman channel.
  * Thus, money that the LSPs have already received via this
    channel is at risk, and the LSPs trust a quorum of its
    partner LSPs to not steal those funds by publishing old
    state or cooperating with Ursula.
  * Ursula is a single signer on the 2-of-2 signing layer,
    thus has no trust requirement for its own funds here.

The trust requirement for ***fund safety*** is thus reduced
for LSPs towards their partner LSPs (Ursula has no trust
requirement for ***fund safety*** at all, as emphasized
earlier).

## On Spilman Unidirectionality

Technically speaking, if an HTLC, PTLC, or MultiPTLC is
failed offchain in a Spilman channel, this *returns* funds
to the sender-side of the Spilman channel.
This violates the unidirectionality assumption of Spilman.

As a concrete example, suppose an LSP forwards an HTLC to
Ursula via a Spilman channel, from the public network.
This is a new state where some funds are now in an HTLC.
However, Ursula says it does not know the preimage anyway,
so they sign a new state where the funds are returned to
the LSP, creating a new state without the HTLC.
Then the LSP propagates the failure back to the public
network, failing its incoming HTLC.

Unfortunately for the LSP, “Ursula” was actually the
sockpuppet of notorious hacker ZmnSCPxj.
ZmnSCPxj was the one who actually sent the HTLC from the
public network, and made “Ursula” fail it.
So when the HTLC failure has propagated back to ZmnSCPxj,
“Ursula” unilaterally closes the Spilman channel using the
old state where the Spilman channel still has the HTLC,
and takes the H branch to get the funds.
Since the incoming HTLC on the LSP side is already
irrevocably committed as having been removed, the LSP cannot
reclaim funds from the incoming channel with the now-revealed
preimage.

Thus, in general, HTLC, PTLC, and MultiPTLC failures
***cannot*** be performed offchain in a Spilman channel.
Once the funds from the sender-side of the Spilman channel
have been put into an HTLC or whatever, it either has to be
claimed by the receiver-side of the Spilman channel, or left
as an HTLC or whatever until the Spilman channel state
becomes immaterial.

This is actually one of the kinds of “dirtiness” in this
mechanism.
If an offerred HTLC, PTLC, or MultiPTLC in a Spilman
channel is *supposed to be* failed, it has to remain as
“definitely still an HTLC / PTLC / MultiPTLC” in the state
of the channel going forward.
Obviously, that fund cannot be reused for a different
HTLC, PTLC, or MultiPTLC, so if there are enough of this
kind of dirtiness “built up”, we need to somehow reset
the Spilman channel state.

Thus, a “Cleanup” is needed, as was discussed earlier, to
get rid of this kind, as well as other kinds, of dirtiness.

### LSP to Ursula Payment Hop

Due to the HTLC (et al) having to remain in the Spilman
state if it would fail, it means that payment faillure
*cannot* be propagated back until a Cleanup.

We can ameliorate this issue by doing a “pre-check”.
When the LSP receives an HTLC (et al) from the public network
to Ursula, instead of immediately fast-forwarding it, it
first does a 1.0-roundtrip query with the hash (or point for
PTLCs) to Ursula.

Then, only if Ursula says it can claim that fund, will
the LSP actually provide the new state that forwards the
HTLC (et al) to Ursula.
If instead, Ursula says “I have never seen this hash in my
life” it provides an payment failure return onion, and the
LSP can fail its own incoming HTLC (et al) safely, as there
is no outgoing HTLC (et al) to Ursula yet.

Of course, if Ursula says “yeah I know it, send it on” and
then the LSP sends the new state, Ursula could just fail it
anyway, and the LSP has to hold the corresponding incoming
HTLC (et al) until a Cleanup can be done to remove the HTLC
dirt.
However, this jamming is covered in our trust assumptions:

> * The LSPs Alice, Bob, and Carol ***do trust*** Ursula will not jam
>   their public channels.

### Ursula to LSP Payment Hop

Since Ursula is not a forwarder, we can leave a failed HTLC,
PTLC, or MultiPTLC on its Spilman channel indefinitely.
It adds dirt, but all that dirt can be cleared by a later
Cleanup operation.

(***IMPORTANT*** It is *stil* unsafe to just return the HTLC
(et al) to Ursula if the LSPs claim it is failed.
Everything is a forwarder: if I am buying something from a
store, I am getting an item in exchange for funds, and thus I
am a forwarder that accepts real-world items to forward Bitcoin
over Lightning.
More concretely: suppose Ursula goes to a store, and then
attempts to pay with Lightning, which creates an HTLC (et al)
on the MultiChannel.
However, the LSP claims the payment failed, so, disappointed,
Ursula leaves the store itemless, and goes on a flight to El
Salvador, which has better Bitcoin support.
On landing, the Ursula wallet gets Internet and sees that the
LSP dropped old state onchain, and claimed the HTLC and
revealed the preimage.
What would the wallet do, say “actually, remember that payment
I said was definitely 100% failed?
ummmm actually it went through, happy now?”
Unfortunately, Ursula is already in El Salvador, very far
away from the store.
Thus, it is ***still*** unsafe in general to return the
HTLC (et al) on failure; it ***has to*** remain in the Spilman
channel state indefinitely, as dirt that is removed on the
Cleanup.)

Just use a MultiPTLC if you are paying a PTLC-based invoice.

The big advantage of the MultiPTLC is that Ursula can offer
just a ***single*** MultiPTLC, and the LSPs can then operate
the “stuckless because that is the name we are stuck with”
payment protocol, in parallel, backed by that single MultiPTLC.
As long as at least one of the attempts succeed, then the
MultiPTLC *will* succeed and we are still properly
unidirectional.

With only PTLCs, Ursula has to either decide to offer just
one PTLC to one of the LSPs, which has low availability (if
all paths through that LSP fail, then that offerred PTLC has
to fail and remains as dirt on the MultiPTLC), or offer
multiple PTLCs, with the knowledge that at most one of those
PTLCs will succeed and the other PTLCs will be dirt on the
Spilman channel.
Thus, MultiPTLC is superior.

Of course, we still need to work with HTLCs for a while.
For this case, Ursula can do the inverse of the “pre-check”
used in the LSP-to-Ursula case: it asks the LSPs to probe
the network to reach the receiver.
The LSPs can even do this probing in parallel.

Of course, that means that LSPs have to send out HTLCs on
behalf of Ursula while Ursula has no HTLCs to the LSP,
which is risky on the LSP.
We can thus use a “provable probe attempt” scheme by ajtowns
long ago (do not remember the link, sorry).

In a provable probe, you generate a fake hash for your HTLC
as below:

* Take a random 256-bit number, `proof_of_probe`.
* Get `real_hash = sha256(proof_of_probe)`.
* Use the hash `fake_hash = real_hash ^ 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF`.

What we do is, Ursula creates multiple possible onion
paths, with `fake_hash`es generated above.
Ursula then asks the LSPs to probe using HTLCs with those
`fake_hash`es along those onion paths, and provides the
`proof_of_probe`s for each one.
The LSPs thus know that the HTLC they are being asked to
send out is unclaimable unless SHA256 has been broken.

The HTLCs sent out are definitely going to fail, and the
LSPs return the payment failures back to Ursula, who can
decrypt them and see which ones reached the receiver.
These probe attempts can even be safely sent out in
parallel.
Then Ursula can select one of the successful payment paths,
or if all of them fail before the receiver, can give up
without putting HTLC dirt on the MultiChannel state.

(A nice benefit of this is that even if a remote node goes
down while holding an HTLC lock, if it happens during this
probing stage, Ursula is unaffected as she has no HTLCs
locked in the HTLC-lock-chain.
We should probably consider implementing this on the
existing Lightning Network right now, even with just
low-availability Channels.)

With MultiPTLC, pre-probing can still be done (the proof
of probe is simpler in that the `sha256(proof_of_probe)` is
just the X coordinate of the point), but Ursula can provide
*all* successful paths in the MultiPTLC instead of just
selecting one as with HTLC, and is protected agaisnt remote
nodes going down while holding a PTLC (with HTLCs, there
is still a tiny chance of failure (because the earlier
failure used in the probe also unlocked the HTLC lock-chain,
this is a double-checked-locking antipattern), and a tinier
chance that a node on the selected path (which was OK just
several hundred milliseconds ago?) will go down).

***IMPORTANT***: Regardless of whether we are using
MultiPTLC or HTLC, the wallet MUST NOT show the payment as
failed until a Cleanup has occurred to actually remove the
underlying MultiPTLC or HTLC; it should still show it as
“pending”, because technically it can still be claimed
until timeout.

## The Ursula Spilman Channel

As mentioned above, the Ursula Spilman channel is unusual
in that one of the 2-of-2 is actually a k-of-n amongst
the LSPs.

Largely, this changes the trust model for the LSPs only;
in effect, any funds that an LSP has claimed from that channel
from successfully forwarded HTLCs or MultiPTLCs, are delegated
to a quorum of the LSPs.
As noted, this is part of the MultiChannel degraded security
and is a deliberate tradeoff.

Otherwise, there is no effect on Ursula.
That it is on an n-of-n signing layer means it has the same
trust assumptions as normal Spilman.

I would like to emphasize that all existing LN implementations
(should) sign unilateral closes only when they actually want
to unilaterally close.
Otherwise, they just store the signature coming from the other
side.

Similarly, when Ursula updates the state of the Ursula
Spilman channel (to offer an HTLC or MultiPTLC, or to finalize a
claimed HTLC or MultiPTLC), it sends a signature for its share
of the 2-of-2, but the quorum of LSPs ***MUST NOT*** create a
signature for the other part of the 2-of-2.
Instead, they should defer signing their side until either a
new state is received (and they then never ever sign the old
state) or when a quorum agrees to unilaterally close the
construction.

## Cleanup and Onchain Cleanup

What I call a Cleanup is simply an update in the nested
decrementing-`nSequence`.
See the original Decker-Wattenhofer for details.

Funds in the Ursula Spilman channel that are now owned by
one of the LSPs (which come from successful claims of HTLCs
or MultiPTLCs) ***cannot*** be used as liquidity from LSP to
Ursula.
Again, the Spilman channel is unidirectional, so there is
no way to safely return funds to Ursula.
This is another example of “dirt” that builds up on the
MultiChannel that needs to be cleaned up.

Similarly, funds in any of the LSP Spilman channels that
have been claimed by Ursula cannot be sent out by Ursula
until a Cleanup.

However, we can still have some time where Cleanup is
absolutely necessary.
For as long as there is still plain liquidity from the source
of each Spilman channel, the participants can continue making
Updates (adding new HTLCs, PTLCs, or MultiPTLCs) with a
dirty MultiChannel and defer Cleanup.
Thus, we expect Cleanup to be reasonably rare, though
continuous multidirectional send/receive/send/receive
sequences increase the dirt and the need to Cleanup.

When a Cleanup is done, the consensus adds up all the funds
that are definitely owned by each participant, and assigns
those as the initial state of each of the Spilman channels.
Any pending HTLC, PTLC, and MultiPTLC that is neither claimed
nor failed is retained in the Spilman channel they were in
before Cleanup (ideally, Cleanup would not occur unless all
HTLCs, PTLCs, and MultiPTLCs have been claimed or failed,
but the Cleanup code should definitely handle the case where
Cleanup occurs while locks are still held).
They then agree on what the next state on the
decrementing-`nSequence` nested mechanism looks like, and
sign the necessary transactions to move to the new state
and to start the recreated Spilman channels.

Now, the decrementing-`nSequence` mechanism is effectively a
counter, and at some point, that counter drops to zero, and
no more state changes can be done on the
decrementing-`nSequence` layer.
When that happens, the participants absolutely need to do an
Onchain Cleanup, which not only resets the Spilman complex
layer, but also resets the decrementing-`nSequence` layer
counter.

Presumably, the LSPs will consider channel forwarding fees
with the knowledge that Onchain Cleanup will occur at
some point, and adjust the forwarding fee rates accordingly.
Thus, these Onchain Cleanups should be paid for by the LSPs
(with the understanding that they will extract this from
Ursula via channel forwarding fees, and if Ursula is truly
inactive, just unilaterally close the construction).

Splicing will require an onchain reseat of the backing UTXO
anyway, so we may as well also reset the counter on the
decrementing-`nSequence` layer when that happens, thus
“coalescing” an Onchain Cleanup with a splice.
The cost of the splice can be billed to whoever initiates
the splice.

Splicing in and out, as well as Onchain Cleanup and even
just plain Cleanup, requires all participants to be online.
This allows the trust-reduction that this MultiChannel
implementation allows.

A Cleanup is desirable for the LSPs if Ursula has
successfully sent funds.
In that condition, the Ursula Spilman channel has funds
of one or more of the LSPs, which is at degraded
security for the LSPs due to being protected by a k-of-n
instead of a strict requirement for the owning LSP.
A completed Cleanup will move those funds into the
per-LSP Spilman channels, with the standard online
Lightning security model.

-------------------------

ZmnSCPxj | 2025-09-18 13:21:56 UTC | #2

[quote="ZmnSCPxj, post:1, topic:1994"]
* The user Ursula ***does trust*** a quorum of LSPs to not lock the Ursula funds for two weeks (or so) due to unilateral exit when Ursula is willing to cooperatively exit.

[/quote]

Whoops!

Looks like this is lost, due to the symmetric nature of Decker-Wattenhofer.  With the scheme proposed in this writeup as-is, every participant, including the LSPs individually, get a signed copy of the kickoff transaction, and can unilaterally exit without permission from any of the other members.

As noted, the goal of this construction is to reduce censorship risk by reducing funds loss risk for LSPs.  The original Poon-Dryja-based MultiChannel construction put significant funds loss risk for the LSPs, meaning that the most likely setup was a single corporation setting up multiple LSPs and only allowing MultiChannels for the LSPs it set up.  This puts Ursula at significant risk of being censored if that single corporation is pressured to censor her.

If any single LSP can unilaterally exit the MultiChannel anyway, then the same censorship risk is still present.

To ameliorate this, I propose returning the asymmetric nature of Poon-Dryja.

Instead of a single chain of transactions funding→kickoff→decrementing-`nSequence`→…→decrementing-`nSequence`→Spilman complex, we have two chains:

* Ursula-side chain: funding→Ursula kickoff→decrementing-`nSequence`→…→decrementing-`nSequence`→Spilman complex
* LSPs-side chain: funding→LSPs kickoff→decrementing-`nSequence`→…→decrementing-`nSequence`→Spilman complex

The Ursula kickoff is signed as an n-of-n of Ursula and the LSPs Alice, Bob, and Carol.  All the subsequent transactions are also signed as n-of-n of all participants, with the Spilman Complex dividing out the funds into the Spilman channels with varying signing protocols as discussed in the main text.

The LSP-side kickoff is signed with two sets of signatures:

* Funds safety set: n-of-n of Ursula and the LSPs Alice, Bob, and Carol
* Unilateral exit set: k-of-n of the LSPs Alice, Bob, and Carol

Subsequent transactions in the LSPs-side chain are all signed as n-of-n of all participants, with the Spilman Complex yada yada.

For the kickoff transactions ***only***, the LSPs provide signatures for the Ursula kickoff, while Ursula provides its signature for the LSPs kickoff.  For all other dependent transactions, all participants have to share full sets of signatures at setup time and at each Cleanup/Onchain Cleanup (Spilman channels are great in that they do not need pre-signed transactions now since `OP_CHECKLOCKTIMEVERIFY` and `OP_CHECKSEQUENCEVERIFY` activation…. I miss the good old days when we could still add opcodes.  I sometimes hallucinate that people are still working on covenant opcodes, even).

-------------------------

ZmnSCPxj | 2025-09-18 14:45:54 UTC | #3

“Everybody say ‘fees’!” - some joker in Adelaide, 2017

Unilateral exit has to pay onchain fees.  This is a rote fact.

Nowadays we have P2A, and it is awesome that instagibbs managed to get that in before the local mempool management heuristics ossified.  In the bad old days Lightning Network participants had to agree on the same onchain feerates, and caused even more expensive channel closures if they disagreed.

With P2A, however, offchain constructions have to have a separate UTXO — the “exogenous fee UTXO” — in order to pay for their unilateral exit in the future.  This is unfortunate:

* Liquidity in the exogenous fee UTXO cannot be spent in Lightning, and spending it onchain risks your future ability to unilaterally exit.
* It is another UTXO, increasing onchain resource consumption.

There is, however, a curious wrinkle: the exogenous fee UTXO has to exist ***per participant,*** not per construction.  A participant with one construction, or millions, can maintain just one exogenous fee UTXO (in practice participants with millions of channels have to maintain more than one, but the point still stands).

With millions of MultiChannels, there would be millions of Ursulae.  But there are still only three LSPs, Alice, Bob, and Carol.  So what we want to do is ***have unilateral exit be paid by LSPs*** (who will indirectly extract those fees from the Ursulae via routing fees, but because they aggregate the requirement for maintaining exogenous fee UTXO into a smaller number, they can get economies of scale, and pass on the savings to the Ursulae).  With this, onchain resource consumption is reduced and the Ursulae do not have to have extra liquidity they cannot spend.

We can enforce this by returning revocation to the construction.

For the decrementing-`nSequence`, the consumed input script (from the kickoff or from a previous decrementing-`nSequence`) has to have two branches:

* n-of-n of all participants.
* CSV(max-`nSequence`+1 hour) && Ursula

At the initial condition after setup or Onchain Cleanup, the decrementing-`nSequence` transactions all have the maximum `nSequence` value they start with (called “max-`nSequence` above).  By adding the alternate branch above, if the decrementing-`nSequence` transaction has not been confirmed after an hour or so after the `nSequence` makes it valid, Ursula can simply get all the funds.  Ursula can create a punishment transaction that pays for its own onchain fees here.  This forces the LSPs to pay for the decrementing-`nSequence` transaction within an hour.

Now, to update the decrementing-`nSequence` to a later version, we have to import the “sign, then revoke” dance from Poon-Dryja.

* First, everyone signs the next decrementing-`nSequence` state, with an `nSequence` that is one hour less than the current state (sign step).
* Then, the LSPs sign for every output of the current decrementing-`nSequence` state, signing a 0-fee transaction that spends that output and gives it outright to Ursula (revoke step).

After the revoke step, the new state is now valid and the participants can proceed after the Cleanup operation.

For intermediate decrementing-`nSequence` transactions, they only have one output, which is the “n-of-n || (CSV && Ursula)”, and signing the revocation transaction is trivial.  For the final decrementing-`nSequence` transaction that instantiates the Spilman Complex, well, Spilman channels are all already “n-of-n | (CSV && participant)” and can be revoked with the n-of-n branch.

The complication here is with the Ursula-sourced Spilman channel.  The transaction that revokes this channel is an alternate transaction from the Spilman-exit transactions that Ursula has been giving to the LSPs, and the LSPs can make old state appear and pay for that to be confirmed.  To protect against this, the Spilman-exit transactions that Ursula gives to the LSPs have to have an `nSequence` that encodes 1 hour delay (the “CSV && participant” branch then needs to have the CSV be at least 2 hours).  Then, a revocation of the Ursula-sourced Spilman channel has an `nSequence` of 0, allowing Ursula to spend it immediately if the old decrementing-`nSequence` state is published by the LSPs instead of the latest one.

As noted, revocation is needed so that the latest decrementing-`nSequence` is confirmed by the LSPs in a timely manner; if they delay, they run the risk that Ursula is actually notorious hacker ZmnSCPxj, who then uses the not-latest, revoked, decrementing-`nSequence` to get all the funds from the MultiChannel.  Thus, the LSPs are forced to confirm the latest decrementing-`nSequence`.

So much for revocation.

Let us now turn to the Spilman Complex and how to force the LSPs to publish the latest state.  For the Ursula-sourced Spilman channel, this is already handled: if the LSPs do not publish ***any*** state, Ursula gets the funds in it, due to the “CSV && Ursula” branch that Spilman has as part of itself.  If the LSPs publish old state, well, that has more funds to Ursula than the latest state, because of unidirectionality.

The issue is with the LSP-sourced Spilman channels.  Unfortunately, those have to have endogenous fees, meaning the LSP has to keep a reserve fund (fortunately, Ursula does not need a reserve fund).  The Spilman-exit transactions given by the LSP to Ursula then spend that reserve fund as fees, to allow Ursula to get it confirmed. (I actually considered another revocation scheme here, but ran out of brainspace and started thrashing swap so I aborted the processing, falling back on endogenous fees).  The feerate for this can be fixed by protocol, so that it always propagates, and also have a P2A so that the LSP can, out of the goodness of its heart, get that confirmed.  Finally, any state other than the first has an output that Ursula can claim outright (either an HTLC (et al) where Ursula knows the preimage, or a plain Ursula-only output).  If that output is significant, Ursula can pay fees from that output directly; if the output is insignificant, Ursula would probably lose it in onchain fees anyway and can just tombstone it with an `OP_RETURN`.

-------------------------

ZmnSCPxj | 2025-09-24 11:48:53 UTC | #4

I should note that fundamentally speaking, the construction shown in this post ***is*** in fact a Burchert-Decker-Wattenhofer channel factory, with the hosted channels being unidirectional Spilman channels.

However, there are important facts:

* There are only N channels hosted in the factory, not the (N \* (N - 1))/2 potential maximum a “true” factory can host.
  * The goal is availability for Ursula.  An “extra” channel between Alice and Bob is useless in terms of availability for Ursula since that channel would be down if ***either*** of Alice or Bob is down; what Ursula wants is to be able to increase its chances of routing, and that does not really help since if Alice is up, Ursula can just hop straight to Alice instead of using Ursula→Bob→Alice (and vice versa).  If Alice is down, then the Bob→Alice channel is unuseable anyway.  Far more efficient liquidity-wise to ***not*** have an Alice-Bob channel and just put the liquidity that would have been placed there to point at Ursula.
* The channel sourced from Ursula has the receiving end be a k-of-n of the LSPs.  You can plausibly argue that it is ***this*** channel that is the true core MultiChannel, and the decrementing-`nSequence` layers are actually a channel factory, and the other channels sourced from the LSPs are “only” Spilman 2-party Channels, and that would be an acceptable point-of-view.
  * The point here is that the MultiChannel is a ***single*** pool of funds that Ursula can use to send to ***any***  of the LSPs, a one-to-many mapping, whereas a “true” channel factory allows Ursula to have ***multiple*** pools of funds to ***multiple*** LSPs, backed by a single onchain UTXO, but ultimately ***only*** a one-to-one mapping.  The innovation here is the possibility of a one-to-many mapping; that is the true innovation of MultiChannel and MultiPTLC.

-------------------------

