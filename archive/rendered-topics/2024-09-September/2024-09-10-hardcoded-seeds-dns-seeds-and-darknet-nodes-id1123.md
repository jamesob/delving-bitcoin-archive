# Hardcoded seeds, DNS seeds and Darknet nodes

virtu | 2024-09-10 08:42:24 UTC | #1

Since darknet-only nodes cannot use DNS seeds[^1] to learn about addresses of Bitcoin nodes to connect to, I wanted to get a feel on how such nodes fare on hardcoded-seed data. To this end, I created some statistics on hardcoded seed nodes[^2] to investigate how fast the number of reachable seeds decreases over time.

![hardcoded_seeds|690x343, 100%](upload://dBVnqgb1Pp2mnUNtdZx7haPAyWt.png)

Each chart shows the number of reachable hardcoded seeds over time for a particular network type (I excluded Cjdns because I've only been collecting this data for a couple of weeks) and includes data for the last four Bitcoin Core releases (red lines correspond to release dates).

Some observations:
- Despite IPv4 appearing to be the most "short-lived" network type, of the seed nodes hardcoded around 21 months ago into v24 still ~50 are reachable.
- I2P nodes seem to be the most "long-lived" ones (the sharp drop shortly after v27 release in April 24 was due to a [DoS attack on the I2P network](https://geti2p.net/de/blog/post/2024/04/25/stormy_weather)).
- It looks like there was an oversight during the v26 release to include new seed nodes.
- Before v27, the number of Onion and I2P seed nodes was rather low (nodes for these networks were manually curated up until recently, and the number of nodes included depended on the data source).

To answer my original question about darknet-only nodes: they should be doing fine. Still I wondered, why they're excluded from taking advantage of the DNS seed-mechanism, which has several advantages: for one, DNS seeds provide more up-to-date on reachable nodes, increasing the probability of quickly finding a reachable address; for another, they might improve privacy by advertising from a larger pool of nodes, thus reducing the likelihood of a hardcoded seed collecting statistics about bootstrapping nodes; etc.

In an old GitHub comment I read that darknet addresses are too large for seeding, it would be unnecessary and it's not practical to access DNS over these networks. I'd like to address the first and last points with a [PoC darknet seeder](https://github.com/virtu/darkseed) I wrote which is capable of serving Onion, I2P and Cjdns addresses using a BIP155-like encoding and is reachable via IPv4 and Cjdns (DNS/UDP) as well as Onion and I2P (DNS/TCP).

The point about necessity is one I don't feel qualified to answer. If most people are running mixed clearnet-darknet nodes, there's no necessity. For darknet-only nodes, it might be worth the effort (~100 loc to create custom DNS `NULL` queries in Bitcoin Core since right now we conveniently use `getaddrinfo` which does not require any low-level DNS functionality). Happy about feedback before taking this any further!

[^1]: For those who don't know, DNS seeds are used by a new Bitcoin node as the default way to learn about Bitcoin nodes it can connect to. The node sends a DNS query to one or more of the DNS seeds whose addresses are hardcoded into the binary, and receive a DNS reply containing a number of IPv4 and IPv6 addresses of nodes believed to be reachable. The node then connects to one or more of these addresses, sends a `getaddr` message to the node it connected to and (ideally) receives an `addr` reply from the node containing around 1000 addresses of other Bitcoin nodes.
[^2]: For those in need of a refresher, hardcoded seeds are used as a fallback when a new node who doesn't know about any peers fails to solicit peer addresses via DNS seeds; for such instances, the Bitcoin Core binary contains a number of hardcoded addresses which the node can connect to and ask for other nodes' addresses by sending a `getaddr` message.

-------------------------

