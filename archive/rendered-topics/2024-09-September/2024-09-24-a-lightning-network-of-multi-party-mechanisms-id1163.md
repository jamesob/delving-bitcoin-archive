# A Lightning Network of multi-party mechanisms

ZmnSCPxj | 2024-09-24 23:58:52 UTC | #1

I would like to propose splitting off the link layer
of Lightning Network from the network layer.

The motivation for this is to allow the network to
directly connect multi-party (N>=2) mechanisms together,
instead of having disparate multi-party mechanisms
transacting over 2-party channel straws, or presenting
a single multi-party pool as several 2-party channels
to the Lightning Network.
The existing network of 2-party channels would then be
a degenerate case where N=2.

This would require changes in the following:

* Gossip
* Onion Routing (of PTLCs, HTLCs, and onion messages)

Of particular note is that, for the *rest* of the
public network, what a group of nodes use to transfer
funds between them is actually immaterial.
That is, the multi-party mechanism might be a
fully-backed n-of-n Decker-Wattenhofer mechanism, or it
might be a federated custodial mechanism with credit
(negative balances) that is enforced by legal contracts,
or an Ark, or a BitVM machine, or whatever.
The rest of the network need not care about what the
exact mechanisms are.
Instead, the rest of the network only cares about how
reliable it would be to route through that mechanism.

Gossip
======

For gossip, the `channel_announcement` message would
have a variable number of nodes.
Note that not only do we need the public keys, we
would also need signatures from each node.
(The signature effectively indicates agreement that
the node is willing to route over this multi-party
mechanism.)
We can aggregate the signatures using MuSig2 so that
no matter how many nodes there are on a mechanism,
only one Schnorr signature is necessary.

An issue here is that messages are restricted by
BOLT8 to 65535-byte payloads.
With 33-byte node IDs we can have up to 1985.9 nodes,
though since we need other data, we might want to
restrict to 1000 published nodes per multi-party
mechanism.
(The multi-party mechanism might have more parties
than 1000, but the rest would not participate in
public routing)

Gossiped `channel_announcement`s would have to allow
nodes to be added and removed, to support mechanisms
that have dynamic membership sets.

Gossip Spam
-----------

The current design for `channel_announcement`s require
that nodes (pretend to) validate their outpoint against
the actual blockchain.
The channel outpoint has to be an actual unspent UTXO.

The intent of this is to prevent someone from making
up nonexistent channels and then using the channel data
to store information over the gossip network.
Actual users suffer since they have to spend more time
doing probes over a larger network where most of the
graph edges would never succeed in routing anyway
(because they are actually nonexistent, and are being
abused to store arbitrary data over the Lightning
Network).

Against this, we should note that some multi-party
mechanism designs, such as Ark, hit the blockchain
often.
That is, they will spend their "main" UTXO and then
create a new one every few blocks.

So, my proposal is:

* We check that the `channel_announcement` outpoint
  exists without checking it was spent.
* We *reject* `channel_update`s if the TXO is *not* a
  UTXO.
  - This limits the amount of data that is propagated
    over the network to only be as large as the UTXO
    set.
  - That is: we accept the *existence* of the mechanism
    even if the TXO backing it is already spent, but
    we refuse *additional data about it* if the TXO
    backing it is already spent.
* A `channel_update` can re-point to a new TXO that
  *is* a UTXO.
  - Thus, for example, an Ark mechanism can send a
    `channel_update` by pointing to its latest TXO,
    the one that is currently unspent.

Pathfinding
-----------

Most pathfinding algorithms assume that edges
connect to exactly 2 nodes on the graph.

To adapt these existing pathfinding algorithms to
multi-party mechanisms on the network,
pathfinding algorithms can "split" a N-party (N>2)
mechanism into multiple 2-party channels to a
common "virtual" node that represents the N-party
mechanism.
When building the onion for routing, you simply
skip over the virtual node that represents the
N-party mechanism.

Onion Routing
=============

An important point about onion routing is:

> We use SCIDs as a small way to identify the next
> *node*, not to identify the next *channel*.

In principle, if a node is a member of multiple
multi-party mechanisms, and the next node in the
onion is also a member of multiple multi-party
mechanisms, then the node has a choice of which
mechanism(s) to use to forward a payment.

Now, when mechanisms are 2-party channels, for each
node, the SCID uniquely identifies which peer is
the next hop.

However, when mechanisms are multi-party, then we
need further disambiguation.
An SCID is not enough to identify the next node in
the routing hop.

