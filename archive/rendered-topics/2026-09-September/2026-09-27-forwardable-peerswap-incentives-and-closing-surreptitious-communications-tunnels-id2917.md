# Forwardable Peerswap Incentives, And Closing Surreptitious Communications Tunnels

ZmnSCPxj | 2026-09-27 19:11:54 UTC | #1

Subject: Forwardable Peerswap Incentives, And Closing Surreptitious Communications Tunnels

Introduction
============

Forwardable Peerswaps is a proposed protocol for managing Lightning Network
liquidity.

It has the following advantages over splicing:

* It provides a unique combination of ***privacy*** and ***incentive-compatibility***.
  - ***Incentive-compatibility***:
    - In splicing, service providers that charge for splicing have an incentive to
      deceive clients about their true available inbound liquidity.
      - For instance, if 10 clients buy 1.0 BTC of inbound liquidity via splicing to
        the same service provider, but that service provider only actually has
        2.0 BTC of inbound from the rest of the network, then the service provider
        is deceptively selling more inbound than it actually has.
        Once all the 10 clients have received a total of 2.0BTC, all the clients
        cannot receive ***any*** funds, despite thinking they have more because
        their local channel state is all they can see.
      - Any attempt to fix this would (1) require some kind of commitment to
        channel states, which is ever-changing at high speeds, and (2) tell the
        clients about how much money the service provider has, a loss of privacy.
      - Yes, splicing *does* let you buy inbound by your initiative.
        In contrast, in forwardable peerswaps, you pay in order to get outbound,
        and must opportunistically await a forwarded peerswap to arrive to you
        to get inbound liquidity.
        The problem is that splicing, as noted above, lets the inbound liquidity
        seller lie about its true available inbound liquidity.
        Why buy a product you cannot be sure you can trust?
    - In contrast, all remote swaps can only provide one side more inbound
      liquidity if-and-only-if the other side *actually has* inbound to take in
      the other side of the swap.
      - If the remote node that you are swapping onchain with *has* no *actual*
        inbound capacity reachable from your node, then the in-Lightning HTLC
        cannot reach them at all, forcing them to abort the protocol.
        This also means that you cannot pay fees (whether to them, or to miners
        to claim the onchain HTLC) for inbound liquidity that the other side is
        lying about; you atomically cannot buy a low-quality "inbound liquidity"
        product!
      - Thus, success of the swap is atomic to proof that the swap counterparty
        ***has*** inbound (outbound) capacity, and is atomic to the transfer of
        that inbound (outbound) capacity to you.
  - ***Privacy***:
    - As noted above, any attempt to show that the splice counterparty ***has***
      inbound capacity also requires leaking their channel state to the client.
    - Forwardable peerswaps are naturally forwarded using only local information.
      No participant in a forwardable peerswap ever has to look beyond their
      local channels, and thus, no participant in a forwardable peerswap ever
      needs to inform their local channel state to any other participant (because
      the other participants never need to learn more than their own local
      channel state).
      - This remains true no matter how many hops away; whether it is one hop
        or 20.
* Splice transactions are O(N) onchain blockspace on the number of channels
  modified, whereas forwarded peerswaps are O(1) onchain blockspace on the
  number of channels through which it is forwarded (= number of channels
  modified).
  - Even considering the constants, if you make the splice transaction batched
    (in a way no current node software ***can*** batch, and using Taproot
    channels as well), all it takes is ***two*** additional participant
    (i.e. going over three channels total) for forwardable peerswaps to beat
    the onchain transaction space of batched splices of three channels.
    - Yes, a splice transaction is one transaction, whereas in an swap, the
      onchain must have two transactions. one to create the HTLC, another to
      claim it.
      However, transaction inputs and outputs are heavy compared to the
      overhead (`nLockTime`, `nVersion`, num-outputs, num-inputs et al.) of a
      transaction.
    - It will take an even longer while for batched splicing to be implemented.
      The advantage of splicing is lost entirely if it cannot be batched.
      Whereas any onchain HTLC will always have just two transactions, one
      to create it, and one to reclaim it.
      We already have implementatiosn to create onchain HTLCs and claim them,
      because all Lightning Network node software have to handle HTLCs being
      dropped onchain when the channel is closed unilaterally.
