# LN Summit 2024 Notes & Summary/Commentary

roasbeef | 2024-10-16 05:53:32 UTC | #1

Hi y'all,

<div data-theme-toc="true"> </div>

~3 weeks ago, 30+ Lightning developers and researchers gathered in Tokyo,
Japan for three days to discuss a number of matters related to the current
state and future evolution of the Lightning protocol (and where relevant,
the Bitcoin p2p and consensus protocol). 

The last such gathering took place in June found here:
https://lists.linuxfoundation.org/pipermail/lightning-dev/2022-June/003600.html.
2022 in Oakland, California. The prior raw meetings notes and summary can be
found here:
https://docs.google.com/document/d/1KHocBjlvg-XOFH5oG_HwWdvNBIvQgxwAok3ZQ6bnCW0/edit?usp=sharing. 

My raw meetings notes for this instance of the Lightning Developer Summit
can be found here:
https://docs.google.com/document/d/1erQfnZjjfRBSSwo_QWiKiCZP5UQ-MR53ZWs4zIAVcqs/edit?usp=sharing.
The rough daily schedule of discussion topics we agreed upon can be found
here: https://gist.github.com/Roasbeef/5ac91c57cb9826c628b1445670219728.

It's worth noting that there were many other break out groups and side
discussions that may not have been captured in my best-effort notes above,
nor reflected in the schedule I put together above.

With all that said, what follows is my attempt to summarize the major
discussion points, and conclusions (with some side commentary ofc ;)). If an
attendee finds anything in my summary inaccurate or incomplete, then
discussion and decision points that took place across the 3 days of the
summit, please feel free to reply and fill in the gaps.

# What's up with Package Relay and V3 Commitments Anyway?

This was the first major topic discussed, right on the heels of the latest
release candidate of Bitcoin Core 28.0 (at the time of writing, has now been
released).

## Fee Estimation & Base Commitments

Before jumping into the latest and greatest as far the newly proposed
commitment design, I'll first give a brief overview on how the commitment
design works today, and the shortcomings thereof.

(if you already know how commitment transactions work today in LN, then you
can skip this section)

A key aspect of the design of the Lightning Network is the concept of
unilateral exit. At any time, either party needs to be able to forcibly exit
from the channel to reclaim their funds after a time delay. A time delay is
needed as it's possible that one party attempts to cheat by publishing an
old revoked state to the chain. The time delay gives the honest party a
chance to refute the claim by an adversary, proving that they know a secret
which could only have been utilized if a revoked state was published.

The ability to unilaterally exit from a channel is also a key component
required to be able to enforce the HTLC contract, which implements the
"claim of refund" functionality that makes multi-hop Lightning payments
possible. These contracts have another time delay component: an absolute
time delay. Along a route, each hop has T blocks (the CLTV time lock delta)
to confirm their commitment transaction, and also time out any lingering
outgoing HTLCs. If they can't confirm in time, then they lose risking funds
with a "one sided redemption", as then the incoming HTLC will timeout,
creating a timeout/redemption race condition.

A node's ability to achieve timely confirmation of their commitment
transaction depends on their ability to do effective fee estimation. If the
commitment transaction (which can't unilaterally be modified by a single
party) has insufficient fees, then it may not be confirmed in time (or even
be accepted to the mempool!). Today, the initiator of a channel can send the
`update_fee` message to increase (or decrease) the current commitment fee
rate. This is a critical tool, however it forces the initiator to either be
prepared to significantly overpay for the commitment fee (to ensure they can
land in the next block) or expertly guess what the fee will be ahead of
time. As the initiator pays all fees on the commitment transaction, the
responder is unable to directly influence what this fee may be, instead
their only recourse today is to attempt to force close if they disagree on
what the fee should be. 

## Anchor Outputs to the Rescue!

In order to address some of the shortcomings of the existing commitment
transaction format, anchor outputs were proposed [1]. The general idea is
that both peers gain a near-dust output that can be spent by only them,
which allows bumping the fee of the commitment transaction after the fact.
This lessens, but doesn't eliminate the requirement to estimate what future
fees will be. Rather than ned to get into the very next block, the goal is
now to have a fee high enough to get into the *mempool*. Once in the
mempool, then either side can bump the fee to eventually confirm the
commitment transaction. Along the way, we also allowed second-level HTLCs to
be bundled together, allowing for a greater degree of batching when sweeping
resolved contracts. The fee needed to get into the mempol, similar to the
fee needed to get into the next block, is also a moving target. As nodes
have a default constrained mempool of ~300 MB, as the transaction arrival
rate increases, nodes begin to drop transactions from their mempool,
increasing their minimum transaction fee in the process. Eventually, the
existing max anchor transaction fee may not be high enough to actually get
into the mempool, meaning the commitment transaction is dropped out. When
this happens, nodes no longer have a way (using the existing p2p network) to
propagate their commitment transaction, meaning it may not confirm in time
to avoid the redemption race condition, or at all.

Along the way developers and researcher discovered a series of subtle
interactions with transaction broadcast, and mempool relay policies that may
actually allow an adversary to "pin" [2] a transaction to the mempool,
thereby purposefully preventing its confirmation. The various known pinning
vectors take advantage of degenerate cases related to BIP 125, and the
various ancestor limits widely utilized in mempool policy today.

