# Reimagining Onion Messages as an Overlay Layer

roasbeef | 2025-06-28 01:56:34 UTC | #1

# Reimagining Onion Messages as an Overlay Layer

## Introduction

Onion messaging as currently specified and deployed _embeds_ the onion
messaging graph into the channel graph. This tight
coupling between the onion messaging graph and the channel graph is a poor
design choice as it: necessitates a slow and brittle all-or-nothing adoption
path, forces an unnecessarily large onion messaging graph diameter,
intermingles distinct networking quality of service concerns, and misses an
opportunity to support native key management.

In this post we explore a deployment that addresses all of the drawbacks
mentioned above, we propose an alternative development path that instead
instantiates onion messaging as an _overlay layer_, bootstrapped using the
LN gossip layer. Compared to the embedded onion message networking of today,
the onion messaging overlay layer promotes rapid incremental deployment,
separates concerns, and may increase reliability and reach ability due to the
dynamic nature of the overlay topology.

## Motivation

The current onion messaging deployment path takes an all or nothing approach:
onion messaging links can only be created via established channels links. Any
sent onion messages therefore must flow over the exact same graph topology as
the channel graph. This tight coupling is fragile, as if a single node across a
prospective onion messaging path opts to not support the protocol, then
messages can't be transmitted. For presentation layer use cases like BOLT 12,
this forces clients to frequently fall back to directly connecting to the
message recipient (or penultimate node). This inevitable fall back sacrifices
privacy, and also damages the UX, as retries and re connections can significantly
delay the time to even first _payment attempt_.

Additionally, the all-or-nothing approach has slowed down deployment timelines,
as until nearly 100% of the nodes on the network support, clients must be
prepared to fall back direct connecting to the receiver. As new onion messaging
links can only be created by piggybacking on existing channels, the evolution
of the onion messaging ground is hampered, as its bound to the block creation
rate, and confirmation periods.

As a decentralized payment network, in ideal conditions, payment latency on the
Lightning Network is a function of the network diameter. A longer path length
requires more cumulative iterative round trips between successive peers, which
increase payment latency. By bolting the onion messaging graph onto the channel
graph, we force the onion messaging graph to operate a graph of larger diameter
than the solution necessitates. If onion messaging were instead an overlay
layer, a more compact messaging graph would emerge. Reducing the amount of
nodes that a BOLT 12 payment request must travel over, serves to reduce the
aggregate latency of the payment experience. Compared to the existing BOLT 11
flow, which permits a user to immediately carry out a payment, BOLT 12 incurs a
cumulative round trip latency over an unreliable messaging system over the
entire path (stalled message delivery attempts may also further hamper UX).
Enabling such invoice requests to travel over a smaller graph diameter can help
to reduce the UX penalty BOLT 12 adoption.

Current deployments of onion messaging re-use the node identified key of a
given peer typically intended for onion payments, in the onion messaging
domain. The lack of domain separation couples concerns, and overloads the onion
payment key. Additionally, as the BOLT protocol doesn't yet support a global
node identity rotation protocol, all onion messages effectively utilize a
static key. This lack of a key rotation mechanism serves to degrade the privacy
of onion messages overtime, due to lack of forward secrecy.

Due to the embedded nature of the current deployment, onion messages travel
over the same TCP connection between peers as gossip messages and channel
updates. By maintaining this tight coupling, the current deployment path isn't
able to independently address quality of service concerns. Due to head-of-line
blocking in TCP, any active gossip or channel update messages that remain
unacked in the TCP window (or even an outbound application queue) will
necessarily _block_ the processing of onion messages. This unnecessarily
blocking once again hurts the payment experience, as it leads to increase
latency of the time-to-payment-attempt.

As we'll see below, A deployment of onion messaging that instead utilizes an
overlay addresses all of the shortcomings described above. The onion overlay
layer separates concerns, promotes rapid deployment and experimentation, paves
a way to key rotation, and results in a smaller diameter graph. Importantly the
creation and processing of onion messages remains unchanged.