* Splices are inherently local.
  Forwardable peerswaps being forwardable makes them non-local.
  - Splices are most easily batched in a "wide" manner, i.e. make a single
    batching splice that modifies two or more of your local channels.
  - Forwardable peerswaps naturally batch the modifications of channels in
    a "deep" manner, i.e. you modify your local channel, and the peer's
    local channel, and the peer's peer's local channel etc.
  - Both splices and forwardable peerswaps are intended to fix liquidity
    problems created by in-Lightning payments.
  - We know how to forward payments "deeply": just use an onion to forward
    the payment over multiple hops.
    - Thus, after several "deep" payments, we can expect that a forwardable
      peerswap will naturally route over the common prefix of the payments,
      and will improve the liquidity towards those nodes.
  - We are still debating on how to forward "widely", i.e. we still have not
    settled on how to best schedule and plan out multipath payments (i.e. the
    "wide" payment option).
    - i.e. the algorithms to implement multipath payments are much much more
      complicated than just "find a route and make onion".
  - Yes, you *can* batch splices in a way that goes "deep".
    BUT you would need a mechanism very much similar to the forwarding
    mechanism of forwardable peerswaps anyway in order to "go deep", and
    the complications (because we refused to standardize by BIP-69) may
    well make it impossible to batch deeply if your counterparty decides
    to splice something completely unrelated but affects the same channel
    while you are also deep-splicing (i.e. the order at which inputs and
    outputs are added to the splice is significant due to not standardizing
    to BIP-69, why is BIP-69 hated again?).

For these reasons, we should really devote much more developer time on
forwardable peerswaps than on splicing.

Participant Incentives In Forwardable Peerswaps
===============================================

We can broadly divide the participants in a forwardable peerswaps into
three main classes:

* The sole Initiator.
* Zero or more Forwarders.
* The sole Acceptor.

The Initiator offers "I want to give you onchain funds for more outbound
Lightning liquidity".

The Acceptor accepts the above offer; because of the symmetric nature of
liquidity, it now has responsibility for a new onchain fund, but also
gets inbound Lightning liquidity.

The Forwarders then effectively:

* Search for the best Acceptor, by forwarding via their own local
  channels.
* They are "paid" by having ***two*** channels rebalanced for free
  (whereas the Initiator and the Acceptor only get ***one*** channel
  rebalanced), without having to pay any onchain fees, whether to
  initiate the swap (Initiator), or to manage the onchain fund
  afterward (Acceptor).

Incentive To "Skip The Forwarders"
----------------------------------

As noted, the trade deal offered by the Forwarders to the Initiator
is:

* You get:
  - Contact with an Acceptor that ***needs*** inbound
    liquidity.
* I get:
  - Two of my channels rebalanced in my favor, for free.

Ideally, we want this trade deal to be atomic.

The problem is that when forwarding, it is possible to use
cryptographic data --- the hashes and public keys --- for the
Initiator to provide the Acceptor with enough information that
they can make a ***direct*** Lightning Network channel to each
other.

Suppose that we designed the protocol naively, and that the
Forwarders simply directly transfer the public key of the
Initiator for the timelock branch of the onchain HTLC, as well
as the hash, verbatim.

In that case, the Initiator might use its own node ID as the
public key to be used for  the timelock branch.
Then the Acceptor can look up the node ID in its gossip map,
see the Initiator node ID, and connect to the Initiator, and
they can directly create a Lightning Network channel,
cancelling the protocol with the Forwarders.

The Forwarders, who provided the service of "looking for someone
who needs inbound liquidity", are then left without what they
expected to get out of the trade deal, i.e. two channels
rebalanced in their favor.

Thus, we should always remember that public keys and hashes
can always be used as surreptitious communications tunnels.