## V3, TRUC, and 1P1C to the Rescue (actually this time)!

Enter transaction v3 and TRUC. The ultimate dream of LN developers has been
to just get rid of `update_fee`, and instead just have the commitment be
*zero fee*. This takes away all the guesswork of figuring out what the true
mempool fee should be. In isolation however, if the commitment transaction
is zero fee, then it won't be accepted to the mempool, so it won't propagate
across the p2p network.

A combination of TRUC (a.k.a BIP 434), a new style of anchors, and
optimistic 1-Parent-1-Child relay is the current best known solution that
practical addresses LN's current transaction relay and confirmation
problems.

TRUC introduces a new set of transaction replacement rules meant to address
the degenerate cases of BIP 125 for a subset of use cases. Along the way, it
introduces a new set of transaction topology size restrictions to further
constrain the problem. TRUC transactions use a transaction version of 3
(instead of a sequence number like BIP 125) to signal that a transaction
wishes to opt into the new rule set. 

Also available with Bitcoin Core 28.0, is a new standard public key script
type: PayToAnchor (P2A) [2]. P2A is a new special segwit v1 output (`OP_1
<0x4e73>`) meant to be used for the purpose of fee bumping via CPFP. The
input spending this output _must_ have a pure witness input, and can be used
without a signature. Future versions of this new output type may eventually
allow outputs to be dust, as long as they're immediately spent in the same
block they're created within (via CPFP).

The final component of the design for the newly revamped LN commitment
transaction is: 1-Parent-1-Child (1P1C) [4]. 1P1C is basically opportunistic
package relay. Rather than rely on a new p2p message which may take time to
be deployed across the entire network, 1P1C has nodes change their behavior
when it comes to receiving orphan transactions (txns where the node doesn't
know the inputs). Instead of just storing the child in the orphan pool, the
node will now opportunistically attempt to fetch the parent from the
reporting node, even if the parent has a fee which is below the current
mempol min fee.

In concert, these three new transaction relay primitives can be used to
re-design the Lightning commitment transaction format (namely anchors) to
resolve many of the long standing issues described above. t-bast has already
started to prototype what the new commitment format would look like:
https://x.com/realtbast/status/1834213774674247987.

