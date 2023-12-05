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

