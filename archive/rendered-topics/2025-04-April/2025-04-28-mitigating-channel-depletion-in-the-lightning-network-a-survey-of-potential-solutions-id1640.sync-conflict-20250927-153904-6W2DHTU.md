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

brh28 | 2025-04-28 20:31:21 UTC | #2

Rather than use gossip to share liquidity info, another potential solution which I briefly describe in [this post](https://delvingbitcoin.org/t/revisiting-pathfinding/1503) is to add a query message to get path(s) from a peer.

- This improves the likelihood of successful payment because each hop can inform the requester of a feasible outbound channel set without revealing any of their channel balances. 
- Depending on implementation, this potentially allows nodes to change their routing policies (e.g fees) on a per transaction basic, which improves their control over liquidity flow.

Of course, this approach has a lot of implications, so I want to create follow-up on my previous post with details regarding an implementation and a cost/benefit analysis, but I think it's actually a relatively simple way to solve a lot of existing problems.

-------------------------

t-bast | 2025-04-29 08:37:42 UTC | #3

[quote="renepickhardt, post:1, topic:1640"]
Semantics of Control valve using `htlc_maximum_msat` are not agreed upon and is thus not supported by node software.
[/quote]

Can you detail that a bit more? We've been using `htlc_maximum_msat` in eclair as a control valve like you suggested a few years ago, and it seems to be working quite nicely (I'm not sure how exactly we could quantify that "it works", but at least we've implemented the mechanism and haven't seen any specific issues with it).

I don't know if other implementations have done it, but it's nice because it doesn't require any protocol changes: you only need to update your implementation, and other nodes on the network will respect the `htlc_maximum_msat` value you dynamically set. It creates a bit of a (reasonable) `channel_update` spam though.

-------------------------

renepickhardt | 2025-04-29 11:44:03 UTC | #4

[quote="t-bast, post:3, topic:1640"]
Can you detail that a bit more?
[/quote]

Sorry I may have shortened my initial post a bit too much with respect to the control valves section. As you pointed out the semantics of `htlc_maximum_msat` are clear in the sense that your node will not accept an hltc of a larger amount and indeed sending nodes will respect this in path or flow finding (though it is not clear what happens if a node tries to add a second HTLC with the same preimage that also respects `htlc_maximum_msat`).

However I have seen various motivations of why node operators select a specific value for `htlc_maximum_msat` for example
* I have seen node operators signaling the amount of liquidity that they have left in the channel (or a fraction of the remaining liquidity, such that they do not have to update the value after each successful routing).
* In my initial Blog article I suggested to use a markov process such that two peers that share a channel converge to a ratio of `htlc_maximum_msat` values so that the expected flow through the channel is balanced - given the demand. That of course is a very different semantic than just selecting a fraction of the liquidity
* In eclair it seems that you set the value to 0 if the channel is almost depleted and that [you have various threasholds depending on the available liquidity](https://github.com/ACINQ/eclair/blob/826284cb277c28c7eef14aa275f3d6e3255c8e66/docs/release-notes/eclair-v0.10.0.md?plain=1#L77). Also you seem to [set the value](https://github.com/ACINQ/eclair/commit/8320964c547022c02c4d6fd0c7af82284534cc22)
 to what you [believe you can forward](https://github.com/ACINQ/eclair/blob/826284cb277c28c7eef14aa275f3d6e3255c8e66/eclair-core/src/main/scala/fr/acinq/eclair/router/BalanceEstimate.scala#L123). 

[quote="t-bast, post:3, topic:1640"]
We’ve been using `htlc_maximum_msat` in eclair as a control valve like you suggested a few years ago, and it seems to be working quite nicely (I’m not sure how exactly we could quantify that “it works”, but at least we’ve implemented the mechanism and haven’t seen any specific issues with it).
[/quote]

I am happy to hear that you an confirmed that a version of my suggestion works for you. I'd love to see some quantification for this. Maybe if it has sufficient interest from your end we could out of band discuss how we could measure and quantify this. Maybe it gives insights on weather this is sufficient or if other steps are necessary.

[quote="t-bast, post:3, topic:1640"]
I don’t know if other implementations have done it, but it’s nice because it doesn’t require any protocol changes: you only need to update your implementation, and other nodes on the network will respect the `htlc_maximum_msat` value you dynamically set. It creates a bit of a (reasonable) `channel_update` spam though.
[/quote]

Yes that was exactly why I advocated for `htlc_maximum_msat` initially. It works out of the box without protocol upgrade which is extremely nice! However as in my first reply a protocol wide agreement on what nodes signal with their value may make this mechanism even stronger.

-------------------------

nick | 2025-04-30 14:15:45 UTC | #5

Hi Rene,

This makes me think of @ZmnSCPxj 's [Forwardable Peerswaps proposal](https://diyhpl.us/~bryan/irc/bitcoin/bitcoin-dev/linuxfoundation-pipermail/lightning-dev/2022-October/003710.txt). Essentially a way to voluntarily optimize on-chain swaps, "forwarding" them from net senders of liquidity to net receivers and replenishing liquidity across the whole route. 

This would take what would be a local peerswap rebalancing one channel and have it rebalance many channels at once, with each participating node benefiting. Should at least be considered in this discussion imo.

-------------------------

ZmnSCPxj | 2025-04-30 16:11:09 UTC | #6

Nearly all proposals listed above require some erosion of privacy.

In addition, flow valves do not fix the core complaint of payment failure anyway --- they just signal the payment failure earlier.

We have to accept that any solution to payment failure requires onchain action.  Full stop.  The point is not to set onchain actions to 0.  The point is to keep the onchain actions low, which is sufficient for scaling purposes.

Forwardable peerswaps scales a single onchain action to multiple channels being rebalanced at once.  Even if only a single channel is rebalanced, it potentially still scales multiple smaller in-Lightning payments over that channel to a single onchain action. It also does not erode existing privacy, as the details of in-Lightning payments are not changed (unlike local forwarding or centralized routing).

Another thing to consider is sidepools.  Sidepools are multiparty channels, but are unpublished. This implies that existing routing is not changed at all (avoiding "rethinking routing itself") and existing privacy is preserved (multiparty channels leak routing to all participants in the channel, but in sidepools, the multiparty construct is restricted to only rebalancing 2-party channels).  Again, this implies onchain actions to move funds to/from sidepools, but again, the point of scaling is to have multiple in-Lightning payments be summarized into a few onchain actions, not eliminate onchain actions to 0.

-------------------------