With all that said, there're a few open design questions, including:
  * How should dust be handled?
    * With the P2A approach, we can put all the dust into the anchor output
      itself, instead of going to miner fees as they do today. This would
      resolve some known issues related to an excessive amount of dust on a
      commitment transaction, but also add some new interactions if the P2A
      anchor output requires no signature.
  * Should the P2A output actually be keyless?
    * If the output is keyless, then any 3rd party can *immediately* help to
      peers can sweep, until 16 blocks after confirmation). sweep the anchor
      output (compared to today when only the two channel
    * Related to the above, if we put all the dust outputs in the P2A, then
      whoever sweeps the P2A can also claim the entire dust output.
      Naturally miners would be the parties that are most reliably able to
      claim these funds, assuming there's no signature required.
    * Some argued that we should still have a signature here, as then it
      restricts the parties that can interfere with the channel peers
      attempting to confirm the commitment transaction. This is meant to
      hedge against a future discovered defect in the TRUC+P2A interaction,
      that enables a yet to be discovered type of pinning.
  * Does the viral nature of V3 transactions impede certain use cases
    related to advanced splicing?
    * All unconfirmed descendents from a v3 transaction must also be v3
      themselves. There was some concern that this would impact uses of
      splicing where a node attempts to satisfy several transaction flows
      with a single batched transaction. The viral nature of the v3
      transaction type would then force any parties that are spending
      unconfirmed change to themselves use txn v3, which may be beyond the
      extents of their current capabilities.


The new TRUC rules also allow for a type of package RBF, where if there's a
new conflicting package, then it'll attempt to do RBF against the existing
conflicting package set. This is also known as sibling eviction in the
context of 1P1C [5].

All the information above can be found in more detail in this handy dandy
guide for wallet developers [6].

When the dust settled, this was probably one of the more concrete new
initiatives following the spec summit. From here, we'll begin to spec out
exactly what the new v3 commitment type looks like, while we wait for a
sufficient amount of the p2p network to update to the new version, so we can
rely on the new relay behavior.

One thing worth calling out is that this shift will have further
implications w.r.t how the wallets that back Lightning nodes handle UTXO
stockpiling. With this approach, given the commitment transaction will have
zero fees, in order to confirm the transaction, a wallet _must_ use an
existing UTXO to anchor down the commitment transaction for it to be even
broadcastable. In practice, this means that wallets will need to reserve
funds on chain for the purpose of force closing channels in a timely manner.
Tools such as splicing, and submarine swaps can be used to allow wallets to
shift around funds, or batch several on-chain transactions in one.

# PTLCs & Simplified Commitments

Next up was a session focussed on discussing the combination of PTLCs, and
the Simplified Channel State Machine. At first, these two topics might
appear to be completely unrelated, but as we'll see shortly, some of the
edge cases that arise with design considerations of PTLCs can be mitigated
by modifying the current channel state machine protocol (dubbed the
Lightning Commitment Protocol - LCP) to a simplified variant.

First, a brief background on PTLCs. Today in the LN protocol, we use payment
hashes to implement the multi-hop claim-or-refund gadget that allows for
trust-minimize multi-hop payments. While simple, the current protocol has a
big privacy drawback: the payment hash is the same over the entire route, so
if an adversary appears in two locations in a route, then they can trivially
  correlate a payment (and also disrupt MPP circuits). 

To fix this privacy hole, many years ago developers proposed we switch over
to using EC points and private keys instead of payment hashes and preimages.
In
2018 a formalized scheme was published [7], that would actually allow the
construction to be instantiated using adapter signatures across both the
ECDSA and Schnorr signature schemes. This was interesting, as it meant that
we didn't necessarily have to wait for taproot (which enabled schnorr) to be
activated. Instead, a single multi-hop lock could be transported across
ECDSA and schnorr hops alike. Ultimately, for various reasons, this hybrid
version was never deployed. The upside is that we can now deploy a simpler,
unified, schnorr-only version of multi-hop locks.

Fast forwarding to the present, instagibbs, alongside of his research into a
concrete design of LN-Symmetry has been exploring the design landscape, from
messaging changes to state machine extensions [8].

After discussing some of his findings, we started to zoom in on some key
design questions:
  1. Should we use a single signature, or multi-sig adapter signature? With
  either approach, the adapter `T` used to create the signature will allow
  the proposing party to complete the private key needed to sign for the
  incoming HTLC.

     The musig2 based adapter signature is smaller (just a single sig
     instead of two), but adds extra coordination requirements as a nonce
     needs to be sent by both parties to properly create a new commitment
     transaction.

     The single sig adapter is larger (two signatures, just like current
     second-level HTLCs today), but simplifies the protocol, as HTLC
     signatures can be sent alongside the `commit_sig` message as usual.

  2. If we decide to go with the musig2 adapter signature design, then
  should we attempt to retain the current full-duplex async LCP flow, or
  should we instead simplify further, instead going for a simple sync
  commitment state machine protocol?

     The introduction of musig2 nonces for second-level HTLCs complicates
     the existing LCP protocol as we can no longer send the second-level
     HTLC signatures along side the `commit_sig` message, as the proposer
     now needs a partial signature from the accepting party before they can
     safely proceed.

     However, if we modify the channel state machine protocol to instead be
     _round based_, then though we sacrifice some x-put, we don't need to
     worry about handling the various interleaved executions possible
     (simultaneously `update_add_sig`+`commit_sig` by both parties). This
     brings us to the topic of Simplified Commit.


## Round-Based Channel State Machine Protocols

Today in the LN protocol, we have a full-duplex async state machine. What
this means is that both sides can propose updates at any time, without
requiring a priori agreement from the other party. At any time, we may have
4 possible commitment transactions at play: one finalized commitment for
each party, and another pending commitment (sig received, but revoke not
sent). Assuming both sides continue to send sigs and revoke their old
commitment, then eventually the "commitment chain head" for both parties
will converge on the same set of active HTLCs.

This scheme is great for throughput, as if both sides sent several
revocation points at the start of a connection, then they can continue to
send new states without every waiting for the other party, as they consume
revocations in a sliding window, gaining a new one from the remote party as
the revoke old states. It resembles the sliding window aspect of TCP.

One downside of this protocol is that we can never recover from a
disagreement or error encountered during execution. As both sides just send
updates at will (continuing to do so even after retransmission), there's no
way to pause or go backwards to recover and resume the protocol.

One example is the channel reserve. With the current commitment transaction,
each time either party adds an HTLC, the initiator needs to pay for the fees
from their funding input. However, it's possible that in order to actually
pay propose new HTLCs at any time, it's possible that for this fee, they
need to dip into the channel reserve. As both parties can propose HTLCs at
anytime, in order to prevent this edge case, they need to leave enough fee
buffer for future HTLCs. However it's difficult to accurately model what the
future HTLCf low from the remote party will be.

If we had a round based protocol, then we'd be able to catch all these edge
cases up front, and ensure that we can always resume channel execution,
avoiding expensive force closes. Such a round based protocol would resemble
RTS (Request To Send) and CTS (Clear to Send) based flow control protocols
[8]. 

During normal execution, both sides take rounds proposing changes to the set
of commitment transactions (add or remove HTLCs). If one side doesn't have
any changes, then they can yield to the other party. Importantly, with this
simplified protocol, either side can explicitly NACK or ACK a set of
proposed changes. With the ability to NACK proposed updates, then we can
recover from incorrect flows, making the protocol more robust in the face of
spurious force closes.

If we send up going with the musig2 based adapter PTLCs, then during the
round based execution, both parties can send nonces up front, eliminating
the difficult to reason about async interleaved nonce exchange. One other
bonus is that this protocol would likely be a lot easier to reason about, as
the current state machine protocol is notoriously underspecified.


# Getting Fancy Off-Chain: Super Scalar, Channel Factories & Friends

To close out the first day, we had a session focused on off-chain channel
constructions that utilize shared UTXO ownership to enable: off-chain
channel creation, cheaper self-custodial mobile on boarding, and batched
multi-party transaction execution. Related proposed protocols include:
channel factories, timeout trees, ark, clark, etc, etc.

A recently published proposal, SuperScalar [9], seeks to combine many of
these primitives, into a solution to the Last-Mile Problem as pertains to
self-custodial mobile Lightning. SuperScalar seeks to improve the state of
the art while: ensuring the LSP can't steal funds, not relying on any
proposed Bitcoin consensus changes, and finally retaining the ability to
make forward progress with the system while allowing some/all users to be
offline.

SuperScalar can best be understood as combination of 3 techniques: Duplex
Micropayment Channels [10], John Law's Timeout Trees [11], and a laddering
technique that allows the coordinator of the SuperScalar instance to spread
out their funds and minimize opportunity cost.

I won't attempt to describe the scheme in full detail here, instead I'd
refer interested parties to the Delving Bitcoin post referenced above. Since
the summit, Z has created a few new iterations of his scheme, addressing
some of the shortcomings and also branching off in distinct directions.

At a high level, once you combine all of the above, you have a big tree of
transactions, with the leaves of each transaction being a normal 2-party
channel, with the LSP and a user. One level up from the channel leaf, is
another combined multisig of the participants in that sub-tree. Each leaf
also has an additional output dedicated to it, with L additional coins that
can be used to allocate additional liquidity to a channel, requiring only
the LSP and that user to be online. If more parties are online, then a
higher branch in the tree can be re-signed, allowing broader redistribution
of capacity in the channel.

The laddering technique comes into play to allow the LSP to distribute their
fund over several instance of this off-chain tree structure. Timeout trees
are utilized to give all users a delay based exit path. Rather than needing
to always force close to reveal the entire tree off-chain, after a period of
time, all funds in the construction go to the LSP. This means that users
need to jump to the next instance/ladder of the structure, similar to the
way shared VTXOs work in the Ark construction (which also uses a form of
timeout trees). As a result, all channels in the construction no longer have
an indefinite lifetime: users either need to send all funds out of the
SuperScalar instance, or work with the LSP to gain a new channel in the next
instance. Otherwise, losers forfeit their funds to the LSP.

The lifetime of a SuperScalar instance can be divided into two periods: an
active period, and a dying period. During the active period, users use their
channels as normal. They might choose to exit the instance early, but can
remain offline mostly. During the dying period, the users MUST come online
to get their funds out to sweep themselves, or two move to another instance
of the tree. There's a sort of safety period built in, once the dying period
starts, the LSP will stop coordinating with users to make updates to the
tree, and also likely only sign off on outgoing payments (the LSP is a part
in all channels, but sub-channels are also possible, with some additional
trust).

Returning back to the extra output L, as described above, the output L is
freely spendable by the LSP. If a users need additional capacity, then they
can spend L, creating a new sub-channel with a given user, A. However, they
can also sign L with another user, B, thereby double spending the output L
_off chain_. Only one version of that spend can ever hit the chain, so in
essence the LSP has overdrawn, potentially stealing funds or causing a user
to forfeit funds they thought were theirs. One solution to this would be
using a signature scheme wherein signing twice causes the singer to lose
their private key. There're a few ways to construct such a scheme: OP_CAT,
decomposing the signature into 7+ instances, or using the two-shot adapter
signature scheme described in this paper [11].

The usage of Duplex Micropayment channels in higher internal branches means
the number of internal updates grows, so the does the amount of transactions
a user need to publish in order to forcibly exit from the construct. As
always, we eventually bump into an inescapable tradeoff related to the unit
economics of blockchains: if the transaction cost required to make a payment
exceeds the value of the payment itself, it either won't happen, or will
happen on a system that makes a tradeoff of security vs cost. In other
words, it may not make much sense for users to have small channels in such a
construct, due to the additional fees required on forced exit. For small
channels to be economical then the coordinator needs to subsidize them, or
users bank on never needing to manifest them on-chain, as they're always
hopping to the next SuperScalar ladder.

One other interesting topic that came up was a sort of conjecture re the
impossibility of securely joining a channel off-chain, without any on-chain
transactions at all. To see why this can be difficult, consider a scenario
where Alice and Bob already have a channel, and want to add Carol to the
channel. A+B make a new state update, and add a third commitment output to
the channel, using Carol's key. Carol asks A+B for some information to
convince her that this is the latest state, but in practice, A+B can always
just fabricate some imaginary state update history. As the only two signers
in the root multi-sig, A+B can always just double spend the commitment they
gave to Carol, removing her from the channel, possibly stealing her balance.
If you squint a bit, this ends up resembling the "nothing at stake" issue
with PoS chains: there's no cost for A+B to make a fake history to fool, and
eventually cheat Carol out of her funds".

The main conclusion of this impossibility conjecture is that purely
off-chain dynamic membership (anyone can join and leave at anytime)
constructs either require: (1) trust in the root signers, (2) some sort of
attribution+punishment mechanism, or (3) on-chain transaction(s). Solutions
in the first category include: Liquid, Statechains, and Ark with
out-of-round payments. In the second category, over the past year we've seen
the emergence of systems like BitVM, which rely on a 1-of-n honesty
assumption, leveraging an interactive on-chain fraud proof to attribute and
punish faults. In the final category, I'd place constructs such as: Ark,
SuperScalar, and generally John Law's timeout trees. In this final category,
users use the new output(s) created by the on-chain transaction to verify a
set of valid+immutable transactions from leaf to root that when broadcast,
allow them to unilaterally claim their new channel.

With all that said, I think some relevant takeaways from this section were
that: 

  * Devs+services are seeking ways to on board users with a low chain
    footprint, that is also capital efficient.

  * Prospective solutions seem to incorporate some combination of: channel
    factories, time out trees, multi-party channels, and ephemeral off-chain
    coin swap protocols (the Ark family).

  * To avoid taking on too much complexity unnecessarily, any new solution
    should likely follow an incremental deployment plan, shipping components
    in series, with each of them building on top of each other.

# Bonus Session: Lightning Talks

In between sessions there was a call for lightning talks re anything cool
that people were working on.

One cool idea presented was basically an ability for users to recover from
spurious force closes. These happen from time to time, due to
cross-implementation issues, most commonly some sort of fee agreement. The
general idea here is to just give away an extra key to allow the remote
party to spend their output asap if they force close. This would be purely
an altruistic action on the behalf of the party that didn't go to chain,
it's a a friendly thing to do that helps out the other party.

Mechanically, one way to accomplish this would be to give the remote party
everything they need to sweep their output via the revocation path (which is
usually used by the opposite party). Some also discussed modifying the
output derivation construct slightly, and packaging new information in the
channel reestablishment message. The non-broadcasting party would only
reveal this information when they know for sure that the latest state has
been published and confirmed.

# Make Gossip Suck Less

To open up the second day, the first session was focused on identifying
concrete improvements that can be made to the gossip protocol.

## Gossip Syncing Heuristics

The gossip protocol has a well defined structure, but leaves many behavioral
aspects up to the individual implementations. Examples of such behaviors
include: How many sync peers do you maintain? Do you rate limiting incoming
gossip at all? How to validate new incoming channels (if at all)? Do you
periodically spot check the graph for missing channels? Do you just download
everything from scratch each time?

Through the course of the conversation, for the most part, each
implementation learned of some new thing that other implementations do that
they don't. Anecdotally every few months/weeks, we discover some subtle bug
that has been hampering the propagation of new channel updates or channel
announcements. lnd found that the biggest improvement to discoverability and
propagation they made recently was to actually start using the channel
update timestamp information in gossip queries. Without this, node's aren't
able to recognize that though they have the same set of channels as the
remote party (based in scid's), they other party may have some _newer_
channels than they did. If an implementation prunes "zombie" channels after
some period of time, but isn't actively syncing gossip, then if they don't
spot check by looking at the channel update timestamp in gossip queries,
then they're bound to fail to resurrect old zombie channels, thereby missing
large sections of the graph.

## Gossip 2.0

Next, we turned to the new upcoming revamp of the gossip protocol, code name
Gossip 2.5 (or 2.0, depending on who you talk to). Since the last spec
meeting, lnd has continued to progress on both the spec [14] and
implementation [15]. At this point the spec is waiting on additional
review/feedback, with lnd having had the protocol working e2e (new channels
only) since the start of the year.

One new addition discussed was adding SPV proofs to the channel
announcements. Some implementation either conditionally (eg: lnd with the
`--routing.assumechanvalid` flag) or unconditionally never validate the
on-chain provenance of announced channels. For light clients that use purely
the p2p network (eg: Neutrino), fetching tens of thousands of blocks can be
a big sink into power/bandwidth/cpu. If channel announcements optional carry
(but always commit to!) an SPV proof, then the existence of a channel can be
verified using only the latest header chain. If only the hash digest/root of
the final payload is signed, then nodes that don't need the extra proofs can
ask the sending party to omit them. In the past lnd has worked on a proof
format that supports aggregation at the batch level, which can likely be
reused [16].

As far as interop testing, other implementations either currently have other
priorities or may be waiting on upstream libraries to integrate musig2 (post
summit, the libsecp PR for musig2 was merged!). Today none of the major
implementations have added support for testnet4, so since it presumably has
no LN channels, attendees agreed to have the testnet4 be the first uniform
testbed for gossip 2.0! 

Gossip 2.0 does away with the old timestamp field on channel updates and
replaces them with block heights. This simplified rate limiting, as you can
prescribe that a peer only gets one update per block. As block heights are
globally uniform (no local aspects such as timezones), they're better suited
for various set reconciliation protocols. Several attendees had done some
research into repurposing the existing minisketch implementation, though as
we have distinct constraints, we may just end up using something different
all together.


(NOTE: I spilled coffee all over my laptop mid way through this session, so
I missed a good chunk of it while troubleshooting).

# Fundamental Limits on Payment Delivery

Next up, we had a session to discuss some of the latest
research/formalizations related to pathfinding/routing in the LN. The main
topic was a presentation/discussion related to some new research that
attempts to uncover the fundamental limitations of payment deliverability in
payment channel networks [13].

At a high level, the research models the graph as a series of edge+vertices,
with each edge carrying 3 attributes: the balance on the local side, the
balance on the remote side, and the total capacity. Given a sample graph,
one can determine if a payment is reachable if there exist a series of pair
wise balance modifications that give the "receiving" party the desired
balance end state. Rather than running normal greedy based path finding
algorithms, this looks at the global feasibility of a payment as a whole.
Note that this approach naturally captures the ability to force a rebalance
during a payment to make an otherwise unsatisfiable flow satisfiable.

Inevitably, there'll be certain payment flows that just won't be possible at
all. Reasons for this include insufficient channel capacity, the sender not
having enough balance, or the receiver, etc. When this happens, within the
model an on-chain transaction must occur to either add or remove funds from
the network's active balance set. Examples of on-chain transactions include:
opening a channel, closing a channel, splicing, or using submarine swaps.

Based on the above, given some starting assumptions (graph topology, balance
distribution, distribution to sample for the likelihood of a payment between
any two nodes), one can arrive at a sort of upper limit of effective
throughput for a payment channel network. To arrive at this value (T), you
divide the bandwidth of chain TPS (Q) by the expected rate of infeasible
payments (R) -- T = Q/R. If we set T to be something like 47k tps, then if
we plug in the current TPS of the main chain (~14), then we arrive at
0.029%, meaning that only 0.29% payments can be infeasible to approach 47k
TPS.

Ultimately, these figures boil down to some back of the envelope math based
on simplifying assumptions. One aspect not factored in is the possibly of
batching chain interactions, s.t several channels/users can configure their
off-chain capacity/bandwidth with a single on-chain transaction. The simple
derivation above also doesn't factor in balanced payments (eg: I send
between my 2 nodes w/ no fee), which will never need to trigger on-chain
transactions, yet aren't counted towards the TPS derivation. Nevertheless,
models like this are useful to get a feel for the limits of the system in
the abstract.

## Multiparty Channels & Credit Channels

The research then identifies two primitives that can help to reduce the
amount of infeasible payments: multiparty channels, and credit within the
network.

Multiparty channels aggregate several users in the channel graph,
effectively forming new fully connected subgraphs. The intuition here is
that: if you hold the amt of coins each party adds to a channel as constant,
then by increasing the number of parties you also include the max amount
that any given user can own. By increasing the max amount that any user can
own, you reduce the amount of feasible payments due to balance/capacity
constraints.

Turning now to credit, the idea is also simple: if a payment is infeasible,
then along some hop, credit can be introduced to permanently or temporarily
expand the capacity within a channel, crediting one party with increased
balance. To minimize systemic risk, such credit likely shouldn't be
introduced in the core of the network, instead only existing at the edges.
Protocols such as Taproot Assets can be used in theory to increase payment
feasibility while also reducing on boarding costs as they enable users to
natively express the concept of addressable/verifiable credit in channels. 

# Last Mile Mobile On Boarding

To close things out, we had two distinct, but related sessions, focused on
self-custodial mobile onboarding and UX. First, the Last Mile Problem as it
pertains to mobile UX onboarding [17].

Today in LN, most of the UX challenges arise when a user attempts to pay
another user, but the receiver is using a self-custodial mobile wallet. This
is similar to the last mile transportation problem when it comes to Internet
infrastructure and bandwidth: the internal of the network contains "fat
pipes" with high bandwidth to quickly shuffle information around the
internal network. However, getting from the internal network to the final
destination is more costly, less reliable, and slower. 

## Onboarding Costs & Channel Liquidity 

In the LN domain, rather than dealing with aging infrastructure or high
construction costs, the challenges are instead related to aspects of the
mobile platform itself. Compared to always-online routing nodes, mobile
nodes need to wake up to sign for new incoming updates. Additionally, if a
mobile node wants to be set up to be a primarily net receiver (no coins yet,
on boarding directly onto LN) then an existing routing node must commit some
liquidity towards them. The establishment of this first channel is a capital
sync on the part of the routing node, as it's possible the mobile node
churns out of the network, leaving an idle channel with funds suspended. To
regain the funds in the face of a persistently offline user, the routing
node then needs to force close, costing chain fees as well as time while the
relative time locks expire (up 2 weeks of delay).

As we dive into the last-mile liquidity costs, we begin to run into some
fundamental limits of unit economics. If a user is receiving just 10 sats
over the channel, and it would cost 1000 sats in chain fees to open the
channel, then creating connectivity for that user would be a net loss for
the routing node (not to mention min channel limits on the network today).
Any inbound that a routing node allocates to a low activity user could
instead be allocated to a higher velocity of corridor of the networking,
wherein the channel can earn fees to offset the chain allocation costs.
Assuming the costs are aligned, or subsidies are in place, then
infrastructure tools such as: Phoenix Wallet's JIT channel system, Liquidity
Ads [18], Sidecar Channels w/ Lightning Pool, Amboss' Magma, etc, etc.

## Protocol Induced UX Concerns

Interactivity and chain fee on boarding costs aside, the current protocol
design has some abstraction leaks that end up bubbling up into end user
mobile wallets. One example is the reserve: to ensure that both parties have
some skin-in-the-game at all times (deterrence against breach attempts) they
must maintain a minimum balance in the channel at all times (commonly ~1%).
This confuses users, as they commonly want to send all funds away from their
wallet to migrate (or otherwise), but instead find they always need to keep
a small amount of funds at all times. Tangentially, as fees rise, then the
size of the economically viable channel also rises along with it.

## Liquidity Fee Rebates

One solution for the dust/small amount problem that has popped up recently
is the concept of "fee rebates" used by phoenixd [21]. A fee rebate is a
non-refundable payment towards future inbound channel liquidity. Each time a
user receives funds via a special routing node (one that supports this
protocol extension), while the user doesn't yet have a channel, received
funds go into this fee rebate bucket. Once the user has enough funds in this
bucket, then the routing node will open a channel towards it, paying the
service and chain fees out of the fee rebate bucket. The min amount needed
to contribute towards channel opening will vary based on the current service
and chain fees.

From a practical perspective, fee rebates work pretty well. Assuming a user
is eventually able to receive enough, then they can instantly receive funds
without needing to pause for channel opening. Once they have enough funds to
warrant an L1 UTXO, then they pay for the creation of that UTXO from their
fee rebate bucket. This technique can be combined with systems like ecash
(pending amount represented in a mint), or even credit channels using
Taproot Assets as mentioned above (asset UTXO represented in a Pocket
Universe to defer L1 costs).

From here, the conversation turned back to various off-chain channel factory
like constructions, and their limits when it comes to a certain distribution
of chain fees, number of users, and the balance distribution of those users.
Basically if you imagine some sort of construct, either based on timeout
trees, then if there're 100 million users and each user has 1 satoshi it,
then it isn't economically feasible for them to unroll the entire thing on
chain (fees >
1 sat). If we assume there's a built in mechanism for users to move funds
elsewhere so the coordinator can reclaim the funds (similar pattern for Ark,
etc), then if the users don't exit in time, the 1 BTC is forfeited to the
coordinator. All users trying to go on chain is paramount to just burning
the entire amount, so some attendees theorized a "big red button" that can
be used to burn all the existing balances. Ideally burning would require
some sort of Script (or client side) verifiable proof that the coordinator
was about to cheat somehow.

While the above scenario is more or less just a thought experiment, I think
it teases at some of the fundamental limitations when it comes to chain
fees, and small L1 UTXOs. Nothing terribly new though, this interaction is
why most full nodes by default will recognize the concept of dust: if it
costs more than 1/3 of the UTXO balance to pay for fees to move the UTXO,
then it's uneconomical. The same applies for off-chain systems, with only
some sort of subsidy or exogenous value system as an escape hatch. Any
transfers that are uneconomical on the base chain, or the next higher layer,
will inevitably migrate to some other system that retains the BTC unit of
account, but trades off cheap fees for security.

# BOLT 12: What’s Next

Amidst the haze, savory flavors, and cold beverages of all you can eat+drink
Sukiyaki the BOLT 12 PR was merged into the spec repo! Earlier in the day,
as one of the last sessions of the summit, we had a session focused on
what's next after BOLT 12, namely what extensions that were cut from the
original version are people interested in pursuing. 

## Potential BOLT 12 Extensions

The first extension discussed was: invoice replacement. Consider a case
where a user fetches an invoice using an Offer, but waits too long before
paying, so all the blinded paths and/or invoice itself are expired. In this
scenario, it'd be useful for the user to be able to ask for a replacement
invoice. Exactly how this differs from just fetching another fresh invoice
using the Offer is perhaps a contextual question.

One area that some of the implementers seemed most poised to dust off again
is: recurrent payments. Portions of recurrence were part of the original
spec, but were eventually ripped out to slim things down some. Relevant
recurrence params include: the time interval, the payment window, limit, and
then start+end period. A neat trick that the receiver can use is utilizing a
hash chain to minimize the amount of preimage storage they need. If they can
send the sender a special salt/seed (during initial negotiation), then only
the sender+receiver would be aware that the preimages nicely arrange into a
hash chain.

On the topic of authentication, a notion of the sort of reverse version of
BIP 353 was brought up. The general idea is to give users the ability to bind a
node's public key to a domain name. This would serve to authenticate that
node Y is actually associated with some service/domain/company.

## Onion Message Rate Limiting & Back Pressure Propagation

At the tail end of the session, the focus shifted to onion messaging, and
the current state of implementation/behavior across the major
implementations. One topic raised was how wallets are handling the fallback
and related UX implications if/when a wallet _fails_ to fetch an Offer.
Onion messaging is an unreliable, best-effort forwarding network w/o any
built-in acknowledgements, so it's possible that a message is just never
delivered. As a result, wallets need to be ready to either try another
route, retransmit the message, or fallback to some other mechanism if/when a
wallet fails to fetch an invoice using an Offer.

Generally the current state of things is that either a single hop onion
messaging route it used, or a direct connection. A direct connection refers
to connection over the p2p network to either the receiver, the introduction
point, or nodes leading up to the introduction point in an attempt to send a
message that travels over a shorter path). If _that_ attempt fails (no nodes
listening, receiver not offline, etc) then wallets either need some other
fallback, or may attempt to send some sort of spontaneous payment.