My concrete proposal is to use a "short node ID",
which is just the last 12 or 16 bytes of the node ID.
This should be sufficient to disambiguate, given
the random nature of node IDs.
This increases the size of the next hop ID by 4 or 8
bytes.
As I understand it, 128 bits is still too large a
space to find collisions with other node IDs, and 96
bits might also be sufficient.

For legacy 2-party channels that cannot have more
than 2 parties, then it can use the shorter SCID, as
in legacy onion routing.

-------------------------

renepickhardt | 2024-09-25 01:15:14 UTC | #2

Dear ZmnSCPxj,

I think it is very important to start thinking about extending the protocol to support multiparty channel constructs. As you know I [recently pointed out the advantages of multiparty channel constructions for payment reliability and service level guarantees of payment channel networks](https://github.com/renepickhardt/Lightning-Network-Limitations/blob/305db330c96dc751f0615d9abb096b12b8a6191f/Limits%20of%20two%20party%20channels/paper/a%20mathematical%20theory%20of%20payment%20channel%20networks.pdf). However I have not quantified the downsides of such constructs with respect to on chain bandwidth and costs of unilateral exists yet. So I am not sure yet about the tradeoffs that come with such constructs.

Additionally I have one concern about your proposal

[quote="ZmnSCPxj, post:1, topic:1163"]
## Pathfinding

Most pathfinding algorithms assume that edges connect to exactly 2 nodes on the graph.

To adapt these existing pathfinding algorithms to multi-party mechanisms on the network, pathfinding algorithms can “split” a N-party (N>2) mechanism into multiple 2-party channels to a common “virtual” node that represents the N-party mechanism. When building the onion for routing, you simply skip over the virtual node that represents the N-party mechanism.
[/quote]

While this makes sense from an engineering perspective I fear this proposed solution is not optimal. From a payment delivery perspective one wishes to forward sats from one multiparty construct to another but does not virtually want to fix 2 party channel(capacitie)s. It is my understanding that virtually creating two party channels would mitigate some of the advantages for routing abilities of the network that we gain from using multiparty constructs.

In any case I am looking forward to see progress on your proposal.

-------------------------

ZmnSCPxj | 2024-09-25 01:14:53 UTC | #3

In most basic pathfinding algorithms, edges have no maximum size. In practice what we usually do is to filter the actual graph when feeding it to the pathfinding algorithm to filter out channels that are smaller than the amount being routed.  The basic pathfinding algorithms are unaware of capacities.  Similarly, we can just make the capacity of the split out channels equal to the total capacity of the mechanism.

This may be different for flow-based pathfinding algorithms as you proposed for renepay before. But I suspect the flow-based pathfinding would need deeper changes to consider that each mechanism has more variables now than just `x` and `1 - x`.

-------------------------

ZmnSCPxj | 2024-09-25 02:22:49 UTC | #4

[quote="renepickhardt, post:2, topic:1163"]
However I have not quantified the downsides of such constructs with respect to on chain bandwidth and costs of unilateral exists yet. So I am not sure yet about the tradeoffs that come with such constructs.
[/quote]

Sure. Not being a mathist, I only have my engineering intuition here.

An intuition is that:

* As the number of participants increases, the probability *one* of them has a reason to force-close must increase. Assuming there is some reasonable average (mean) probability of force-close at any moment of time, `P(c)`. Then for N participants, the probability of force-close at any moment of time is `1 - (1 - P(c))^n`.
* More concretely: HTLCs are the most common reason for force-closing. In particular, once you participate in public routing, some fraction of your *incoming* HTLCs can be held hostage by your *outgoing* HTLCs, for a time.  If the outgoing HTLCs are *not* resolved, you need to drop them onchain.  If it resolves onchain, great, but if the outgoing HTLC is ***not*** confirmed onchain, ***or*** you, by complete accident, are forced offline (e.g. clumsy human foot kicking power supply), then your *incoming* HTLC is also dropped onchain.
  * Dropping an HTLC onchain *usually* means the hosting mechanism is *also* dropped onchain, i.e. force-closed.
  * Force-close due to HTLC timeout affects ***all*** participants in the mechanism; usually, the mechanism cannot continue operating offchain if *any* hosted HTLC has to be dropped onchain due to the accepting node going offline.

Thus, unless a multi-party mechanism can drop individual HTLCs onchain ***and*** continue operating, then N-party mechanisms are more brittle than 2-party channels.

It is probably measurable how much uptime a random node gets (just try to `connect` to a bunch of them, divide the successes by the number of attempts), and from there, we can assume that probability that an HTLC will time out due to timeout based on the probability of a random node being offline.  We can then compute the brittleness of N-party mechanisms as we vary N.

-------------------------

