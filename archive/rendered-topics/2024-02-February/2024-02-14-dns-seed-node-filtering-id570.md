# DNS seed node filtering

virtu | 2024-02-14 14:23:24 UTC | #1

Although this is somewhat of follow-up on [PR29149](https://github.com/bitcoin/bitcoin/pull/29149), I did not want to revive the discussion on GitHub because by the time the PR was closed I was under the impression that the PR's supporters weren't really interested in a discussion but were deliberately misconstruing [DNS seed policy rules](https://github.com/bitcoin/bitcoin/blob/baed5edeb611d949982c849461949c645f8998a7/doc/dnsseed-policy.md).

Despite this, I came away from the PR wanting to know how widespread the filtering was. Because other people have expressed interest in the data or suggested monitoring node versions permanently, I wanted to share my findings here.

The figure below shows the Bitcoin Core versions run by the reachable nodes advertised by individual DNS seeds on a monthly basis. Note that node releases prior to 22.0 that use the 0.major.minor versioning scheme had their leading zero stripped to make the graph more readable. Daily updated monitoring data is available [here](https://21.ninja/dns-seeds/node-version-range/).

![comparison-version-minmax|690x459](upload://lLrgKcsbNmzIZDPZwpDrmro95ow.png)

The data clearly show that although `dnsseed.bitcoin.dashjr.org` is the most strict in terms of filtering out old versions, `dnsseed.bluematt.me` and `seed.bitcoin.wiz.biz` also employ version-based filtering.

The minimum version encountered from `seed.bitcoin.jonasschnelli.ch`, `seed.bitcoin.sprovoost.nl`, and `seed.btc.petertodd.net` during the entire data-collection interval was 0.9.0, while that from `seed.bitcoin.sipa.be`, `dnsseed.emzy.de`, and `seed.bitcoinstats.com` was 0.8.6. Although this could hint at filtering performed by the first group, it might as well just be coincidence (my guess is there weren't a lot of reachable nodes running 0.8.6 during the data-collection interval and despite the first group knowing about these nodes, they just never got selected).

-------------------------