Returning back to message delivery, it's clear that some sort of rate
limiting is needed. Nodes may start with some sort of free budget, but will
need to throttle messaging as otherwise tens a node could mindlessly forward
10s of GBs of free onion messaging traffic (IMO, it's inevitable that after
a free tier, most nodes will end up switching to a bandwidth metered payment
system for onion messaging [22]). Therefore nodes will need to adopt some
sort of bandwidth and rate limiting. If the network is significantly over
provisioned relative to the typical messaging usage, then service will
remain relatively high, as nothing approaches the configured bandwidth
limits. However if the network is under provisioned relative to messaging
activity (people are trying to livestream their gaming sessions or w/e),
then service is hampered, as most message attempts fail due to a tragedy of
the commons). As is, the difference between a message being dropped, not
delivered, or an offline receiver are all indistinguishable from each other,
creating further UX challenges.

Eventually discussion turned to the old back pressure rate limiting
algorithm previously proposed on the mailing list [23].  While this
algorithm, nodes can maintain a relatively compact description of the set of
peers that had sent them message last. Once a peer exceeds a limit, then an
`onion_message_drop` message is sent to the sender. The sender then attempts
to trace back who sent the message to itself, further propagating the
`onion_message_drop` message backwards, halving the rate limit in the
process. If the sender doesn't overflow the rate limit within a 30 second
interval, then the receiver should double their rate limit until it reaches
the normal rate limit.

