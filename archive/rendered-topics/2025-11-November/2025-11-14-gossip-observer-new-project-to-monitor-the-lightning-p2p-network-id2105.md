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

