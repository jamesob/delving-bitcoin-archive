# Mitigating Channel Depletion in the Lightning Network: A Survey of Potential Solutions

renepickhardt | 2025-04-28 19:08:03 UTC | #1

# Good morning fellow Lightning Network developers (:

Recent analyses suggest that [most channels on the Lightning Network are **expected to be depleted** over time, primarily due to selfish routing behavior within the current protocol design](https://bitcoinops.org/en/podcast/2024/12/12/).

The problem: Depleted channels drastically lower the likelihood of successful payments. Without probing, any given link has roughly a 50% chance of fulfilling a routing request.

**Question:** Can we reasonably do something about this?

This post collects potential solutions — please view it as a reference and starting point. I'm happy for feedback, refinements, and critiques.  
Some ideas included might be philosophically undesirable but deserve technical consideration.

---

# 1. Two Broad Strategies

- **Protocol changes** (requiring updates)
- **Voluntary node behavior changes**

---

# 2. Protocol Changes

Changes would modify the LN protocol, sometimes needing broad adoption.

---

## Sharing liquidity information

Extend the gossip protocol to [share liquidity info](https://github.com/lightning/bolts/pull/780), either locally (e.g. the [friend of a friend network](https://en.wikipedia.org/wiki/Friend-to-friend)) or network-wide.  
Even partial signals (e.g., liquidity at channel ends) could be helpful.

- **Pros:**  
  (Min-cost flow) routing becomes much easier as the uncertainty decreases.
- **Cons:**  
  Honest signaling enforcement, bandwidth, and privacy.

---

## (Centralized?) routing coordinators

[Selfish node behavior exacerbates depletion](https://delvingbitcoin.org/t/channel-depletion-ln-topology-cycles-and-rational-behavior-of-nodes/1259). A coordinator could reorder payments for better liquidity use and engage in flow and congestion control. One could see a [trampoline node as a routing coordinator](https://medium.com/@ACINQ/phoenix-wallet-part-4-trampoline-payments-fb1befd027c8) for its clients

- **Pros:**  
  Less gossip overhead, global optimization possible.
- **Cons:**  
  Trust issues, decentralization concerns, liquidity limits may still remain to some degree.

---

## Distance-vector or opportunistic routing

Move from source-routing to local, best-effort forwarding decisions.

- **Pros:**  
  Forwarders can act on real liquidity, less reliance on probing. 
- **Cons:**  
  Privacy concerns (sender/receiver visible); new forms of selfishness.

---

## Symmetric channel fees

Enforce fee symmetry between directions to encourage balanced flows.

- **Pros:**  
  Supports circular economies, mitigates depletion.
- **Cons:**  
  Needs broad adoption — difficult to bootstrap.

---

## `update_update_htlc`: Partial forwarding

Rather than failing large HTLCs, allow partial forwarding (push-relabel style, [Rohrer et al. 2017](https://arxiv.org/abs/1708.02419)).

- **Pros:**  
  Enables "link AMP", more natural flow network behavior.
- **Cons:**  
  Could worsen depletion without better depleted-channel signaling.

---

## Multiparty channels

Using shared UTXOs among multiple participants increases capital efficiency and boosts reliability but [they come (even with softforks) with their own challenges](https://petertodd.org/2024/covenant-dependent-layer-2-review)

**Challenges**:
- Secure unilateral exits
- Interactive complexity
- Multihop routing is unclear
- Potentially rethinking routing itself as that may not be necessary

Further reading: [Burchert, Decker, Wattenhofer (2017)](https://royalsocietypublishing.org/doi/10.1098/rsos.180089).

---

# 3. Behavior Adoption by Routing Nodes

There are various ways nodes can voluntarily address liquidity issues.

---

## Dynamic pricing

Already common:  
- [CLBOSS](https://corelightning.org/automate-lightning-node-management-with-clboss/), [lndmanage](https://github.com/bitromortac/lndmanage)
- [Fee rate cards](https://diyhpl.us/~bryan/irc/bitcoin/bitcoin-dev/linuxfoundation-pipermail/lightning-dev/2022-September/003685.txt)
- [Negative fees](https://github.com/lightning/blips/pull/22)
- [Inbound fees](https://github.com/lightning/bolts/issues/835)

**Limits:**  
Reactions are delayed due to gossip forwarding, based on local information, it is questionable weather the network can find an equilibrium. 

---

## Flow and congestion control

Beyond pricing:

- [Use `htlc_maximum_msat` as a control valve](https://blog.bitmex.com/the-power-of-htlc_maximum_msat-as-a-control-valve-for-better-flow-control-improved-reliability-and-lower-expected-payment-failure-rates-on-the-lightning-network/)
- Available Liquidity may be signaled  via `htlc_maximum_msat`, which is at least a semantic that nodes understand. 
- ([Sivaraman et al. 2020](https://www.usenix.org/conference/nsdi20/presentation/sivaraman)) proposed a routing scheme based on nodes buffering routing requests for congestion control.

Semantics of Control valve using `htlc_maximum_msat` are not agreed upon and is thus not supported by node software.

---

## Reverse logistics

Relocating liquidity via:

- **Off-chain**: [BOS](https://github.com/alexbosworth/balanceofsatoshis), decentralized rebalancing ([example paper](https://arxiv.org/abs/1912.09555)), [min-cost circulation ideas](https://x.com/renepickhardt/status/1909918246020448693).
- **On-chain**: [PeerSwap](https://www.peerswap.dev/), [Loop](https://lightning.engineering/loop/), [Splicing](https://lightningsplice.com/splicing_explained.html).
- **Liquidity markets**: [Liquidity Ads](https://bitcoinops.org/en/topics/liquidity-advertisements/), [proprietary solutions](https://amboss.space/magma),...

Note: (Fee Free) offchain rebalancing schemes might also be enabled or supported on a protocol level. In particular offchain rebalancing can be modeled as a [circulation problem](https://en.wikipedia.org/wiki/Circulation_problem) instead of just finding a single rebalancing cycle.

---

## Hierarchical topologies

Organize Lightning like real transport systems (compare with Highway, Trainsystems,...):

- Spanning tree based "core" node networks (spanning trees don´t deplete but split if a link breaks)
- Peripheral nodes connect strategically
- Use sparse graphs or skip topologies

---

# 4. Conclusion

This is an overview, not a final prescription.

Many solutions interact and involve trade-offs between implementation overhead, privacy, scalability, and decentralization.

**Feedback, missing ideas, and criticisms welcome!** (:

## Disclaimer:
1. While those strategies may help to fight depletion of channels and improve reliability as they support sending nodes to quickly find the liquidity most of them do not address the fact that [payments can be infeasible in payment channel networks without making additional on chain transactions](https://delvingbitcoin.org/t/estimating-likelihood-for-lightning-payments-to-be-in-feasible/973).  
2. I have a long form of this text on my computer with a bit more context but used an LLM to make it more succinct for your convenience.

-------------------------

