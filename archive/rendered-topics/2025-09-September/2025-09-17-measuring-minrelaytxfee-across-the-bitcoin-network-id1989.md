# Measuring minrelaytxfee across the Bitcoin network

danielabrozzoni | 2025-09-17 13:22:50 UTC | #1

I crawled the Bitcoin network to record the minimum relay fee advertised by reachable nodes through the `FEEFILTER` message.

The scan used a [modified version](https://github.com/danielabrozzoni/p2p-crawler/tree/feefilter_parsing) of [virtu’s p2p-crawler](https://github.com/virtu/p2p-crawler/) which connects to every node, waits up to three minutes for a `FEEFILTER` message, and records the result. If no `FEEFILTER` message is received within the timeout the node is listed as “No `FEEFILTER` sent.”

Only nodes that accept inbound connections are included, and the results are based only on what each node advertises in its `FEEFILTER` message. I don’t think nodes have a reason to fake this field, but I suppose it’s possible, and I did not perform additional checks such as sending test transactions or monitoring relays.

Two full scans were run on 10/09 and 15/09.

## Networks covered

| Network | Nodes 10/09 | Nodes 15/09 |
|----|----|----|
| i2p | 2,980 (10.6%) | 3,081 (10.8%) |
| ipv4 | 9,020 (32.0%) | 8,681 (30.4%) |
| ipv6 | 2,002 (7.1%) | 2,008 (7.0%) |
| onion_v3 | 14,216 (50.4%) | 14,769 (51.8%) |

## Reported minimum relay fees

| MinRelayFee | Nodes 10/09 | Nodes 15/09 |
|----|----|----|
| 1000 | 25,125 (89.0%) | 24,695 (86.5%) |
| No FEEFILTER sent | 2,135 (7.6%) | 2,331 (8.2%) |
| 100 | 569 (2.0%) | 1105 (3.9%) |
| 9170997 | 84 (0.3%) | 73 (0.3%) |
| 1 | 81 (0.3%) | 91 (0.3%) |

(Because categories with very few nodes were excluded, the percentages do not add up to exactly 100%.)

### Network-by-Network Breakdown

#### i2p

| MinRelayFee | Nodes 10/09 | Nodes 15/09 |
|----|----|----|
| 1000 | 2,862 (96.0%) | 2,844 (93.6%) |
| 100 | 61 (2.0%) | 143 (4.6%) |
| No FEEFILTER sent | 20 (0.7%) | 21 (0.7%) |

#### IPv4

| MinRelayFee | Nodes 10/09 | Nodes 15/09 |
|----|----|----|
| 1000 | 6,879 (76.3%) | 6,295 (72.5%) |
| No FEEFILTER sent | 1,758 (19.5%) | 1,924 (22.2%) |
| 100 | 223 (2.5%) | 307 (3.5%) |

#### IPv6

| MinRelayFee | Nodes 10/09 | Nodes 15/09 |
|----|----|----|
| 1000 | 1,685 (84.2%) | 1,639 (81.6%) |
| No FEEFILTER sent | 221 (11.0%) | 239 (11.9%) |
| 100 | 59 (2.9%) | 69 (3.4%) |

#### onion_v3

| MinRelayFee | Nodes 10/09 | Nodes 15/09 |
|----|----|----|
| 1000 | 13,699 (96.4%) | 13,877 (94.0%) |
| 100 | 226 (1.6%) | 586 (4.0%) |
| No FEEFILTER sent | 136 (1.0%) | 147 (1.0%) |

---

## Open questions

* I was surprised to see that a large share of IPv4 nodes didn’t send a `FEEFILTER`. Peter Todd suggested these might be spy nodes. When I checked the IP addresses, I noticed several clusters of about 50 nodes sharing the same subnet, though I did not investigate further.
* The **9170997** entries are a bit odd… I connected to a few of those nodes with Bitcoin Core and checked `minfeefilter` in `getpeerinfo` to make sure the crawler wasn’t wrongly parsing messages. The value really is what they’re advertising, but I have no idea why… if anyone has a theory, I’d love to hear it.

## Acknowledgments

The idea for this post was inspired by tweets from Murch ([original tweet](https://x.com/murchandamus/status/1962634242581766643), [correction](https://x.com/murchandamus/status/1962751958894485534))

Thanks to Peter Todd for a few pointers and a quick sanity check. All remaining errors are my fault :)

-------------------------

1440000bytes | 2025-09-17 13:47:18 UTC | #2

[quote="danielabrozzoni, post:1, topic:1989"]
I was surprised to see that a large share of IPv4 nodes didn’t send a `FEEFILTER`. Peter Todd suggested these might be spy nodes. When I checked the IP addresses, I noticed several clusters of about 50 nodes sharing the same subnet, though I did not investigate further.
[/quote]

Some of them might be running version older than [2016](https://github.com/bitcoin/bitcoin/pull/7542) so they cannot send feefilter or disabled fee filter using a version older than 2021 with [`-feefilter`](https://github.com/bitcoin/bitcoin/pull/21992) config option.

-------------------------

0xB10C | 2025-09-18 08:55:46 UTC | #3

Thanks for sharing!

[quote="danielabrozzoni, post:1, topic:1989"]
The **9170997** entries are a bit odd… I connected to a few of those nodes with Bitcoin Core and checked `minfeefilter` in `getpeerinfo` to make sure the crawler wasn’t wrongly parsing messages. The value really is what they’re advertising, but I have no idea why… if anyone has a theory, I’d love to hear it.
[/quote]

Bitcoin Core "rounds" the feerate filters a bit (i.e. it put them into bins) as sending an exact feefilter based on mempool contents is a way of fingerprinting nodes across networks. See [`FeeFilterRounder`](https://github.com/bitcoin/bitcoin/blob/1444ed855f438f1270104fca259ce61b99ed5cdb/src/policy/fees.h#L322-L343) in `src/policy/fees.h`.  The `FeeFilterRounder` defines a `MAX_FILTER_FEERATE` of `1e7` (10,000,000). `1e7` happens to "round" to `9170997`. This can also be seen in this test where `MAX_MONEY` (21,000,000 * COIN) is "rounded" to `9170997`.

https://github.com/bitcoin/bitcoin/blob/1444ed855f438f1270104fca259ce61b99ed5cdb/src/test/policy_fee_tests.cpp#L31-L32


When in IBD, a feefilter of `MAX_MONEY` is set:
https://github.com/bitcoin/bitcoin/blob/1444ed855f438f1270104fca259ce61b99ed5cdb/src/net_processing.cpp#L5382-L5386

so they should return a feefilter of `9170997`. Maybe the nodes were in IBD when you connected to them?

---

Looking at some logs I can see the following pattern happening repeatedly: 

We connect to peer=5129 at height 908087, the peer is at height 907929 (158 blocks behind). Initially, the peer sends a feefilter of 9170997. After four seconds, the peer lowers its feefilter of 9170997 to 1000. 

```
2025-09-01T09:29:34.83Z [net] send version message: version 70016, blocks=908087, txrelay=1, peer=5129
2025-09-01T09:29:34.83Z [net] receive version message: version 70016, blocks=907929, txrelay=1, peer=5129
..
2025-09-01T09:29:35.99Z [net] received: feefilter of 0.09170997 BTC/kvB from peer=5129
2025-09-01T09:29:39.08Z [net] received: feefilter of 0.00001000 BTC/kvB from peer=5129
```

The code for is:

https://github.com/bitcoin/bitcoin/blob/1444ed855f438f1270104fca259ce61b99ed5cdb/src/net_processing.cpp#L5388-L5391

-------------------------

danielabrozzoni | 2025-09-18 15:17:12 UTC | #4

[quote="1440000bytes, post:2, topic:1989"]
Some of them might be running version older than [2016](https://github.com/bitcoin/bitcoin/pull/7542) so they cannot send feefilter or disabled fee filter using a version older than 2021 with [`-feefilter`](https://github.com/bitcoin/bitcoin/pull/21992) config option.

[/quote]

That’s what I initially thought too :slight_smile: However, the majority of the nodes that didn’t send a reply are running Core 27.x:

```txt
Feefilter None - 2135 nodes (7.6%)
User agents:
1040 nodes /Satoshi:27.0.0/
255 nodes /Satoshi:27.1.0/
221 nodes /Satoshi:29.0.0/
96 nodes /Satoshi:28.1.0/
51 nodes /Satoshi:28.0.0/
...
```

[quote="0xB10C, post:3, topic:1989"]
so they should return a feefilter of `9170997`. Maybe the nodes were in IBD when you connected to them?

[/quote]

Thanks a lot for the detailed explanation! Somehow it didn’t occur to me to simply `git grep 9170997` in the codebase :) 

Yes, those nodes were in IBD when I initially connected to them, I just checked the logs.

This also explains why some nodes replied with `9170997`, but didn't have `9170997` as `minfeefilter` when I later connected to them with Core to double-check. Thanks again!

-------------------------