We should remember that one reason for wanting data on network
conditions (i.e. channel data) is to discover opportunities to
undercut competitors.
Even though the Forwarders have not handed over their channel
balance data directly, the fact that the Initiator and Acceptor
can make a direct connection means that the Initiator has still
taken advantage of that data (as a form of summary).
The important part is to protect against this abuse, not just
protect actual bits of the data from being leaked.

Closing The Surreptitious Communications Tunnel
===============================================

In order to close this surreptitious communications tunnel,
the forwarders need to add their own randomizing data.

For public and private keys, this is trivial --- we just use
actual addition modulo `p` (or is it `n`? whatevs) on 256-bit
numbers for the private keys, and the homomorphic equivalent
for public keys.

SHA256, however, is not linear, and there is no similar way
to add randomizing data.

Fortunately, we do not need to actually provide the hash,
as we shall see.
The Forwarders can force the Initiator and Acceptor to commit
to continuing the Forwardable Peerswap protocol by the time
the hash is exchanged, so even if the hash is actually used
to encode the node ID of the Initiator, it would be pointless;
the Initiator is already committed to completing the protocol.

Forcing Commitment To The Forwardable Peerswaps Protocol
--------------------------------------------------------

Now, typically when we think about the onchain hop of an
onchain-offchain swap, we think of it in terms of creating an
onchain HTLC directly.

However, another way to view it is that the onchain hop is
actually a temporary, time-bound, unidirectional channel
(the timelock of the HTLC being the time bound), which then
hosts an HTLC.
Then the onchain-offchain swap can be seen as a simple
circular rebalance, i.e. a self-payment.

Have we seen time-bound unidirectional channels before?

Why yes we have, it is called the Spilman channel.

A Spilman channel uses a SCRIPT that encodes:

* One of:
  * Initiator AND Acceptor; OR,
  * Initiator after some time (i.e. the time bound).

Then, the Initiator can provide a signature to a
transaction that spends the above SCRIPT using the first
branch (Initiator AND Acceptor) to the Acceptor.
This transaction now contains any payments or HTLCs the
Initiator wants to offer to the Acceptor.

The drawback is that it is unidirectional, and the
second branch timelock creates a time bound; the channel
must be closed before this time.

This is ideal for the protection of the interests of the
Forwarders.

- As we have noted, SHA256 is not linear.
  The Forwarders cannot obfuscate the hash by somehow
  adding randomizing data, unlike with public keys.
  - However, notice the SCRIPT for the Spilman channel.
    There is no hash involved.
    All we need are two public keys: one encoding the
    condition "Initiator AND Acceptor", and the
    Initiator public key for use in the timeout
    branch.
    The Forwarders can prevent surreptitious data
    transfer by adding a random number to the public
    keys.
- The timebound assures the Forwarders that the
  Initiator will, in the future, still use the "real"
  Poon-Dryja Lightning Network channels for forwarding.
- The unidirectionality prevents the channel from
  being used for forwarding in the Lightning Network.
  - This is subtle!
  - On the actual Lightning Network, payment failures
    *can* occur, and by Murphy's Law, *will* occur.
    - Thus, we need a mechanism to refund an HTLCC
      that fails.
  - Refunding an HTLC is actually a violation of
    unidirectionality!
    - In the old state the HTLC exists.
      This HTLC encodes the ability of the Acceptor
      to claim the fund.
      However, refunding the HTLC means the new
      state has that amount given back to the
      Initiator, thus it is no longer unidirectional.
    - If the Initiator (incorrectly!) assumes that
      HTLCs hosted on the Spilman channel can be safely
      refunded, then the Acceptor can exploit this
      by using standard bidirectional Poon-Dryja
      channels back to the Initiator (via any number
      of obfuscating hops), then making a circular
      self-payment that terminates in the Spilman
      channel to the Acceptor.
      The Acceptor then pretends that the payment
      failed and offers to refund the HTLC on the
      Spilman channel back to the Initiator.
      If the Initiator then also refunds the
      incoming HTLC on the Poon-Dryja channel, the
      Acceptor can then steal by using the
      old Spilman state that still has the HTLC,
      and since this is Spilman and not Poon-Dryja
      it will not be punished (it is the
      unidiretionality that ensures correct Spilman
      operation).
  - There ***is*** a way to refund an HTLC under
    Spilman, but there are conditions.
    - The conditions are:
      - The HTLC must be the only one that was ever
        hosted on the Spilman channel.
      - The refund also causes the Spilman channel
        to close.
    - We do this by having the Acceptor sign a
      closing transaction that gives the entire
      channel fund to the Initiator (or equivalently,
      having the Acceptor hand over its private key).
      The Initiator can only treat the HTLC as
      being truly refunded if this close transaction
      is confirmed deeply.
    - This is ideal for driving the onchain hop of
      the onchain-offchain swap, because the channel
      closing in case of a protocol failure is exactly
      what the Forwarders want.
      Note that the Forwarders themselves want the
      protocol to succeeds (as success leads to two of
      their channels being rebalanced for free), and
      thus they will want to ensure reliability; they
      have no incentive to interrupt the protocol.

