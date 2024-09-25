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

