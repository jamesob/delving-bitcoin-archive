# Disclosure: DoS vulnerabilities fixed in Eclair v0.14.1

erickcestari | 2026-10-01 18:50:37 UTC | #1

Two denial-of-service vulnerabilities in Eclair's channel opening flow were fixed in Eclair v0.14.1. Users should upgrade to v0.14.1 or later to protect their nodes.

### LNF-2026-0003: open_channel race DoS

Eclair v0.14.0 and earlier reject a duplicate `temporary_channel_id` by consulting the peer's channel map to see if the `temporary_channel_id` already corresponds to another channel.  But there is a delay between this check and the later insertion of the new channel into the map.  An attacker that pipelines several identical `open_channel` messages can slip two of them past the check, causing Eclair to spawn two channel actors for one `temporary_channel_id`.  The second actor overwrites the first in the channel map, leaving an **orphaned** channel actor that holds roughly **25 KB** of heap until the peer disconnects.  An attacker repeatedly exploiting the race causes Eclair to leak about **1 MB per second**, eventually causing a JVM out-of-memory crash or a garbage-collection death spiral that pegs every core and takes the node off the network.

Matt Morehouse found this with smite while fuzzing Eclair's funding flow.  smite flagged that Eclair would sometimes accept two `open_channel` messages with the same `temporary_channel_id`. Further investigation surfaced the race.

More details in the [lnfuzz advisory](https://lnfuzz.org/advisories/eclair-open-channel-race-dos/).

### Unfunded channel flood

In Eclair v0.14.0 and earlier, the `PendingChannelsRateLimiter` is intended to prevent [fake channel DoS](https://morehouse.dev/lightning/fake-channel-dos/) attacks by limiting the number of unconfirmed channels that are in flight at any time. But an attacker can bypass the rate limiter by reusing each channel's final id as the `temporary_channel_id` of the next `open_channel`. The `Peer` duplicate check only looks at temporary ids, so it accepts the reused id, and the rate limiter ends up tracking that id twice. When the new channel gets its final id, the rate limiter removes both entries at once, so its count never goes above 2 while the real number of pending channels grows without bound. Since every channel stores a copy of the peer's `init` features, inflating them to about 65 KB makes each channel take about 65 KB. A single connection, at no on-chain cost, ran a node with a 4 GB heap out of memory in about 48 minutes. Because the channels are persisted, the node crashed again on every restart until someone raised the heap or deleted the rows by hand.

I found this with an LLM-based harness that maps a codebase's entry points and the invariants each should hold, then hunts for violations using the BOLTs as reference. It flagged the channel id collision; I confirmed it and built the proof of concept that drove the node to OOM.

More details in the [blog post](https://erickcestari.dev/blog/eclair-oom-pending-channels/).

-------------------------

Anzus_GemWallet | 2026-10-06 03:26:40 UTC | #2

Thanks for publishing this. For someone using a mobile Lightning wallet rather than operating an Eclair node themselves, is any action required from the wallet user, or is upgrading entirely the responsibility of whoever operates the Eclair node or backend? Clarifying that distinction may help less technical users understand whether they are affected.

-------------------------