Thus, the Forwarders can use this mechanism to force
the Initiator to commit to completing the Forwardable
Peerswap protocol.

The only requirements are:

* The Forwarders *must* add obfuscation nonces to the
  two public keys involved:
  - The "Initiator AND Acceptor" public key.
  - The Initiator public key for the timelock branch.
* As mentioned, the tweak is necessary to prevent the
  Initiator and Acceptor from surreptitiously sending
  their node IDs to each other, to ensure that the
  Forwarders get their reward (two rebalanced
  channels, rebalanced gratis) for connecting the
  Acceptor to the Initiator.
* Once the Initiator and Acceptor have agreed to the
  Forwarder-tweaked public keys, they can derive the
  intended address for the temporary Spilman channel.
* Once the Initiator has created a funding transaction
  output for the temporary Spilman channel, the
  Initiator is committed to completing the protocol.
  - The Initiator obviously has to pay onchain fees
    to get the Spilman channel instantiated onchain.
  - The Initiator is thus forced to lock up those
    funds, which can only be freed if they complete
    the protocol.
* Thus, after the Spilman channel is confirmed to
  be opened, there is no need for any further
  obfuscation by the Forwarders, and other data,
  such as HTLC hash, can be safely copied verbatim
  by the Forwarders --- after this point, even if
  the Initiator and the Acceptor can locate each
  other, the Initiator funds are already locked
  into the Spilman channel.

Transferring The Initiator Public Key
-------------------------------------

Let us begin with the easy part: transferring the
timelock-branch Initiator public key.

During the first step of the Forwardable Peerswap
protocol, the Initiator must provide the public
key for the timeout branch of the Spilman
channel:

    I = i * G

Suppose the peerswap is forwarded by the peer by
one hop away, i.e. the Initiator's direct peer is
actually a Forwarder and not the final Acceptor.
In that case, the Forwarder will generate a 256-bit
tweak less than `n` (or is that `p`), and will send
 the point as:

    I + F = I + f * G

If the next peer then decides to be an Acceptor, it
also further generates another tweak, and in its
`forwardable_peerswap_accept` message, will *also*
reply with the tweaking scalar in addition to the
final point:

    a <- random
    I + F + A = I + F + a * G

The Forwarder can validate that `a * G`, added
to the original `I + F` it forwarded, matches the
`I + F + A` that the Acceptor replied with.

Then the Forwarder itself also replies to the
Initiator, sending:

    a + f
    I + F + A (copied verbatim from Acceptor)

Then the Initiator will validate that
`(a + f) * G`, added to the original `I` it
requested, matches the `I + F + A` that the
Forwarder replied with.

Since the Initiator is the only one that
knows the original `i`, and learns `a + f`
from the Forwarder and Acceptor, the Initiator
can use `I + F + A` as an Initiator-only public
key.

