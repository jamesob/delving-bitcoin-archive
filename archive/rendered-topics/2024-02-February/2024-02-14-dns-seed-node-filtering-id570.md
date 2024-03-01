# DNS seed node filtering

virtu | 2024-02-14 14:23:24 UTC | #1

Although this is somewhat of follow-up on [PR29149](https://github.com/bitcoin/bitcoin/pull/29149), I did not want to revive the discussion on GitHub because by the time the PR was closed I was under the impression that the PR's supporters weren't really interested in a discussion but were deliberately misconstruing [DNS seed policy rules](https://github.com/bitcoin/bitcoin/blob/baed5edeb611d949982c849461949c645f8998a7/doc/dnsseed-policy.md).

Despite this, I came away from the PR wanting to know how widespread the filtering was. Because other people have expressed interest in the data or suggested monitoring node versions permanently, I wanted to share my findings here.

The figure below shows the Bitcoin Core versions run by the reachable nodes advertised by individual DNS seeds on a monthly basis. Note that node releases prior to 22.0 that use the 0.major.minor versioning scheme had their leading zero stripped to make the graph more readable. Daily updated monitoring data is available [here](https://21.ninja/dns-seeds/node-version-range/).

![comparison-version-minmax|690x459](upload://lLrgKcsbNmzIZDPZwpDrmro95ow.png)

The data clearly show that although `dnsseed.bitcoin.dashjr.org` is the most strict in terms of filtering out old versions, `dnsseed.bluematt.me` and `seed.bitcoin.wiz.biz` also employ version-based filtering.

The minimum version encountered from `seed.bitcoin.jonasschnelli.ch`, `seed.bitcoin.sprovoost.nl`, and `seed.btc.petertodd.net` during the entire data-collection interval was 0.9.0, while that from `seed.bitcoin.sipa.be`, `dnsseed.emzy.de`, and `seed.bitcoinstats.com` was 0.8.6. Although this could hint at filtering performed by the first group, it might as well just be coincidence (my guess is there weren't a lot of reachable nodes running 0.8.6 during the data-collection interval and despite the first group knowing about these nodes, they just never got selected).

-------------------------

1440000bytes | 2024-02-14 15:39:37 UTC | #2

[quote="virtu, post:1, topic:570"]
The data clearly show that although `dnsseed.bitcoin.dashjr.org` is the most strict in terms of filtering out old versions, `dnsseed.bluematt.me` and `seed.bitcoin.wiz.biz` also employ version-based filtering.
[/quote]

Thanks for sharing this although I do not agree that I was 'deliberately' misconstruing DNS seed policy rules.

-------------------------

cdecker | 2024-03-01 10:38:03 UTC | #3

bitcoinstats.com will return nodes in the same order the crawler scans them, as that's the most recently seen working peer giving me the highest confidence that the requester will actually be able to connect to it. From there I expect `bitcoind` to use the `getaddr`/`addr` mechanism to fill their address manager further, obviating future DNS bootstrapping.

Remember that DNS seeds are not intended for day-to-day peer lookups, but rather are a bootstrapping solution for completely new nodes. As long as there is any working node among the 25 seeds usually return you will be able to join the network, independently of the version they return.

Honestly reading your post I was wondering if I should implement a bit of sanity filtering too, and what the negative impact of that might be (old versions getting fewer incoming connections).

-------------------------

mzumsande | 2024-03-01 18:12:35 UTC | #4

I also don't think there is anything wrong with filtering extremely old nodes at the discretion of the operators - but it's weird and probably a bug if a DNS seed wouldn't ever return nodes with a current and widely used version, which was the case in the linked issue (and quickly fixed).

In my opinion, the most important benefit of gathering this kind of statistics isn't so much evaluating / comparing the quality of individual seeds, but to hopefully make it visible if something abruptly changed in one or more of those - that could be a sign that this seed got compromised.

-------------------------