## Proposal

Instead of the current embedded approach, we instead propose an alternative
(can even be concurrent!) onion messaging overlay. The overlay utilizes the
existing LN gossip network and long-term node identity keys to boostrap a
distinct onion messaging overlay.

Onion messaging on the network won't require all 12k+ public nodes to function,
instead we'd be able to get by with even just 1% of the amount of nodes.

Beyond the immediate deployment advantages, the overlay architecture enables
rapid experimentation with onion messaging primitives beyond
BOLT 12. Examples of such proposals include: the addition of a payment level ACK to give immediate feedback to path finding iterations, and also safe retryable (stuckless) payments that rely on transmitting an extra preimage from sender to receiver.  These experiments can proceed in parallel w/o affecting the broader payment network, as early adopters can iterate on protocol designs, taking advantage of the swifter deployment path of the overlay network. 

### Connection Establishment

An existing active BOLT 9 connection is used to establish a new onion messaging
link. As we'll see below, link establishment produces a 3rd verifiable musig2
signature to prevent connection spoofing (claiming an onion messaging
connection exists when one doesn't).

```mermaid
sequenceDiagram
    participant A as Initiator
    participant B as Responder
    
    A->>B: onion_link_req
    
    B->>A: onion_link_resp
    
    A->>B: onion_link_sign
    
    B->>A: onion_link_sign
    
    A->>B: onion_link_proof
```

#### `onion_link_req`

The `onion_link_req` message is a pure TLV message that's sent from one peer to
another to request the establishment of a new onion messaging link. It carries
the sender's current `onion_key`, and an `onion_nonce` to use in the `musig2`
signature.

**Message Type**: `???`

##### TLV Fields

| Field | Type | Length | Description |
|-------|------|--------|-------------|
| `onion_key` | 1 | 33 | The sender's onion public key used for establishing the onion messaging link |
| `onion_nonce` | 3 | 66 | The sender's MuSig2 nonce for the link establishment signature |

#### `onion_link_resp`

This message is sent in response to an `onion_link_req` message. It includes
identical field, but this time instead for the responded.

**Message Type**: `???`

##### TLV Fields

| Field | Type | Length | Description |
|-------|------|--------|-------------|
| `onion_key` | 1 | 33 | The responder's onion public key used for establishing the onion messaging link |
| `onion_nonce` | 3 | 66 | The responder's MuSig2 nonce for the link establishment signature |

#### `onion_link_sign`

With the `onion_key` of both initiator and responded known, the initiator can
now generate the `musig2` partial signature that comprises of half the
`onion_link_proof`.

At this point, we assume that both initiator and responded already know the
long-term `node_identity` of each other. The generated signature will be
created using an aggregated public key of the set of `onion_key` and
`node_identity` key for both parties.

`aggregate_onion_proof_key = musig2_key_agg(onion_key_1, onion_key_2, node_key_1, node_key_2)`

**Message Type**: `???`

##### TLV Fields

| Field | Type | Length | Description |
|-------|------|--------|-------------|
| `onion_proof_partial_sig` | 5 | 32 | The initiator's MuSig2 partial signature for the onion link establishment |


Upon receipt, the responder:
  * Re derives the `aggregate_onion_proof_key`
  * Verifies the `onion_proof_partial_sig` from the sender
  * Generates its own `onion_proof_partial_sig`
  * Proceeds to construct the `onion_link_proof` message`

#### `onion_link_proof`

A proof of mutual agreement to establish an onion messaging link between two
nodes. This serves to allow 3rd party nodes to verify that an advertised
connection actually exists.

The initiator constructs this message, and sends to the responded.

##### TLV Fields

| Field | Type | Length | Description |
|-------|------|--------|-------------|
| `onion_key_1` | 7 | 33 | The initiator's onion public key |
| `onion_key_2` | 9 | 33 | The responder's onion public key |
| `node_key_1` | 11 | 33 | The initiator's node identity public key |
| `node_key_2` | 13 | 33 | The responder's node identity public key |
| `onion_link_sig` | 15 | 64 | The complete MuSig2 signature proving mutual agreement to establish the link |

Once the both parties have the `onion_link_proof` message, they'll create a new
TCP connection that uses a distinct port, using their the new `onion_key` pair
to establish the encrypted p2p connection.


### Onion Link Discovery

Now onto onion link discovery. As mentioned, node the existing LN gossip
network is used to bootstrap the new onion overlay network.

#### `node_announcement` Extension

A new TLV field is added to the existing `node_announcement` message. This
field contains a series of repeated `onion_link_proof` messages. In aggregate
his new field can be seen as an authenticated agency list, that can be used to
build up the latest view of the onion overlay network.

#### New TLV Field

| Field | Type | Length | Description |
|-------|------|--------|-------------|
| `onion_link_proofs` | 17 | variable | A list of onion link proofs for this node |

#### Nested `onion_link_proofs` 

The value of this TLV field contains:

| Field | Length | Description |
|-------|--------|-------------|
| `num_proofs` | 2 | The number of onion link proofs that follow |
| `proofs` | variable | `num_proofs` encoded `onion_link_proof` structures |

### Onion Graph Assembly

To construct the onion graph, a node syncs the LN graph as normal. They then
filter through all the nodes that advertise the onion messaging feature bit
(likely a new one is used for the overlay variant), verify all the
`onion_link_proofs`, then reconstruct the current onion message graph from the
authenticated adjacency list.

### Routine Operation

After boostrap, onion messaging functions pretty much as normal. Each member of
the overlay network has an authenticated mapping between a node's long term
identity key, and their latest ephemeral onion key. Using either, they can
perform path finding as normal to send and receive messages (blinded paths is
utilized as normal).

To rotate an onion public key, nodes can simply restart the flow to obtain a
new `onion_link_proof`, then advertise that once again in the network. Rotating
`node_key` pairs also necessitates reconnecting with a fresh BOLT 8 brontide
handshake.

## Interoperability with Existing Infrastructure

The proposal as described above doesn't disrupt the existing embedded onion message deployment. Instead, the embedded and overlay messagign networks can actually _co exist_. Nodes can simultaneously maintain connections to both the embedded messaging network and the overlay network, treating them as complementary transport mechanisms. 

The onion packet construction remains identical regardless of the underlying transport. As a result, any existing deployed applications that use onion messaging can seamless utilize both the overlay and embedded network. The overlay network can be more rapidly deployed and utilized, while also avoiding fragmentation which may hamper the primary use cases. 

## Conclusion 

In this post, we've proposed an alternative (and even parallel!) deployment of onion messaging. As an overlay, the onion messaging network can be decoupled from the existing channel graph. With as few as 100 nodes using this overlay, higher level applications using onion messaging can immediately begin to seamlessly utilize this new overlay, which allows application developers to hone in on the last 20% of work to truly polish user experience. The overlay approach supports loosely synchronized deployment, which embraces permissionless innovation and rapid experimentation.

-------------------------

shocknet_justin | 2025-06-28 18:12:18 UTC | #2

I'm disheartened to see mental cycles as valuable as yours Mr. Beef being used on something like this, onion messages are a terrible thing and beyond saving. 

[As I have long warned and been vindicated](https://stacker.news/items/730371/r/justin_shocknet?commentId=731289), each hop inherently increases latency and failure probability. This makes OM's unfit for communication in areas where high reliability and performance are paramount, like payments. Bolt12 has no future because of this. 

Anyone who has used Tor can attest to the fact there is no production value to OM's. To the extent OM's work on Lightning even conceptually is due only to the nature of the network being a defacto bonding apparatus, any further decoupling from that can only make reliability worse. 

Now, Lightning would absolutely benefit from a better overlay network for communication, but that is almost unanimously Nostr. Concerns are properly separated, web comportability allows deprecation of things like LNURL, and identity can be either leveraged trivially or entirely de-coupled based on the use-case. 

Better protocols like we're developing with [CLINK](https://clinkme.dev) solve the node-to-app coordination problem without the unreliable p2p topology. The preceding link demonstrates Nostr static payment offers that retrieve bolt11 invoices from a simple static web page without wasm, and there are also specs for the reverse flow (debits) and a draft for remote management of static offers (app manages offers on a remote node).

-------------------------

MattCorallo | 2025-06-30 12:41:34 UTC | #3

The reason the onion messages go over channel links is there's an inherint anti-DoS protection to using your existing channel partners to accept onion messages for forwarding. What your proposal appears to be missing is a way to handle DoS/onion message peer selection in a useful way.

[quote="roasbeef, post:1, topic:1799"]
Due to the embedded nature of the current deployment, onion messages travel over the same TCP connection between peers as gossip messages and channel updates. By maintaining this tight coupling, the current deployment path isn’t able to independently address quality of service concerns. Due to head-of-line blocking in TCP, any active gossip or channel update messages that remain unacked in the TCP window (or even an outbound application queue) will necessarily *block* the processing of onion messages. This unnecessarily blocking once again hurts the payment experience, as it leads to increase latency of the time-to-payment-attempt.
[/quote]

There's nothing in the spec that mandates this. It is how its implemented in most clients, today, indeed, but AFAIK all such clients have active queue management to address the QoS concerns. Future implementations that have issues here can, of course, establish parallel TCP connections to forward onion messages over, as you noted in your QUIC exploration.

-------------------------

roasbeef | 2025-06-30 19:23:57 UTC | #4

> What your proposal appears to be missing is a way to handle DoS/onion message peer selection in a useful way.

Peers would be free to _reject_ an attempt to create an onion link if the requesting peer doesn't already have a channel with the responding node. 

_Requiring_ that a channel already exists doesn't necessarily provide DoS protection. It just increases the cost to creating an onion link. The onion link establishment protocol sketched out in the OP can be extended to require an upfront payment, require presenting a UTXO provably locked to a future time, stream funds to maintain link uptime, etc.  

> There’s nothing in the spec that mandates this. 

That mandates that the same TCP connection be used? Can you point to where this is explicitly outlined? I can find no text that discusses anything of the sort. Minimally, it would need to select/dictate a new port. Implicitly, after BOLT 1, all p2p interaction is assumed to take place over a single port. 

> but AFAIK all such clients have active queue management to address the QoS concerns

Can you describe such active queue management? Last time I asked about how clients handle active queue management (BOLT issue where I brought up QUIC), the responses were all something along the lines of "we make sure not to send too much". Which is a writer side heuristic, rather than a reader side queue management. 

No matter what type of active queue management takes place, a single global TCP connection for all p2p messages will _still_ run into head-of-line blocking.

-------------------------

ZmnSCPxj | 2025-07-01 12:00:57 UTC | #5

[quote="roasbeef, post:4, topic:1799"]
> What your proposal appears to be missing is a way to handle DoS/onion message peer selection in a useful way.

Peers would be free to *reject* an attempt to create an onion link if the requesting peer doesn’t already have a channel with the responding node.

*Requiring* that a channel already exists doesn’t necessarily provide DoS protection. It just increases the cost to creating an onion link. The onion link establishment protocol sketched out in the OP can be extended to require an upfront payment, require presenting a UTXO provably locked to a future time, stream funds to maintain link uptime, etc.
[/quote]

As I understand it, the DoS issue here is that I can invent any number of "public nodes" by just runnng a cheap CSPRNG and getting 256-bit sections of its output, then inventing a bunch of `onion_link_proof` messages with myself and my random number, and pump it at the gossip network.

Third-party nodes (i.e. every node other than me and "partner" in the `onion_link_proof`) can only verify that the node exists as an actual entity if there is an onchain trace that pays for blockspace.  Note that a "UTXO provably locked to a future time" is no different from a channel in that regard --- the DoS-protection heuristic is the same, "blockspace was used, so it was not a cheap CSPRNG output", and would fall victim to your exact objection, that an onchain action (and thus non-cheap blockspace) is necessary.

-------------------------

MattCorallo | 2025-07-01 12:55:02 UTC | #6

[quote="roasbeef, post:4, topic:1799"]
Peers would be free to *reject* an attempt to create an onion link if the requesting peer doesn’t already have a channel with the responding node.

*Requiring* that a channel already exists doesn’t necessarily provide DoS protection. It just increases the cost to creating an onion link. The onion link establishment protocol sketched out in the OP can be extended to require an upfront payment, require presenting a UTXO provably locked to a future time, stream funds to maintain link uptime, etc.
[/quote]

So what you're saying is that in practice your proposal wouldn't change the network topology at all? What's the point, then?

[quote="roasbeef, post:4, topic:1799"]
That mandates that the same TCP connection be used? Can you point to where this is explicitly outlined? I can find no text that discusses anything of the sort. Minimally, it would need to select/dictate a new port. Implicitly, after BOLT 1, all p2p interaction is assumed to take place over a single port.
[/quote]

The spec doesn't say that it has to be on the same connection, which implies that nodes are free to do whatever they want. Nodes today may restrict peers to a single connection with the same pubkey, but that doesn't mean its banned? Also, there's no need for a separate port for this? You can just open a second connection on the same port.

[quote="roasbeef, post:4, topic:1799"]
Can you describe such active queue management? Last time I asked about how clients handle active queue management (BOLT issue where I brought up QUIC), the responses were all something along the lines of “we make sure not to send too much”. Which is a writer side heuristic, rather than a reader side queue management.
[/quote]

Yes, TCP is all about applying backpressure from an application through to the sender to ensure the sender can do these things without making the recipient unable to respond. You don't need receiver-size queue management if the sender is handling TCP correctly :).

[quote="roasbeef, post:4, topic:1799"]
No matter what type of active queue management takes place, a single global TCP connection for all p2p messages will *still* run into head-of-line blocking.
[/quote]

Sure, but if you don't send too much the head-of-line blocking isn't going to slow down channel operations. Head-of-line blocking isn't some death curse, for a long-lived connection where we're processing messages fairly quickly (basically all lightning messages are *super* quick to process) its not gonna materially impact latency. The web has major head-of-line blocking issues because they can't open parallel connections (due to slowstart, which isn't an issue for us) nor signal (in practice) which resources are needed urgently and which won't impact pageload.

-------------------------

roasbeef | 2025-07-01 22:18:36 UTC | #7

> As I understand it, the DoS issue here is that I can invent any number of “public nodes” by just runnng a cheap CSPRNG and getting 256-bit sections of its output, then inventing a bunch of `onion_link_proof` messages with myself and my random number, and pump it at the gossip network.

Not quite. Today nodes store a `node_announcement` if a node has active channels. Going a step further, `lnd` will also prune out any `node_announcement` instance that have no channels after we process the set of closed channels in a block. 

As a result, a `onion_link_proof` (a public one) will only be accepted if both nodes have channels. Clients can opt to filter out further and only accept onion link proofs from a node that has at least `N` channels with `X` BTC total across the channels, etc, etc. 

The onion overlay anchors into the _existing_ LN channel graph, but isn't _embedded_ within it.

-------------------------

roasbeef | 2025-07-01 22:49:08 UTC | #8

> So what you’re saying is that in practice your proposal wouldn’t change the network topology at all? What’s the point, then?

The fact that you arrived at the conclusion after reading my last post shows an impressive ability to make logical leaps. If one carefully reads the OP and my initial reply, they'd arrive at the conclusion that: the onion overlay doesn't necessarily _mirror_ the topology of the existing channel graph. This is due to the fact that: nodes that don't have direct channels with each other can create+advertise an onion link. 

> The spec doesn’t say that it has to be on the same connection, which implies that nodes are free to do whatever they want.
> Also, there’s no need for a separate port for this? You can just open a second connection on the same port.

The [very first BOLT](https://github.com/lightning/bolts/blob/68881992b97f20aca29edf7a4d673b8e6a70379a/01-messaging.md#connection-handling-and-multiplexing) is pretty clear on this point: 
> Implementations MUST use a single connection per peer; channel messages (which include a channel ID) are multiplexed over this single connection.

Again, can you point me to the where in the spec literally any of what you're describing is outlined? 

>  which implies that nodes are free to do whatever they want

Sure, nodes implementation can implement pretty much anything they want to. However, interoperability requires that the behavior be specified so all implementations are on the same page. This is why we write _explicit_ specs. 

> Yes, TCP is all about applying backpressure from an application through to the sender to ensure the sender can do these things without making the recipient unable to respond

You seem to have repeatedly missed the nuance in this example. So here's a simplified scenario that you should be able to understand:
  * Alice has an FIFO internal queue that contains 5 messages: 3x `channel_update`, 1x `commit_sig`, and 1x `onion_message`. 
  * Bob only processes a single message a time, he's entirely single threaded. He's slow, and a single `channel_update` can take him 5 seconds to process. Bob never drops incoming messages, they're processed in order. 
  * Alice sends her first channel update, to Bob, the TCP parameters have been arbitrary restricted s.t only a single message can be outstanding at a time.  
  * In this scenario, only 15+ seconds after the first `channel_update` is sent can Alice send the `onion_message`. 

Can you see how using a distinct TCP (or even [a single QUIC connection](https://github.com/lightning/bolts/issues/1257)!)  connection would allow Alice to send/propagate her `onion_message` more expediently? If not, then I'm not sure we can continue to have a productive conversation on this matter. 

> Head-of-line blocking isn’t some death curse

I'm not presenting it as such, that's just a strawman you seem to enjoy wrangling with. My statement is pretty simple: onion messaging over a distinct TCP eliminates unnecessary blocking, and sidesteps networking+processing contention w/ other node functionality.

-------------------------

gijswijs | 2025-07-02 13:50:39 UTC | #9

Firstly, onion messages today don't have to keep to the same graph topology as the channel graph. The BOLT specifically states:

> Onion messages don't explicitly require a channel, but for spam-reduction a node may choose to ratelimit such peers, especially messages it is asked to forward.

This would allow for a multi-hop message over nodes without any channel between them. But this is a kind of hail Mary approach to sending messages, as the chances of a message being dropped are substantial. Current implementation most likely stick to their current path finding functionality that takes to the channel graph.

More likely, this property of onion messaging is used to fall back to directly connecting to the recipient (with whom you don't have a channel) so that the onion message gets delivered directly.

Secondly, I'm not convinced the resulting onion messaging graph will turn out smaller. (depending on your definition of smaller) For a graph to become smaller (by means of graph diameter) the edge density would have to increase. Meaning that nodes would have to be quite permissive in accepting onion message connections. But if nodes either

- only accept `onion_link_req` from peers with which they have a channel, or
- filter out `onion_link_proof`s from nodes with less thean `N` channels or with less than `X` BTC total capacity,

the resulting graph won't be substantially smaller in diameter, because you are tying the cost of edge creation in the onion message graph (which should be low to be able to get a smaller diameter) to the cost of edge creation in the underlying channel graph (which is high).

I do like the separation of concerns that lies at the foundation of this proposal and I think that warrants further exploration of this topic.

-------------------------

ZmnSCPxj | 2025-07-03 01:13:14 UTC | #10

[quote="roasbeef, post:7, topic:1799, full:true"]
> As I understand it, the DoS issue here is that I can invent any number of “public nodes” by just runnng a cheap CSPRNG and getting 256-bit sections of its output, then inventing a bunch of `onion_link_proof` messages with myself and my random number, and pump it at the gossip network.

Not quite. Today nodes store a `node_announcement` if a node has active channels. Going a step further, `lnd` will also prune out any `node_announcement` instance that have no channels after we process the set of closed channels in a block.

As a result, a `onion_link_proof` (a public one) will only be accepted if both nodes have channels. Clients can opt to filter out further and only accept onion link proofs from a node that has at least `N` channels with `X` BTC total across the channels, etc, etc.

The onion overlay anchors into the *existing* LN channel graph, but isn’t *embedded* within it.
[/quote]

Let us say I invent 10 nodes and create 10 "channels" from my real node to each of my fake nodes.  `lnd` will now treat those 10 fake nodes as "real".  I can then have the 10 nodes claim `onion_link_proof` to each other to create (10 * (10 - 1)) / 2) fake onion links.  That is, for O(n) blockspace and minimum channel size amount, I can push O(n^2) onion links.  Presumably, other nodes who are interested in sending onion messages would need to store those O(n^2) onion links in the very-unlikely case they might send to those nodes.

(We can of course use shortest-path-tree to prune the graph instead of a full graph, but that becomes fragile)

That is precisely why we cannot just invent "overlay" links that are not backed by actual UTXOs on the blockchain layer, and precisely why every channel should have a backing UTXO (or if we want to relax that, we want it to be linear on number of backing UTXOs at worst, not quadratic).  For linear cost to me I can create quadratic cost on everyone else.

We could of course just break that principle, but if we do, we should just allow virtual *full* channels and have an "onion-message-only" flag instead..

In particular, if even if we do heuristics like "the node should have some minimum of M satoshis locked in C channels or else we will drop their `onion_link_proof`" this does not change the fundamental O(n) cost on me vs O(n^2) cost I can impose on others.

-------------------------

roasbeef | 2025-07-08 23:04:27 UTC | #11

> Let us say I invent 10 nodes and create 10 “channels” from my real node to each of my fake nodes

Thanks for this example! I understand the scenario you have in mind now. 

>  That is, for O(n) blockspace and minimum channel size amount, I can push O(n^2) onion links. Presumably, other nodes who are interested in sending onion messages would need to store those O(n^2) onion links in the very-unlikely case they might send to those nodes.

I think the asymptotics here ignore some of the concrete realities that serve to mitigate nuisance attacks like this. 

For one, as this information would be piggy backed along side a `node_announcement`, which has a 65 KB limit like all other messages, the amount of onion links a node can advertise has a hard upper limit. As a result, for node announcements, we already have an upper bound on the amount of storage required per node. 

The creation of each of those initial backing costs also has a direct on-chain cost. Going further, clients that observe/implement this overlay can enforce additional constraints on the minimum eligibility requirements to advertise an onion link for public nodes. An example of such requirements would include: a min channel size, number of onion links per channel, etc. 

> (We can of course use shortest-path-tree to prune the graph instead of a full graph, but that becomes fragile)

Why would the maintenance of a [minimum spanning tree](https://en.wikipedia.org/wiki/Minimum_spanning_tree) be fragile? Such algorithms are routinely used in the domain of computer networking. The textbook algorithms are also relatively efficient ($O(m \log n)$) as far as graph algos go. Can you elaborate on why you think techniques to maintain a minimal graph like MST algos are fragile in this domain? 

Nodes are also free to add nearly arbitrary constraints on how their construct their view of the network. As an example, nodes can set a hard budget on the amount of vertexes to add to such a tree, as there's no need for an onion overlay with say, 100k links. If a node's added constraints impact reachability, then they can just re-run the algo with a higher threshold (they've already downloaded the information via the set of node announcements).

-------------------------