There're some open questions lingering here such as: How can nodes make sure
they are attributing the spam to the proper peer? Is it possible for nodes
to frame other nodes to cut off their messaging activity? Is there any
additional meta data required to properly attribute the source of the spam?
Is this resilient to a spammer that knows the rate limit and can stay right
under it, while maximizing utilized bandwidth? When this scheme was
originally brought up, some basic simulations were run [24] to gauge the
efficacy and resilience of the scheme. Initial results were promising, with
some additional research questions posed [25]. 

Ultimately, some attendees agreed to start to revive the work/research on
the back pressure algorithm, applying conservative rate limiting parameters
in the short term.

-- Laolu

[1]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2018-January/000943.html
[2]: https://github.com/t-bast/lightning-docs/blob/398a1b78250f564f7c86a414810f7e87e5af23ba/pinning-attacks.md
[3]: https://github.com/bitcoin/bitcoin/pull/30352
[4]: https://github.com/bitcoin/bitcoin/pull/28970
[6]: https://docs.google.com/document/d/1Topu844KUUnrBED4VaJE0lVnk9_mV6UZSz456slMO8k/edit
[5]: https://github.com/bitcoin/bitcoin/pull/29306
[8]: https://gist.github.com/instagibbs/1d02d0251640c250ceea1c66665ec163
[7]: https://eprint.iacr.org/2018/472
[8]: https://en.wikipedia.org/wiki/IEEE_802.11_RTS/CTS
[9]: https://delvingbitcoin.org/t/superscalar-laddered-timeout-tree-structured-decker-wattenhofer-factories/1143?
[10]: https://tik-old.ee.ethz.ch/file/716b955c130e6c703fac336ea17b1670/duplex-micropayment-channels.pdfkj
[11]: https://github.com/JohnLaw2/ln-scaling-covenants
[12]: https://eprint.iacr.org/2024/025
[13]: https://github.com/renepickhardt/Lightning-Network-Limitations/blob/305db330c96dc751f0615d9abb096b12b8a6191f/Limits%20of%20two%20party%20channels/paper/a%20mathematical%20theory%20of%20payment%20channel%20networks.pdf
[15]: https://github.com/lightningnetwork/lnd/pull/8256
[16]: https://github.com/lightningnetwork/lnd/pull/5987
[17]: https://bitcoinmagazine.com/technical/assessing-the-lightning-networks-last-mile-solutions
[18]: https://github.com/lightning/bolts/pull/1153
[19]: https://lightning.engineering/posts/2021-05-26-sidecar-channels/
[20]: https://amboss.tech/docs/magma/intro
[21]: https://phoenix.acinq.co/server
[22]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2022-February/03498.html
[23]: https://gist.github.com/t-bast/e37ee9249d9825e51d260335c94f0fcf
[24]: https://gist.github.com/joostjager/bca727bdd4fc806e4c0050e12838ffa3
[25]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2022-July/003663.html

-------------------------

everythingSats | 2024-10-16 11:26:07 UTC | #2

80 IQ Version here:https://stacker.news/items/726105/r/EverythingSatsoshi

-------------------------

ariard | 2024-10-17 22:09:00 UTC | #3

Make technical comments here:
https://github.com/lightning/bolts/issues/1204

I’m still waiting, and I guess more people in the community too,  some TheBlueMatt comments (https://github.com/lightning/bolts/issues/1201) on the organization of this so-called LN dev summit. After all, consensus topics are discussed at this type of “closed-doors” meetings, and I don’t wish freely to accuse Lightning Labs and Block Inc to more or less make a cartel to hijack bitcoin consensus development. This is not like that Lightning Labs’s style of development of complex codebases have many times, be it in 2020 or more recently showing shortcomings: https://delvingbitcoin.org/t/non-disclosure-of-a-consensus-bug-in-btcd/1177/6

-------------------------

benthecarman | 2024-10-17 23:37:44 UTC | #5

Was there any leanings for the two PTLC directions? People have been saying these two different alternatives for awhile now but haven't seen any movement in either direction.

-------------------------

