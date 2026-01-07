# Outgoing Reputation: Simulation Results and Updates

carla | 2025-10-17 18:20:51 UTC | #1

This post will report on simulation results for our updated reputation algorithm, based on the discussion in [this post](https://delvingbitcoin.org/t/hybrid-jamming-mitigation-results-and-updates/1147). It assumes understanding of the scheme, so please see the [BOLT PR](https://github.com/lightning/bolts/pull/1280) for details.

#### Simulation Results

We have re-run both the [resource](https://delvingbitcoin.org/t/hybrid-jamming-mitigation-results-and-updates/1147#p-3212-resource-attacks-3) and [sink](https://delvingbitcoin.org/t/hybrid-jamming-mitigation-results-and-updates/1147#p-3212-manipulation-sink-attack-9) attacks outlined in the original post [1], finding that:

* Protection against resource attacks remains intact; the amount paid to build reputation to jam channels compensates the targeted node.
* With outgoing reputation, attacking nodes running a sink attack are quickly cut off when they start to misbehave.
* When a honeypot is created to passively build reputation with a target node, the amount it earns from new traffic is greater than the damage that can be done by abusing the “freely earned” reputation.

<details> <summary>Exact numbers for the attacks we ran are available here.</summary>

The table below shows for each attack the change in the targeted node’s revenue (compared to its revenue in times of peace), and the cost to the attacker.

|                                   Attack | Target Peacetime Revenue | Target Attacktime Revenue | Channel Opens for Attack |   Attacker Reputation Prepay | General Jammed Channels\* |
| ---------------------------------------: | -----------------------: | ------------------------: | -----------------------: | ---------------------------: | ------------------------: |
|      Resource Attack / Slow Slot Jamming |                  835,448 |               247,946,078 |                        3 |                  247,269,056 |                         1 |
| Resource Attack / Slow Liquidity Jamming |                  835,448 |             3,240,062,877 |                        3 |                3,240,385,855 |                         1 |
|                              Sink Attack |            8,405,842,914 |            15,453,671,671 |                        6 | No prepay, passive forwarder |                        16 |


*Our simulation has a helper to general jam channels. In reality, this would require opening in expectation 50 channels or finding 40 unique paths through the network per channel general jammed.

</details>

It’s worth pointing out that if an attacker builds reputation then [targets a chain of nodes](https://delvingbitcoin.org/t/hybrid-jamming-mitigation-results-and-updates/1147/8#p-4312-path-length-attack-multiplier-1), only the last is compensated [2]. While compensation isn’t distributed along the chain, there is a significant cost to an attacker to build reputation to attack multiple nodes.

#### General Bucket Usage

The general bucket limitations introduced prevent trivial, costless attacks that would crowd out honest usage. Inspired by Bitcoin Core's addrman, we aim to impose restrictions on attackers without hindering honest payments.

The main concern is whether slot/liquidity allocation suffices for honest traffic lacking reputation. Previous data collected shows slot-count issues are unlikely (few HTLCs typically in flight). We validated liquidity resources by simulating on a mainnet graph snapshot, comparing payment success rates with/without mitigation for mean payment amount of $10 and $100 and found no significant change in payment failure rates.

#### Next

We think that this mitigation has reached the stage where it is good enough™️. If you disagree, our simulator is [here](https://github.com/carlaKC/jam-ln) to try out some attacks!

Our next step is to sanity check the following against real world data:

* Are node peers able to gain reputation?
* Will restrictions on general resources impact payment reliability?
* What are sane defaults for resource bucket sizes?

<details> <summary>Footnotes.</summary>

[1] To adjust to the changes in our mitigation, reputation in attacks is not built in the outgoing direction. General resources are also less trivially filled, so we count the requirement to jam them towards the cost of the attack.

[2] This is an unavoidable characteristic of local-only solutions, as they can’t take a look at the bigger picture.

</details>

:heart: in collaboration with @ClaraShk and @elnosh

-------------------------

ClaraShk | 2025-12-01 14:50:24 UTC | #2

A [concern](https://github.com/lightning/bolts/pull/1280#discussion_r2297895412) was raised that an attacker might game the reputation system by replying within the 90‑second window while forcing the next hop, due to natural network delays, to respond after 90 seconds. In such a scenario, the honest node's reputation would drop while the attacker's remains intact.  
The key insight is that to meaningfully damage a node's reputation, the attacker has to outcompete the natural flow of honest payments through the channel, which is difficult, and, under upfront fees, also costly. In a generous back-of-the-envelope estimate (granting the attacker the unrealistic ability to push 500 HTLCs through a single channel and ignoring several parts of our mitigation), we assumed the node processes around 85 honest payments a day. Even under these favorable assumptions, the attacker can only reduce the node's reputation by less than 2%, while paying over 3× that amount in fees.  
Our conclusion is that this attack is not practically feasible. It resembles the quick jamming strategy, which upfront fees are explicitly designed to make prohibitively expensive. You can explore the numbers yourself [here](https://docs.google.com/spreadsheets/d/1V0h7rrnEgrLbFvwq1DeLP0--pB9_DkWpKkg-3lfx_9Q/edit?usp=sharing).

-------------------------

