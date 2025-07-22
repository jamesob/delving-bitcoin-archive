# Disclosure: LND gossip_timestamp_filter DoS

morehouse | 2025-07-22 18:46:25 UTC | #1

*The following disclosure is copied verbatim from a [blog post](https://morehouse.github.io/lightning/lnd-gossip-timestamp-filter-dos/) on [morehouse.github.io](http://morehouse.github.io), reproduced here to facilitate discussion.*

LND 0.18.2 and below are vulnerable to a denial-of-service (DoS) attack involving repeated gossip requests for the full Lightning Network graph.
The attack is trivial to execute and can cause LND to run out of memory (OOM) and crash or hang.
You can protect your node by updating to at least [LND 0.18.3](https://github.com/lightningnetwork/lnd/releases/tag/v0.18.3-beta) or by setting `ignore-historical-gossip-filters=true` in your node configuration.

# Background

To send payments successfully across the Lightning Network, a node generally needs to have an accurate view of the Lightning Network graph.
Lightning nodes maintain a local copy of the network graph that they continuously update as they receive channel and node updates from their peers via a [gossip protocol](https://en.wikipedia.org/wiki/Gossip_protocol).

New nodes and nodes that have been offline for a while need a way to bootstrap their local copy of the network graph.
A common way this is done is to send a [`gossip_timestamp_filter`](https://github.com/lightning/bolts/blob/6c5968ab83cee68683b4e11dc889b5982a2231e9/07-routing-gossip.md#the-gossip_timestamp_filter-message) message to some of the node's peers, requesting that they share all gossip messages they have that are newer than a certain timestamp.
Nodes that cooperate with the message will load the requested gossip from their databases and send them to the requesting peer.

# The Vulnerability

By default, LND cooperates with all `gossip_timestamp_filter` requests.
Prior to v0.18.3, LND's [logic](https://github.com/lightningnetwork/lnd/blob/9380292a5a41697640c2284186c82dad6f7b004f/discovery/syncer.go#L1317-L1384) to respond to these requests looks like this:

```go
func RespondGossipFilter(filter *GossipTimestampFilter) {
  gossipMsgs := loadGossipFromDatabase(filter)

  go func() {
    for msg := range gossipMsgs {
      sendToPeerSynchronously(msg)
    }
  }
}
```

LND loads *all* requested messages into memory at the same time, and then sends them one by one to the peer, pausing after each send until the peer acknowledges receiving the message.
The peer can specify any filter, including one that requests *all* historical gossip messages to be sent to them, and LND will happily comply with the request.
As a result, **LND can load potentially hundreds of thousands of messages into memory for *each* request**.
And since LND has no limit on the number of concurrent requests it will handle, memory usage can get out of hand quickly.

# The DoS Attack

Exploiting this vulnerability to DoS attack a victim is easy.
An attacker simply needs to:

1. Send lots of `gossip_timestamp_filter` messages to the victim, setting the timestamp to 0 to request the full graph.
2. Keep the connection with the victim open by periodically sending pings and slowly ACKing incoming messages.

This causes LND's memory consumption to grow over time, until an OOM occurs.

## Experiment

I carried out this DoS attack against an LND node with 8 GB of RAM and 2 GB of swap.
After a few minutes, the node exhausted its RAM and started using swap, and LND's performance slowed to a crawl.
After about 2 hours, LND exhausted the swap as well and the operating system killed the LND process.

# The Mitigation

LND 0.18.3 added a [global semaphore](https://github.com/lightningnetwork/lnd/commit/013452cff0788289aae3aa296242c698c9beff9d#diff-fd66292d846960a30b8ff5e63dbf15b846fdb6b55afe6dde63c8c2ebca66674dL22-L513) to limit the number of concurrent `gossip_timestamp_filter` requests that LND will cooperate with.
While this doesn't fix LND's excessive memory usage per request, it does limit the global impact on memory usage, which is enough to protect against this DoS attack.

# Discovery

This vulnerability was discovered while looking at how LND handles various peer messages.

## Timeline

- **2023-07-13:** Vulnerability reported to the LND security mailing list.
- **2023-12-11:** Failed [attempt](https://github.com/lightningnetwork/lnd/pull/8030/commits/a242ad5acb6b46e82ef839be84b0695b2de089a7#diff-5321d5dad7ab003eff5e595aef273619fd9c22586f28f1b141940932b93c6ec7) at a stealth mitigation, which could be bypassed by using multiple node IDs when carrying out the attack.
- **2023-12-11:** Emailed the security mailing list again, explaining the problem with the attempted mitigation.
- **2024-08-27:** Proper mitigation [merged](https://github.com/lightningnetwork/lnd/commit/013452cff0788289aae3aa296242c698c9beff9d#diff-fd66292d846960a30b8ff5e63dbf15b846fdb6b55afe6dde63c8c2ebca66674dL22-L513).
- **2024-09-12:** LND 0.18.3 released containing the fix.
- **2025-07-22:** [Gijs](https://github.com/gijswijs) gives the OK to disclose publicly.
- **2025-07-22:** Public disclosure.

# Prevention

This vulnerability has existed ever since gossip filtering was added to LND in 2018.
The [pull request](https://github.com/lightningnetwork/lnd/pull/1106) that added the feature contained over 5k lines of new code and received only minor review feedback.
It seems that no one was thinking adversarially about the new code at that time, and apparently no one has re-evaluated the code since then.

While it's understandable that developers were more focused on building features and shipping quickly in the early days of the Lightning Network, I think it is long overdue that a shift is made to more careful development.
Engineering with security in mind is slower and more difficult, but in the long run it pays dividends in the form of greater user trust and disasters avoided.

# Takeaways

- Update to at least LND 0.18.3 or set `ignore-historical-gossip-filters=true` to protect your node.
- More investment in Lightning security is needed.

-------------------------

Crypt-iQ | 2025-07-22 20:39:23 UTC | #2

Nice work finding this.

[quote="morehouse, post:1, topic:1859"]
**2023-12-11:** Failed [attempt](https://github.com/lightningnetwork/lnd/pull/8030/commits/a242ad5acb6b46e82ef839be84b0695b2de089a7#diff-5321d5dad7ab003eff5e595aef273619fd9c22586f28f1b141940932b93c6ec7) at a stealth mitigation, which could be bypassed by using multiple node IDs when carrying out the attack.
[/quote]

Just to set the record straight, this was my mistake even though the commit attribution says otherwise.

-------------------------