Notice also that the Acceptor cannot generate
`a` as some encryption of its own node ID;
even if it did, what the Forwarder will reply
to the Initiator is `a + f`, i.e. it will get
obfuscated using a scalar that only the
Forwarder knows.

There are two things we need to ensure here:

* Since the sum `I + F + A` would be copied
  verbatim by the Forwarder, we must ensure
  that the Acceptor cannot provide a fake
  `I + F + A` sum that is actually its node ID
  (or an encryption thereof).
  - Since the Acceptor has to provide `a`
    and the Forwarder validates this, if the
    Acceptor wants to send the Surreptitious
    point `S` instead of an honest `I + F + A`,
    it has to solve `a = (S - I - F) / G`, i.e.
    solve the Discrete Log Problem.
* Since the sum `I + F + A` will be used by the
  Initiator as an Initiator-only public key, we
  must ensure that the Acceptor cannot perform
  key cancellation, by providing `A = S - I - F`
  such that `I + F + A = I + F + S - I - F = S`
  for a secret key `S = s * G` known only by the
  Acceptor.
  - Again, this will require solving
    `a = (S - I - F) / G`, i.e. the Discrete Log
    Problem.

Thus, we know our method of obfuscating the
Initiator public key will prevent surreptitious
communication between Initiator and Acceptor,
*and* prevents key cancellation:

* Against surreptitious communications:
  - In the Initiator->Acceptor direction, the
    Forwarder sends `I + F`, so `I` becomes
    obfuscated, and any surreptitious data
    in `I` is masked by `F`.
  - In the Acceptor->Initiator direction, the
    Forwarder sends `a + f`, so `a` becomes
    obfuscated.
    - The `I + F + A` is copied verbatim, but
      it cannot be a surreptitious point `S`
      because the Acceptor has to reveal `a`
      and solve the Discrete Log Problem.
* Against key cancellation:
  - The need to reveal `a` and `a + f` prevents
    key cancellation by requiring you to solve
    the Discrete Log Problem.

Transferring the Initiator AND Acceptor Public Key
--------------------------------------------------

From the previous section, we already argued that if
the Acceptor can demonstrate knowledge of `a`, this
prevents both surreptitious data transfer from
Acceptor to Inititator, and key cancellation by the
Acceptor.

The easiest way to demonstrate knowledge is to reveal
`a` openly, but there is also a thing called
"zero-knowledge proof of knowledge".

As it happens, Schnorr signatures are actually an
example of "zero-knowledge proof of knowledge"; they
prove that the signature generator has knowledge of
the scalar.

Let us review how a Schnorr signature `(e, s)` is
generated:

    A = a * G ; public and private keys
    m ; message

    r <- random
    R = r * G
    e = hash(A | R | m)
    s = r + a * e
    (e, s)

The above can be validated by:

    R[v] = s * G - e * A
    e[v] = hash(A | R[v] | m)
    e ?= e[v]

Now, we should ask: why does the `e` calculation
include `A` in the hash?

The reason: because without it, it is possible
to malleate a signature for `A` into a signature
for `A + b * G` where `b` is a new random number.

However, for our purpose, that is actually what we
want!
The thing is that the Forwarder also needs to
demonstrate knowledge of `f`, without revealing
how many hops away the Acceptor is.
Thus, it should always provide a single `(e,s)`
that can be used with `A + F = f * G + A`.
So we can simply remove the `A` from the hash.
In addition, we can also remove `m`: what we
need is only to prove knowledge of `a` and `f`.

Thus generation of the zk proof-of-knowledge:

    A = a * G ; public and private keys

    r <- random
    R = r * G
    e = hash(R)
    s = r + a * e
    (e, s)

And validation:

    R[v] = s * G - e * A
    e[v] = hash(R[v])
    e ?= e[v]

When the Forwarder has validated the response
from the Acceptor, it can the malleate the
above proof by computing `s[a + f]` as:

    s[a + f] = s + f * e
    (e, s[a + f])

This then serves as a combined proof of knowledge
of `a + f`, the same as needed in the previous
section.

