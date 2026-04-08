# Towards A K-of-N Lightning Network Node

ZmnSCPxj | 2026-04-08 09:05:39 UTC | #1

Subject: Towards A K-of-N Lightning Network Node

Z. jxPCSnmZ

Introduction
============

With the [Nested MuSig2 Paper](https://eprint.iacr.org/2026/223)
recently released, we want to consider: can we make a
multisig self-custodial wallet with Lightning Network
integration?

In particular, our hope is that with Simple Taproot Channels
on [PR995](https://github.com/lightning/bolts/pull/995), we
can use Nested MuSig2 to seamlessly have multisig Lightning
Network nodes.

But first:
What does it mean for an end-user, when they say, "I want a
k-of-n multisig wallet"?

I submit that the central desire, for the end-user, is the
following:

> By "k-of-n multisig wallet", I mean, I possess N devices,
> and if lesss than K of them are compromised, my funds are
> SAFU.
> If I irretrievably lose less than N - K + 1 of them,
> I can still spend my funds.

How should we translate the above into a "Lightning Network
wallet", in practice?

While an onchain wallet can trivially express the k-of-n
requirement directly onchain, Lightning Network wallets
cannot.
Lightning Network channels have additional complications.
These additional complications ***require a change in the
Lightning Network BOLT protocol***, not only for the k-of-n
Lightning Network wallet, but also for its channel peers
that themselves may or may not be k-of-n.

This implies that an end-user that wants to use a k-of-n
wallet must:

* Wait for mass adoption of the modified Lightning Network
  BOLT protocol, OR
* Accept services from gateway nodes that implement the
  modified Lightning Network BOLT protoocl "early", with
  the loss of privacy and reduced forwarding fees that
  implies.

A major use-case for k-of-n Lightning Network nodes is for
large HODLers to be able to use their HODLings to provide
liquidity to the network, while ensuring that the funds are
SAFU and recoverable.

However, until the modified Lightning Network BOLT protocol
is massively adopted by the network, such large k-of-n
multisig nodes can only connect to gateways that bridge
between the modified protocol and the unupgraded network;
these gateway nodes will, understandably, charge a premium
for this, by hook or by crook.
Any gateway node that does ***not*** charge a premium will
find their available liquidity used up quickly by such
large HODLers, and they cannot have large amounts of
liquidity without themselves being large HODLers, who (by
our own assumptions) will want to use a multisignature
setup themselves, and thus need further gateways that
also support the modified protocol.

Thus, for the purposes of such use-cases, the protocol
***must*** be modified "early" to drive adoption for the
modified protocol, before large HODLers can use their
HODLings to provide liquidity.

Trust-Minimized Channel Partners
--------------------------------

Some might quibble and say that "Lightning Network is
actually more secure because you also have to compromise
the channel counterparty in addition to losing your own
key".

The above thought is oh so very wrong.

The correct thought is "the channel counterparty is in
the best position to rob you so you should be extra
wary about your channel counterparty".

In particular, consider this: if you are a large HODLer
who wants to provide your liquidity to the Lightning
Network in exchange for fees, you want ***anyone*** to
be able to open channels for you.
Others opening channels to you is more opportunities to
earn routing fees!

If you do ***not*** allow complete Internet strangers
to open channels to your large routing node, and want
to curate some limited number of trustworthy entities
allowed to channel with you, then I assure you:

> By hook or by crook, by overt means or covert, your
> curated "trustworthy" counterparties will extract the
> fees you earn as rent for their liquidity, or else you
> will find your liquidity underutilized and
> underperforming.

The fact that you ***must*** allow complete Internet
strangers, like YaiJBOjA (who is totally not me, just
to be clear), to open channels with you, means that
***you are allowing possible thieves to become
channel counterparties***.
You are allowing others to paint a target on you.

Thus, you ***must*** assume that only your own local
channel signer(s) are trustworthy, and dispense with
curation of counterparties.
Your security armor must be strong enough, even with a
target painted on your back, to repel thieves, without
ever invoking your channel counterparty, as the
channel counterparty is ***precisely the best thief***.

Your counterparty can channel with you using only their
funds, then send out those funds through your node to
another node they control, then somehow steal the funds
in the channel (which, after they send out those funds,
are semantically yours), if they are able to compromise
enough of your signers to do so, or to exploit weaknesses
in your multisig scheme.
Thus, they can steal that amount from you.

Your Fork Of Morton is thus:

* You allow Internet randos to potentially steal from
  you if you are not careful about your signer security.
* You do not allow Internet randos and thus your liquidity
  is not utilized by the network (and you earn no fees,
  you are just putting your funds in an online hot wallet
  for no benefit to you or to anyone).

Thus, the requirement to have better security postures
than just the simple "I have one root signer for all my
Lightning Network channels".

Multisig Is An Extra Dimension
------------------------------

You might say "but my signer is inside a 'Tr\*sted'
Execution Environment that is running on a 'S\*cure'
Element that is implemented with a Physically
Unclonable Function!!!!"

First and foremeost a Physically Unclonable Function
is just a secure dice roll for an embedded private
key.
It does nothing if you have access to debug interfaces
on the 'S\*cure' Element, which ***every*** chip
manufacturer ***must*** havem because chip manufacturing
is ***not a perfect process***, there will be
***defects***.
The same debugging interfaces that detect defects can
be used to extract intermediate calculation results
using the embedded private key;
the intermediate calculation ***circuit*** needs to
be checked that it was manufactured correctly, too,
after all.
You can't have a single calculates-everything circuit
in practice, you need flip-flops to hold intermediate
results (otherwise you would need a ***humongous***
and therefore ***expensive and slow*** circuit,
intermediate results stored in flip-flops is
preciseely what lets the same generic circuit, such
as a multiplier or adder, to be reused for different
parts of the calculation) and it is precisely those
intermediate-result flip-flops that are used as part
of the debugging interface (the "scan chain", look it
up) at the manufacturer to check for defects on
manufactured chips.

It is precisely those flip-flops that store
intermediate calculation results, and those flip-flops
***cannot*** be in the Physically Unclonable Function,
you ***want*** those flip-flops and the circuitry
writing into them to be Physically The Same As How
You Designed Them or else you have an unreliable
chip.
Physically Unclonable Functions are just parts of the
chip where you deliberately increase the defect
rate to make it practically impossible to clone!

Those intermediate calculation results can then be
used to reduce the number of bits you have to guess
for the embedded private key.

Also remember that the manufacturer has access to
the aforementioned manufacturer debugging interface
/ scan chain or else the word "manufacturer" would
not be in that term.
Did you really believe the manufacturer
does not have the ability to just read out the
embedded private key in the Physically Unclonable
Function you are so in awe of?
You are better of with a Trezor without a
'S\*cure' Element, because the private key you put
into it is something you can literally generate
with coin flips and dice rolls, using coins, dice,
and other hardware you actually control, and
whose operation you actually know and understand.

'Tr\*sted' Execution Environments are just
"protection from your little sister", ***not***
"protection from major world governments".

(source: I used to design LCD display driver ASICs.
Before that, I used to reverse-engineer ASICs, we
literally ground away each metal layer and took
pictures via microscope to figure out the circuit.
Did you know that the big chip manufacturers in
Taiwan only accept your digital logic circuit if
you used one of the Big Three Verilog simulators
(Mentor, Synaptic, forgot the third one, we were a
Synaptic shop if you are curious) and will ignore
your manufacturing request if you don't have sim
results from one of the Big Three?
They do ***not*** trust Icarus Verilog at all,
they will not even reply if you mention it.)

Thus, possession of the hardware ***always*** equates
to possession of the private key, and no amount of
verbiage about 'Tr\*sted' Execution Environments,
'S\*cure' Elements, or Physically Unclonable Functions
changes "possession is 9/10 of the law".
Not your hardware, not your keys.

And regardless: even if you have a 'p\*rfect' circuit,
such as a simple Trezor built using off-the-shelf
parts you can order literally **anywhere** and not
trigger anything unusual because they are ***so
commonplace***, nobody will bother to replace your
supply chain because there are ***too many supply
chains*** for boring old microcontrollers, you can
still do better with a multisig scheme where multiple
such circuits must be stolen by thieves in order to
steal your funds.

Multisig adds more dimensions for your setup,
wheter it is a "s\*cure" setup based on the
'Tr\*sted' Execution Environment provided by your
vendor who promises not to steal your keys, or an
actual device you have physical possession of.

Multisig, as indicated in the first part of the
introduction, means "k devices must be compromied
to make my funds unSAFU", ***if*** we can achieve
it.

Not Quite Multisig Alternative
==============================

There are other ways to get some of the benefits of
multisig without requiring widespread changes to the
Lightning Network.

For instance, each Lightning Network channel has a
separate pair of public keys, one from each endpoint.

The protocol itself does not specify any specific
derivation scheme from some "root" key, or from the
node ID, to the per-channel public key you indicate.

It is thus possible to have N signers, then whenever a
channel is opened, to select one of those signers to
specify their own per-channel public key.
As nothing relates the public key you use for your
channel to your node ID, you are free to select any
signer for each channel.

This is compatible with the Lightning Network as-is
without changes to the protocol, and without even
waiting for PR995 to merge.

If you have N signers with this scheme, having one of
the signers compromised risks only 1 of N of your
channels to thieves.
This is still an improvement over having only one
signer, whose compromise risks losing ***all*** your
channels.

See [rotating-signer-provider](https://github.com/ZmnSCPxj-jr/rotating-signer-provider)
for an example of this scheme.

This may be a practical deployment for now, as
at least it spreads risk.
This would be similar to using multiple nodes,
except this would let you make one big node,
just manage the funds using multiple separate
single signer devices.
Using one big node will generally be better
for your ability to be on routes efficiently
and be a preferred router for much of the
network, increasing your fee earnings.

The BOLT Change: Revocation Keys
================================

The big BOLT change that is absolutely necessary for
a true k-of-n multisig Lightning Node is
the removal of the shachain requirement from the
remote side per-commitment revocation key.

Why is this necessary?

Recall what the ***actual*** onchain contract is,
for the funds you control on ***your*** commitment
transaction:

* Either one of:
  * Both of:
    * The commitment transaction has been deeply
      confirmed (usually two weeks).
    * Your signature.
  * Both of:
    * Counterparty signature.
    * Revocation key.

The second option above is the important part.

***Your*** commitment transaction is the only way
you can recover your funds.
For example, a thief can open a channel with you
using only their own funds, send out all the
funds to another node they control, then shut down
their thief node; at this point, they have nothing
to lose (other than the 1% reserve, but that is just
the cost of doing theft).
You ***must*** then use ***your*** commitment
transaction; if you do not, your funds are ***not***
SAFU, they are unspendable.

What Is Revocation?
-------------------

The various Lightning terms can be confusing:

* *Punishment* is what you do to your counterparty
  when your counterparty *unilaterally closes* to an
  old state.
  It requires that the counterparty broadcasts a
  commitment transaction and has it confirm, and that
  particular commitment has been *revoked*.
* *Unilateral closure* is when you broadcast a
  particular state, asserting that it is the latest
  state of the channel.
  It can be *punished* if you already *revoked*
  that state in the past (meaning it is not actually
  the latest state and thus your assertion onchain
  is wrong).
* *Revocation* is when you agree that an old state
  really is old.
  It gives enough information to your counterparty
  so that, if you *unilaterally close* to a state
  you already revoked, you can be *punished* later.

So ***when*** do these things happen?

* *Revocation*:
  - Occurs on each state change, i.e. whenever you
    add or remove an HTLC on the channel.
  - For example, you are at state number N.
    Both you and your counterparty agree to move
    to state N + 1.
    It proceeds this way in a hand-over-hand scheme:
    - Your counterpatry signs state N + 1.
      - At this point, both state N and state N + 1
        are valid and either can be used to
        unilaterally close without punishment.
      - In terms of "hand-over-hand", you are now
        holding both hands on the rope, with one
        hand higher up, as you ascend the rope
        i.e. advance the state.
    - You revoke the old state N.
      - This *irrevocably commits* you to the state
        N + 1, as that is now the only valid state.
      - In "hand-over-hand", this releases your
        lower hand via the revocation process.
  - At any time, there are at most two unrevoked
    states, and most of the time there is only one
    unrevoked state.
* *Unilateral Close*:
  - Occurs when you decide to sign any commitment
    transaction (adding your own signature to the
    counterparty signature you previously got during
    normal hand-over-hand state advancement) and
    broadcast it.
  - There is no hard cryptographic mechanism which
    prevents you from using an old commitment
    transaction signature, only an ecnomic
    incentive: if you use an old commitment
    transaction that you have already revoked, you
    can be punished and lose all funds in the
    channel.
* *Punishment*:
  - Occurs when you notice that your counterparty
    has done a unilateral close on an old, revoked
    state.

The issue is with revocation, not unilateral close
or punishment.

*Revocation* requires handing over something
called the *revocation key* to your counterparty.
Then the counterparty can use their own signature,
plus the revocation key.

The Issue With K-of-N Revocation
--------------------------------

Now, consider:

* Suppose you have a k-of-n multisignature setup.
* Now, consider: how do you generate the revocation
  key for each commitment transaction?
  * How many devices does your counterparty need
    to compromise, to steal the latest revocation
    key?

The big question is: ***how is the revocation key
created***?

The answer is: according to the BOLT specifications.
The BOLT specifications indicate: the shachain.

The problem is that, as the name implies, the shachain
uses iterated applications ("chain") of the
SHA2 function ("sha", thus "shachain").

It is not possible to create a k-of-n SHA2 function.
SHA2 is not linear in any way, thus it is simply
***impossible*** to create a k-of-n SHA2.
This holds even if k = n.

Even if it ***were*** possible to create a k-of-n
SHA2 function via some arcane cryptographic magic
involving virtual circuits where bits 0 and 1 are
encoded as homomorphic commitments, possession of
an ***intermediate*** shachain result (i.e. the
output of *one* SHA2 in the iteration) is just as
bad, because it is precisely the *intermediate*
results of the shachain iteration that *are* the
revocation keys.

Such virtual-circuit schemes work by using
iterations, such that intermediate results are
made known to more than one participant in the
virtual circuit;
these virtual-circuit schemes protect only the
root inputs without protecting intermediate
results.

The problem is that it is the ***intermediate
results*** of the shachain that ***are*** the
revocation keys!
It is immaterial that a virtual-circuit using
homomorphic encryptions of 0 and 1 protects the
root of the revocation keychain, when it must
reveal intermediate results of the iteration,
which *are* the sequence of revocation keys.
As a *sequence*, one of the sequence of
revocation keys is the revocation key for the
***latest*** state.

You would need to unroll the entire shachain
iteration and implement ***that*** as a
humongously big virtual circuit, in order to
protect even the intermediate results.
Worse, you need different unrolled virtual
circuits, for however many intermediate results
you must compute from the shachain, and that is 2
to the 48th power according to the BOLT spec.

So, virtual-circuit schemes of multiparty
computation are probably not feasible.

(warning: I am not a real cryptographer.
consult with a real cryptographer who actually
knows what homomorphisms are and how multiparty
computations work;
even so, I would not invoke multiparty computation
for the shachain, as I think it is unworkable
and I am not enough of a cryptographer to find a
way to make it workable)

Now, recall:

> By "k-of-n multisig wallet", I mean, I possess N devices,
> and if less than K of them are compromised, my funds are
> SAFU.

If the root of the shachain iteration is known by
***all*** signers, then *any* of them being compromised
--- that is, just ***one*** being compromised ---
lets the counterparty outright steal all channel
funds, and breaks the above expectation.

If the root is not known by all signers, but is
given as shares by k of them, then consider:
***what device*** then performs the repeated
shachain iteration in order to provide the
revocation key to the counterparty?
If that ***one*** device is compromised while
it has possession of the root key, or even
an intermediate result, then that is
***also*** equivalent to possession of the
latest revocation key, and thus allows theft
of funds.

Again, the point of k-of-n is that at least k
devices ***must*** be compromised.
Even if only one device has possession of the
revocation key, even temporarily, compromise of
***that one device*** allows theft.
Thus, we ***cannot*** ethically market this as
k-of-n multisig, as that is ***not*** what an
end-user will expect when we say "k-of-n".

Thus: shachain must be removed from the BOLT
specifications.

BOLT Modification Proposal
==========================

I propose adding a new pair of [feature](https://github.com/lightning-developer/lightning-rfc/blob/4f03a1a454ff0ba1c2cd63edc48923a4c5248dfe/09-features.md)
bits, named `no_more_shachains`, in both `globalfeatures`
and `localfeatures`:

* Odd bit: I will not perform the shachain validation
  on my counterparty, but will still provide
  shachain-valid revocation keys to my counterparty.
  - This implies back-compatibility to unupgraded
    nodes that still expect legacy shachain.
  - This implies that I will store all revocation
    keys, instead of using the shachain acceleration
    structure that compresses the available shachain
    to an O(1) space.
  - This lets non-shachain nodes make channels with
    me, while still letting me bridge them to
    shachain-requiring nodes.
* Even bit: I will not provide shachain-valid
  revocation keys and will not validate the
  counterparty revocation key for shachain-validity.

The `globalfeatures` bit allows k-of-n nodes to
identify other nodes they can safely make channels
with.
We should note that "creating a [BOLT8][] connection"
does not imply "creating a channel", and a k-of-n
node can create a BOLT8 connection, download the
[gossip][BOLT7] map, and check `globalfeatures` for
the `no_more_shachains`.

[BOLT7]: https://github.com/lightning-developer/lightning-rfc/blob/4f03a1a454ff0ba1c2cd63edc48923a4c5248dfe/07-routing-gossip.md
[BOLT8]: https://github.com/lightning-developer/lightning-rfc/blob/4f03a1a454ff0ba1c2cd63edc48923a4c5248dfe/08-transport.md

A k-of-n Lightning Network node would then enable
the even bit, while a gateway node would enable the
odd bit.
Eventually, we hope that all nodes will at least
enable odd bit and eventually we can just use
even bit for all nodes, even for 1-of-1 LN nodes.

The above respects "it's okay to be odd" where
legacy nodes can still interoperate with nodes
that enable the odd bit but not the even bit.
Nodes that enable the odd bit can also interoperate
with nodes that need to raise the even bit.
K-of-n nodes then need to raise the even bit,
but can interoperate with nodes that raise the
odd bit.

Note that enabling just the odd bit will
***triple*** the effective on-disk size of channels
with long history, though splices should allow you
to trim these.

The reason the space is tripled is:

* With anchor commitments, the only state changes
  are addition and removal of HTLCs.
  * Each HTLC is thus 2 state changes.
* Each historical HTLC is ~32 bytes of hash data.
* Each state change is an extra revocation key.
  * The shachain scheme only requires constant-size
    storage, but by dropping the shachain scheme,
    we now require linear storage.
  * Each revocation key is ~32 bytes of data
    (first as public key, but after revocation we
    can replace the public key with the private
    key).
* An HTLC will thus be 32 bytes for the hash, then
  32 bytes for the revocation of the state that
  *adds* the HTLC, and another 32 bytes for the
  revocation of the state that *removes* the HTLC,
  * Previously, with shachain, we only have the
    32 bytes for the hash itself, thus, this
    ***triples*** the on-disk size of channel
    history.

Non-issue: MuSig2 Nonce Knowledge
=================================

MuSig2 is a two-round protocol:

* First, exchange `R` contributions.
* Then, exchange partial `s`.

For Taproot channels specifically, the "exchange
partial `s`" really only has one side send over
their partial `s`; the other side then stores that
partial `s` on-disk.
If the other side decides to unilaterally close
using that state, it then generates its own
partial `s`, and "sends" it to the other side
by combining it and generating the final `R, s`
signature and publishhing it onchain.

Nested MuSig2 is largely the same two-round
process, with the nested signer doing each
round within themselves, combining their results,
and then completing them and performing the
exchange to the layer above them.

Due to having two rounds, PR995 specifies using
the current partial `s` exchange to send over the
`R` nonce contributions for the *next* signing
session.
This reduces the number of roundtrips in practice,
important for our colleague out in Australia with
all that latency down under.

K-of-N MuSig2?
--------------

While MuSig2 (and Nested MuSig2) are designed as
n-of-n signing protocols, I should note that
"k-of-n" FROST is really a Verifiable Shamir
Secret Sharing scheme that ultimately uses much
of the MuSig2 signing protocol for actual signing.
That is, instead of using the MuSig/MuSig2 key
combination function (distinct from the MuSig2
signing protocol), FROST uses its own multiparty
computation scheme to generate a set of "shards"
you have to store, one for each of your
co-signers, in addition to your own share of
the key, as well as a combined public key.

At signing time, FROST basically just uses something
very much like MuSig2
(and as such, mechanically it *should* be possible
to do FROST with Nested MuSig2 as well; no promises
on whether the proof for Nested MuSig2 extends out
to FROST-in-MuSig2, however!).

At signing time, you figure out *who* the
online signers are (the quorum of k
co-signers), then all the online signers
generate the first-round `R` contributions as
per the MuSig2 signing scheme, exchange them,
and then generate the second-round partial `s`.

The issue is that for the PR995 proposal, every
time we send out the second-round partial `s`
for ***this*** signing session, we *also* send out
the `R` contributions for the ***next*** signing
session.

So, for example, suppose:

* We have three signers, Alice, Bob, and Carol,
  in a 2-of-3 setup for our Lightning node.
* In the current signing session, Alice and Bob
  are online, and they thus generate the `R`
  contributions, process them, and send out the
  combination to the counterparty.
* Before the ***next*** signing session, however,
  Alice dies in a stray nuke from the inevitable
  robot uprising.
* Bob brings up his spare waifu, Carol.
* Carol does not know the nonce secret that Alice
  used, and thus cannot sign using the combined
  Alice+Bob nonce.

Fortunately, while the PR995 proposal specifies
that the counterparty will remember the nonce
contributions from the other side until the next
signing session, that nonce is dropped if the
BOLT8 connection is lost.
When re-connecting, a `channel_reestablish` is
sent, and can be sent with fresh nonces.

So, in the above case, Bob and Carol can continue
signing, despite the robot uprising, by simply
disconnecting from the counterparty and forcing a
new combined Bob+Carol nonce.

Thus, the nonce contribution round is not an issue
for k-of-n multisignature here.

-------------------------

