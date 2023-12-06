# Quick peer gossip catchup protocol

rustyrussell | 2023-12-05 03:22:20 UTC | #1

Hi all!  I've been reworking CLN's gossip daemon, and it is clear that we can fairly easy save a little information on a per-peer basis and send them all the gossip which we've received since they disconnected (minus a little bit, probably).

I'm thinking of it as a new feature "option_serves_gossip_catchup".  The peer would send the gossip_timestamp_filter with an odd TLV field indicating it wanted gossip since last time, though I suspect we'll want a new message to say "cannot serve from last time" if (say) the peer doesn't have a channel, or dataloss; in this case just stream gossip from now on, if they want the entire gossip then they simply re-send gossip_timestamp_filter with 0 as time.

Implementing this for us simply means tracking the last gossip_store offset we sent, and adjusting it when we compact (on restart).  This requires some work and storage per peer, so we may want to restrict it to those which have channels with us.

As an addition, we could prioritize "important" gossip (new channels of significance), though that's harder to specify.

-------------------------

MattCorallo | 2023-12-05 05:49:50 UTC | #2

Can we accomplish this without any per-peer storage? Probably something like "the last time I was online, running, and received gossip was block header/timestamp X". From there any peer that does receive-time tracking should be able to go back into their gossip history and check for everything newer than said time (minus a fudge factor).

Of course that assumes nodes keep track of when they received all gossip messages. Currently we only track that for channel announcements, not updates. TBH I'd really rather we Just Do Minisketch and move on rather than trying to keep applying bandaids to the current gossip.

-------------------------

rustyrussell | 2023-12-06 05:06:18 UTC | #3

> TBH I’d really rather we Just Do Minisketch and move on rather than trying to keep applying bandaids to the current gossip.

Sure, but you need gossip v1.5 for minisketch.  And you still might want this for catchup, if you only have a few peers (i.e. one peer!).

Meanwhile, I thought "no, your proposal is just like gossip_timestamp_filter which we're moving away from", but actually, /it's different/, being based on the gossip time, not the timestamps.

If you record the timestamp when you commit the gossip, assuming 60 second relay, you're pretty close.  If the peer reports when they last received gossip you can subtract 120 seconds and probably do a pretty good job.

It turns out, we even have a field in our gossip_store header "timestamp", which is redundant (it used to be a copy of the update timestamp, used for easy timestamp filter when we supported that).  We use that without a format change.

We'd still need to keep some offset markers to avoid having to sweep the entire file when asked for a specific time, but that's actually extremely implementable...

-------------------------

cdecker | 2023-12-06 13:23:54 UTC | #4

Fwiw this sounds very close to what `lnsync` does. It assigns an arbitatry ordering to the messages and allows clients to seek through that ordering. Since the ordering only has to make sense to the serving node, and any misordering can only result in a bit too much being sent across this seems like a trivial and backwards compatible solution to quickly sync up.

While the idea of canonical ordering by the serving node works quite nicely, we can notice that we also have a partial order among messages, with the `channel_announcement` being the outlier, since it cannot be uniquely ordered among the other messages that may pertain to the channel or the endpoints.

 - `channel_update` sort by timestamp, and allow querying a range based on these timestamps
 - `node_announcement` sort by timestamp, and allow querying a range based on these timestamps
 - `channel_announcement` does not have a timestamp, and ought to be included if the querying node is asking for a timestamp range that include ANY update to the channel

As you can see the only potential over-sharing is in the form of the `channel_announcement` but I don't think that is solvable, since there is no unique position in the ordering that'd guarantee inclusion iff a matching `channel_update` is included. 

This allows us to avoid per-peer storage: on startup we load the timestamp when we were last online, we then subtract a buffer from that (1/2h?) and we tell our peers that timestamp when reconnecting. This means we don't store anything per-peer, rather we only store a timestamp up to which we believe we are up to date, and then ask for incremental diffs from that point onwards.

-------------------------

rustyrussell | 2023-12-06 20:01:08 UTC | #5

This is, in fact, what gossip_timestamp_filter does:

```
1. type: 265 (`gossip_timestamp_filter`) (`gossip_queries`)
2. data:
    * [`chain_hash`:`chain_hash`]
    * [`u32`:`first_timestamp`]
    * [`u32`:`timestamp_range`]
```

The "timestamp" for channel_announcement is defined as " the `timestamp` of a corresponding `channel_update`." (rather than "all corresponding channel_update" which is what you specified).

It turns out, however, that everyone uses "none", "all" or "recent", where recent is 10 minutes ago (CLN), now (LND and Eclair), 1 hour ago (LDK).  To avoid scanning the entire gossip_store (or keeping more metadata), we now keep a single "first record with timestamp > 2 hours ago" offset and scan from there, unless they set first_update to 0.  This basically turns the field into a trinary.

But this is *still* not what a single-peer-gossiper wants!  The timestamp of the record is a very rough guide to "have you seen this before".  A better guide is when the *peer* received the gossip: presumably it started streaming it within 60 seconds.  It's also much easier for us to implement: for speed we can keep the received timestamps at every ~1MB of gossip store, since they will be monotonic.

-------------------------