Notice that we use `(e, s)` instead of `(R, s)`.
The reason is that `R` might be used to
surreptitiously transfer data from the Acceptor
to the Initiator; notice how in the forwarding
case above, `e` is copied verbatim, and if we
were using `R`, then it would have to be copied
verbatim as well.
However, `e` is  the output of a hash function,
and using `e` to transmit information would require
reversing a hash function.

Then, for this particular public key, `I + F + A`
can be treated by the Initiator as
"Initiator AND Forwarder AND Acceptor".
This can be trivially reduced to "Initiator AND
Acceptor", after the Spilman channel is opened
using `I + F + A` as the combined key, by having
the Forwarders then reveal `f` to the Acceptor.
The `f` can be included in the message sent by the
Initiator once the Spilman channel is deeply
confirmed `forwardable_peerswap_onchain_ready`,
with the Initiator providing a `0` `f`, and each
Forwarder adding its own `f` when forwarding that
message until it reaches the Acceptor (the Forwarders
should cache the message until it has seen the
Spilman channel as deeply confirmed).
(For privacy, the Initiator should actually also
add its own obfuscating `f` when it first sends
`I` beforehand, so that the first Forwarder does
not learn that the direct peer is the true
Initiator; this is left as an exercise to the
reader.)

Operation After Spilman Confirmation
------------------------------------

Once the Spilman channel has confirmed, as noted,
for the "Initiator AND Acceptor" public key, the
Forwarders need to reveal their `f` tweaks to the
Acceptor.

After the Spilman channel is confirmed, the
Initiator can then offer an HTLC inside the
Spilman channel to the Acceptor.
This is done by creating a transaction that
spends the funding outpoint and has a single
output encoding the HTLC, and having the
Initiator and Acceptor use a MuSig2-like
protocol to generate the signature for that
transaction.

Then, once the Acceptor is capable of claiming
an HTLC on the Spilman channel, it can forward
to the Poon-Dryja Lightning Network channels,
eventually reaching back to the Initiator.

At this point, one of the following two things
can happen:

* The protocol completes.
* The protocol aborts (e.g. channel conditions
  changed so much that it is no longer possible
  to fulfill the in-Lightning side of the swap).

In both cases, we can use Private Key Handover
to reduce the onchain blockspace used to close
the channel.

As noted, in the abort case, we can simply close
the channel in favor of the Initiator, and once
that is confirmed, the Initiator knows that the
abort completed and it can recover its funds.

-------------------------

ZmnSCPxj | 2026-09-28 11:20:26 UTC | #2

Closing the Amount And Timelock Surreptitious Tunnels
-----------------------------

Despite this, there are still two remaining pieces of information that cannot be obfuscated by Forwarders:

* The amount to be peerswapped.
* The timelock for the Spilman channel hosting the "onchain" hop.

Of this, the amount is the one with the most bits available where the Initiator can provide information about its identity.  While a node identifier is 257 bits formally, what you need to locate a node on the actual living Lightning Network is far fewer bits; we only need to surreptitiously transfer enough bits to identify ONE node on the gossip map.  To close the surreptitious communication tunnel in the amount, we can standardize the onchain amounts among a small number of options, e.g.

* 10 BTC
* 1.0 BTC
* 0.1 BTC
* 0.01 BTC
* 0.001 BTC
* 0.0001 BTC

For the timelock, we can standardize that the timelock must be +2016 blocks from the current block height, and standardize a tolerance from +2012 to +2016 blocks (to tolerate new blocks arriving during initial negotiation).

Another piece of data that we might be concerned with be onchain feerates, which would need to be agreed upon by the Initiator and Acceptor, who are involved in onchain operations.  However, feerate decisions can now be deferred today by simply using a 0-feerate transaction for the in-Spilman HTLC offer; then if the Acceptor has to use it to claim the fund, it can pay using the child transaction, or if the Initiator has to use it to reclaim its fund in a protocol abort, it can pay using the child transaction as well.  Thus, feerates can simply not be transmitted in the protocol, preventing it from being used as a surreptitious communications tunnel.

-------------------------

