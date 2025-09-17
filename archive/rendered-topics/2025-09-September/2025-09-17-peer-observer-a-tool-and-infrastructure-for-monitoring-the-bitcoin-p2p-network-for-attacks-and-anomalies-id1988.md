# Peer-observer: A tool and infrastructure for monitoring the Bitcoin P2P network for attacks and anomalies

0xB10C | 2025-09-17 12:08:29 UTC | #1

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

