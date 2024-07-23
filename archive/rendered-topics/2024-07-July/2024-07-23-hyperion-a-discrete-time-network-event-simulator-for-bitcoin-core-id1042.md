# Hyperion: A discrete time network event simulator for Bitcoin (Core)

sr-gi | 2024-07-23 19:56:21 UTC | #1

A few months ago I started working on reviving [Erlay](https://github.com/bitcoin/bitcoin/pull/30116) mainly motivated by the apparent bandwidth improvements it will bring to Bitcoin nodes and seeing how the project seemed to have been put on hold, potentially after it hasn't gotten the sufficient reviewers attention to make it trough.

Revisiting the proposed PR led to some changes in the design, mainly regarding how fanout peers should be picked:
- On a transaction basis or on a connection basis
- Based on whether the peer is inbound or outbound
- Proportional to or independent of the number of peers
- ...

Given some of the changes introduced (or considered) more simulations needed to be run to understand the tradeoffs and fine-tune the parameters. This led to building a network simulator that could compare the traditional inv->getdata->tx flow with the transaction reconciliation flow introduced by Erlay.[^1][^2] 

## Hyperion

https://github.com/sr-gi/hyperion

Hyperion is an event-based simulator using discrete time. A network of nodes is built, deterministically at random, given a number of reachable and unreachable nodes. Once nodes are connected to each other, a transaction is created from a random node and broadcast to its peers. This triggers a series of events that will spread the transaction across the network until every single node is reached (assuming the network is not partitioned).

Nodes in the network are well-behaved, meaning they stick to the `inv->getdata->tx` flow, do not announce transactions to peers more than once (nor to a peer that has announced it to them), do not send out a transaction more than once, do always reply to `GETDATA` messages is requested, but won't request multiple `GETDATA` for the same transactions, etc.

The transaction relaying logic follows Bitcoin Core's design (w.r.t privacy/DoS protection):

* `INVs` are sent to peers on random intervals based on whether a peer is inbound or outbound:
  * For inbounds, the delay follows a Poisson process with an expected value of `INBOUND_INVENTORY_BROADCAST_INTERVAL(5s)`. All inbounds are on the same timer
  * For outbounds, the delay follows a Poisson process with an expected value of `OUTBOUND_INVENTORY_BROADCAST_INTERVAL(2s)`. Every outbound has a unique timer
* `GETDATA`s are prioritized to outbound peers, hence if an inbound peer announces a transaction, the request will be delayed by `NONPREF_PEER_TX_DELAY(2s)`, and superseded by any other request by an outbound peer.

## Simulations

Here's an example of a (non verbose) simulation in a network with 10K reachable nodes and 100K unreachable, sending a single transaction from a random node:

```cmd
> hyperion --reachable 10000 --unreachable 100000

2024-07-23T19:31:02.327Z INFO  [hyper_lib::simulator] Using fresh rng seed: 16441876165305951121
2024-07-23T19:31:02.341Z INFO  [hyper_lib::network] Creating nodes (110000: 10000 reachable, 100000 unreachable)
2024-07-23T19:31:02.341Z INFO  [hyper_lib::network] Connecting unreachable nodes to reachable (800000 connections)
2024-07-23T19:31:02.549Z INFO  [hyper_lib::network] Connecting reachable nodes to reachable (80000 connections)
2024-07-23T19:31:02.588Z INFO  [hyperion] Starting simulation: broadcasting transaction (txid: 1c832fdc) from node 82519
2024-07-23T19:31:04.787Z INFO  [hyperion] Reachable nodes sent/received 48.4265/70.5934 messages (4926.208/4483.7017 bytes) (avg)
2024-07-23T19:31:04.787Z INFO  [hyperion] Unreachable nodes sent/received 6.15997/3.94328 messages (384.23712/428.48776 bytes) (avg)
2024-07-23T19:31:04.787Z INFO  [hyperion] Transaction (txid: 1c832fdc) reached 90% of nodes in the network in 10.027488s
```

## State of the project

The project currently supports the traditional transaction workflow, and I'm working on implementing the Erlay-related functionality

Features, requests, comments, and suggestions are more than welcome. Feel free to try it out and let me know if you find it helpful or what is missing and/or wrong.

[^1]: Q: Why not use the existing simulator that was used for previous Erlay simulations? 
A: Because it is a general-purpose simulator adapted to work with Bitcoin and written in Java, which may be overkill for the goal, and I'm not willing to keep modifying and maintaining
[^2]: Also [RIIR](https://github.com/ansuz/RIIR)

-------------------------

