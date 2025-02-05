# Stats on orphanage overflows

0xB10C | 2025-02-05 14:48:07 UTC | #1

I've recently been asked about some numbers on orphanage overflows of my nodes. In Bitcoin Core, [`LimitOrphans()`](https://github.com/bitcoin/bitcoin/blob/33932d30e382d1296be438ec5365fa0a56cf0864/src/txorphanage.cpp#L123) is [called](https://github.com/bitcoin/bitcoin/blob/33932d30e382d1296be438ec5365fa0a56cf0864/src/node/txdownloadman_impl.cpp#L431) every time we add an orphan to the orphanage. Next to removing expired orphans from time to time, `LimitOrphans()` will remove (usually) one random orphan from the orphanage if it's larger than the maximum orphanage size (default 100; controlled with `-maxorphantx=n`). This is logged with `orphanage overflow, removed 1 tx`.

I analyzed the debug.logs of some of my nodes, which all have the default maximum orphanage size of 100.

![image|100%](upload://xjkMVmwC03Cg7pwsL5VelspjPMX.jpeg)

-------------------------

0xB10C | 2025-02-05 15:46:40 UTC | #2

I found >10M removals on, for example, 2024-09-14 across all nodes to be surprising. Looking at data from node alice on 2024-09-14 shows that we're sometimes removing more than 100k orphans per minute. This feels like someone flooding us with orphans.

![image|690x339](upload://n3pEWKpXabKdDeq704rVAh3dtPX.png)

Something like this is probably pretty effective at clearing our orphanage from orphans received from other peers.


Edit: >99% of these have a weight of 501 WU or 502 WU and are similar to this [transaction](https://mempool.space/tx/ac8990b04469bad8630eaf2aa51561086d81a241deff6c95d96d27e41fa19f90) which seems related to runestone mints.

Edit2: I briefly checked and it seems like these transaction were all solicited. Additionally, grepping for `Requesting tx` yields more than 10M lines.

-------------------------

