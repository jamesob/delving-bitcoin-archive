# Gossip Observer: New project to monitor the Lightning P2P network

jonhbit | 2025-11-14 20:48:38 UTC | #1

For the past few months (since plebfi miami), I've been working on a project to monitor the Lightning gossip network by collecting gossip messages from numerous nodes. The repo with the code used, the raw data collected before lightning++, and the slides from my talk is here:

<https://github.com/jharveyb/gossip_observer>

## Observations:

- Convergence delay (time for a message to propagate around the network) has dropped a significant amount since similar measurements in 2022, from ~500 to ~200 seconds for 75% propagation. I suspect this is due to LN implementations defaulting to making more P2P connections.

- A significant subset of all messages received were sent to my node by less than 1/4th of its peers. This could be due to a poorly connected graph of P2P connections, or some filtering policy in the LN implementations.

- For channel_update messages, which were 60% of total messages, 20% of channels had more than 144 messages. This is relevant when compared to the proposed gossip rate limit for Taproot gossip / gossip 1.5(?):

<https://github.com/lightning/bolts/pull/1059>

- For node_announcement messages, which were 30% of total messages, 2.5% of nodes were announced more than 144 times. This seems like a unintended behavior of some implementation or node automation tool.

- The total size of all unique messages collected over that day was 103.2 MB, which is greater than I expected. This does not account for the bandwidth overhead of receiving the same message from multiple peers, so node bandwidth usage is likely much higher.

## Future Work

So far, I only collected data over a 24 hour period and analyzed that. In the near term I'll be setting up permanent infrastructure to receive gossip from multiple geographic regions and neighborhoods in the P2P graph. I'll also be adding functionality to broadcast gossip messages from these nodes, and observe their propagation. Work will continue in the gossip-observer repo linked above.

## Getting Involved

If you have feedback on interesting metrics to compute, a request to track gossip propagation from your node, or anything else, feel free to message me here!

I'm also investigating how this raw data could be published so others can perform their own analysis. I'm particularly interested in options for unsupervised anomaly detection, but I'm also definitely not a data scientist.

-------------------------

jonhbit | 2025-11-14 21:48:21 UTC | #2

A related subject is how we could switch the LN P2P network from message flooding to something closer to the design outlined in the Erlay paper and BIP. Some observations:

- Latency / propagation delay is much less important for LN gossip; implementations will allow slightly outdated fees to apply to their channel, for example.

- Gossip messages are signed by the sender, and are intended to be public. So the propagation method does not need to conceal information about the sender's position nor connections in the LN P2P graph.

There has been some existing work on this subject:

<https://endothermic.dev/p/magical-minisketch>

### Adapting Minisketch to LN Gossip

A key detail is how to map gossip messages to Minisketch set elements. This mapping should be collision-resistant, and ideally the elements are short, since reconciliation time is quadratic wrt. element length.

In the Erlay design, collisions are avoided by generating a salt for each peer connection, and using that and the TXID as input to a fast non-cryptographic collision-resistant hash like SipHash. This means that the total amount of hashing operations needed scales with the number of peer connections, but having a shorter (32-bit) set element greatly reduces the compute needed for set construction and reconciliation.

Alternatively, set elements could be computed from a channel's short channel ID (64 bits), plus the block height included in the gossip message. However, there are some issues with this approach.

One is that the components of a short channel ID (channel open block height, transaction index, and output index) don't have a lot of entropy. For example, in a full block, one could have ~36000 1-in 1-out P2TR TXs of 111 vB, or 1 TX with 1 input and 93020 outputs with weight ~3999900 vB. In either case, the entropy of the TX + output index is only ~17 bits. Mixing in block heights does not add much to the total entropy, since a message can only use a block height from the last two weeks (~2016 possible values, 11 bits of entropy).

In addition, we would need a different mapping for node_announcement messages.

Given that LN nodes already store some per-peer state on connection (re)establishment, I'm in favor of borrowing the Erlay approach and using a per-peer salt + a hash like SipHash to generate short Minisketch set elements for each gossip message.

### Gossip v1.5 message format

Another detail is that the gossip 1.5 proposal allows for optional fields in gossip messages, which is not possible with current gossip messages:

<https://github.com/lightning/bolts/pull/1059>

However, I think peers can decide which fields to send to their peer based on their announced feature bits.

### Other considerations

Compared to the Erlay design, I don't think LN implementations would need to perform any message flooding at all. Combining limited flooding and set reconciliation is useful for reducing total propagation delay, but the LN gossip propagation delay is so large that it could be maintained through set reconciliation alone. This should greatly simplify implementation complexity as well.

With respect to implementation, one problem would be different nodes filtering messages with different policies. This would lead to persistent set differences, which would waste bandwidth and, in the worst case, cause reconciliations to fail. I suspect that these filtering policies are either legacy behavior, or adaptions to quirks in existing gossip behavior, and could largely be removed if we move to set reconciliation.

### Future Work

In parallel to the gossip monitoring work, I'd like to get feedback from LN implementers on which designs could work for them. Is the per-peer state a significant drawback or source of complexity? Would there be an issue with more frequent (every 30 seconds?) set reconciliation and gossip message handling? Are there other issues that haven't been considered?

I'll update this thread if/when I have a more concrete proposal or draft BIP.

-------------------------

