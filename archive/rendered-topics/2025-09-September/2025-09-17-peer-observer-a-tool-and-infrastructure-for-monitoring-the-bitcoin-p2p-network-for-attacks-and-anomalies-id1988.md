# Peer-observer: A tool and infrastructure for monitoring the Bitcoin P2P network for attacks and anomalies

0xB10C | 2025-11-12 14:52:50 UTC | #1

I've written about my peer-observer project in https://b10c.me/projects/024-peer-observer/ and post about it here for me to share project updates, enable discussion about the project, and for readers to leave ideas further monitoring ideas.

Since the project relies on  "honeypot nodes", the actual web front end isn't publicly accessible (though I've set up accounts for a bunch of people over the last years) as the data would make the nodes identifiable. On [public.peer.observer](https://public.peer.observer) a list of the nodes and their configurations can be found. Similarly, the fork-observer instance connected to the nodes is also [publicly reachable](https://public.peer.observer/forks/).

I do however plan to set up a separate demo instance with public dashboards and data (and thus IPs, accepting the fact that some people might see this as invitation to mess with the nodes) soon.

As for code, the tooling can be found on [github.com/0xB10C/peer-observer](https://github.com/0xB10C/peer-observer), and I have a NixOS package and module in [github.com/0xb10c/nix](https://github.com/0xb10c/nix/) that I use to run the tooling. The (opinionated) infrastructure configuration for deploying, managing and connecting the tools to e.g. a Prometheus and Grafana instance, debug.log rotation, addrman-observer, ... isn't public yet, but I hope to publish this with or after the demo set up to enable others to run their own set up similar to mine (to be clear: running this yourself is entirely possible right now, but publishing my infrastructure configuration should make it easier to replicate my setup elsewhere).

-------------------------

0xB10C | 2025-09-17 12:48:29 UTC | #2

Initially, peer-observer did only extract data from the tracing / eBPF interface. The [ebpf-extractor](https://github.com/0xB10C/peer-observer/tree/master/extractors/ebpf) hooks into the tracepoints and passes the events on to tools which then process these events (e.g. create prometheus metrics, publish them as JSON via a websocket for web visualizations, ... ). This works well for everything that needs realtime events.

To supplement the real-time event data, I [added](https://github.com/0xB10C/peer-observer/pull/191) an [RPC-extractor](https://github.com/0xB10C/peer-observer/tree/master/extractors/rpc) in August with `getpeerinfo` have staleful data about the connected peers that we can't get from the tracepoints alone. For example:
- how many connections to spy nodes or nodes on a banlist does the node have?
- what share of connections are connected via BIP324 v2 transport connections?
- how does the mean/median Bitcoin protocol ping to my connections change over time?
- how many peers [relay sub-1sat/vbyte transactions](https://github.com/0xB10C/peer-observer/issues/243)?
- ...

While only `getpeerinfo` is implemented for now, there are a bunch of other RPCs that would be useful to have in there. A few examples are listed in https://github.com/0xB10C/peer-observer/issues/199 and I also want to explore how to add WIP `getpeerinfo` fields like `cpu_load` in there https://github.com/0xB10C/peer-observer/issues/200.

Recently, I've been thinking about how to effectively detect P2P DoS attacks or anomalies (i.e. bugs). While I run a [process-exporter](https://github.com/ncabatoff/process-exporter) to collect data on how much time is spent in e.g. the `b-msghand` thread, an alternative might to also track the time it takes for the node to respond to a ping via the P2P network (https://github.com/0xB10C/peer-observer/issues/212). This has been a good DoS indicator in https://b10c.me/observations/15-inv-to-send-queue/#effect since pings are handled in queue with all other messages. It measures processing backlog and network latency. For this, I've started working on a p2p-extractor that frequently pings the node from localhost (to minimize network latency) and publishes the time it takes for a pong to arrive. This can then be used in alerting.


As part of https://github.com/0xB10C/peer-observer/issues/141, I've also been thinking about a log-extractor similar to the one used in [bmon](https://github.com/chaincodelabs/bmon). However, I'll probably first explore an IPC-based extractor - that might possibly even replace the ebpf / tracing extractor as it should resolve some of the painpoints of the eBPF based tracing interface (see https://github.com/bitcoin-core/libmultiprocess/issues/185 and https://github.com/bitcoin/bitcoin/pull/32898).

---

In other news, I've recently added a Knots node called `nico` to my infrastructure (the others are all Bitcoin Core). Since people are using it, it  makes sense to include it in the monitoring too.

-------------------------

0xB10C | 2025-10-17 16:22:08 UTC | #3

I've set up a demo instance of peer-observer on [demo.peer.observer](https://demo.peer.observer) with two nodes `hal` and `len` and opened it up for full public access. Feel free to explore! Huge thanks to https://lclhost.org/ for sponsoring the servers!

The NixOS infra definition can be found in [https://github.com/0xB10C/peer-observer-infra-demo](https://github.com/0xB10C/peer-observer-infra-demo) which uses https://github.com/0xB10C/peer-observer-infra-library under the hood.

-------------------------

0xB10C | 2026-04-27 14:19:09 UTC | #4

It's been a while, so I figured I'll give an update on what's changed in peer-observer. To recap, peer-observer extracts events from a Bitcoin Core (or software-fork) node and has a few tools that process and show them. The goal is to detect anomalies (i.e. bugs) and attacks against honeypot nodes.

On the extractor side:

### p2p-extractor

I implemented a custom P2P client that the node connects to via `-addnode` (on localhost) called `p2p-extractor`. This allows us to do the following measurements:

- message backlog: measure the time it takes the node to process a ping and respond to a pong message. When the node is under high load (slow-to-validate block, DoS-attack, large inv-to-send sets as in e.g. https://bnoc.xyz/t/increased-b-msghand-thread-utilization-due-to-runestone-transactions-on-2026-02-17/81), it will take the node a while to respond to us with a pong. Since we are on localhost, we assume the network latency is zero.
- addr announcements: The p2p-extractor receives addrv2 messages from the node and we can, for example, deduce the rate of addresses relayed per hour and per addrv2 message.
- inv announcements: The p2p-extractor receives inv messages from the node and we can calculate an average inv-size, the WTx inv announcement rate per second, and also the INV rate per second.
- feefilter changes: Collecting feefilter information allows us to see when and how often the node updates its fee filter.


### log-extractor

While parsing log messages from the human-readable `debug.log` isn't really a stable interface, it still can be used to supplement with log-based events. @m4ycon [implemented](https://github.com/peer-observer/peer-observer/pull/272) a regex-based log-extractor. This inspired some discussion about better ways of extracting known log messages from the source code and using log-parsing algorithms in https://github.com/peer-observer/peer-observer/issues/336. Additionally, having structured, e.g. JSON based, logging in Bitcoin Core came up too. Currently, there's work being done to implement compact block reconstruction tracking and timing measurements based on the log messages.


### rpc-extractor

[GuiSchet](https://github.com/GuiSchet) and @deadmanoz helped implement a bunch of new RPCs to the rpc-extractor. Currently, the RPC extractor fetches `getpeerinfo`, `getmempoolinfo`, `uptime`, `getnettotals`, `getmemoryinfo`, `getaddrmaninfo`, `getchaintxstats`, `getnetworkinfo`, `getblockchaininfo`, `getorphantxs`, and `getrawaddrman`. Personally, I found `getorphantxs` and `getrawaddrman` to be the most interesting ones we added. All these RPCs are fetched regularly and the response is published as an event.

- `getorphantxs`: Allows us to have an overview over e.g. the size of the orphanage (after https://github.com/bitcoin/bitcoin/pull/31829) - while I have some basic orphan DoS stats, there's [issue #350](https://github.com/peer-observer/peer-observer/issues/350) to iterate on this at some point to go into more detail on orphanage metrics.
- `getrawaddrman`: Provides us with insights into the addrman of the nodes. We can keep track of service bit usage, port usage, etc over time. Additionally, comparing two getrawaddrman responses a few minutes apart allow us to calculate a rate at which we add or replace entries in the `new` and `tried` tables. We have metrics for this, but no dashboards yet.

### ipc-extractor

@xyzconstant has been working on an IPC extractor connecting to the Bitcoin Core IPC interface in https://github.com/peer-observer/peer-observer/pull/379. It's currently built against Bitcoin Core v31.0 and is mainly a minimal proof-of-concept on how to extract data from the IPC interface. Once merged, it can be used to review, test and give feedback on https://github.com/bitcoin/bitcoin/pull/29409  (see https://github.com/peer-observer/peer-observer/issues/370). Additionally, a mid-term goal could be to have a dedicated IPC tracing interface in Bitcoin Core as discussed in https://github.com/bitcoin/bitcoin/issues/35142 to replace the eBPF / USDT interface to reduce some of the current tracing pain points.

---

On the tooling side: 

### archiver-tool

@octaviolucca has been working on a tool that archives all events (or a filtered set of events) to a compressed archive in https://github.com/peer-observer/peer-observer/pull/373. These archives can then be used in future analysis when deeper inspection of events is required. This includes a replayer which allows to replay events. 


### metrics-tool anomaly detection

[RazorBest](https://github.com/RazorBest) has been looking into Prometheus based Anomaly detection in https://github.com/peer-observer/peer-observer/pull/400 which has been on my wish-list for a while. 

### alerts-tool

Next to alerting on automatically detected anomalies, we can also come up with a few heuristics we want to alert on. I started to list some in https://github.com/peer-observer/peer-observer/issues/185 and GuiSchet has been working on an initial implementation in https://github.com/peer-observer/peer-observer/pull/383.

---


Next to features, there also has been a bunch of work on fixing intermittent test failures, cleaning up the code here and there, and keeping the demo and production monitoring infrastructure running. I'm happy to see so many new contributors joining.


The current goal is to get a "version 1.0" out at some point with the above mentioned extractors and tools implemented and polished a bit. This should give a good base and having somewhat good coverage on the passive P2P monitoring side.

-------------------------

