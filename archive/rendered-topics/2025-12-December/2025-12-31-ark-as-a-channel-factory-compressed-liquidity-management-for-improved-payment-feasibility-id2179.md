# Ark as a Channel Factory: Compressed Liquidity Management for Improved Payment Feasibility

renepickhardt | 2025-12-31 11:42:49 UTC | #1

Over the past months, I have discussed the ideas in this post in many bilateral conversations. To avoid repeating arguments inconsistently and to invite broader feedback, I am writing them down here. I apologize to readers already familiar with these concepts and welcome comments, objections, and open questions. 

Note: **As I am not a native english speaker I used an LLM for copy editing; the technical content and arguments are my own. While I supervised the LLM I did not diverge from the typical formatting and linguistic style**


## Abstract

The Lightning Network enables fast, non-custodial Bitcoin payments when sufficient liquidity exists along a payment flow (often just a single path). However, **payment feasibility** (the existence of such a flow) is a structural constraint that cannot be resolved by routing heuristics or off-chain rebalancing alone. When liquidity is insufficient, **on-chain transactions are required to modify the channel graph**, which fundamentally limits the Lightning Network’s ability to scale bitcoin payments.

Multi-party channels are known to improve capital efficiency but are difficult to coordinate in practice. Ark proposes a mechanism for coordinating multi-party state updates in rounds with a relatively low coordination overhead for the members to agree on a new state. This post challenges the emerging narrative that Ark should primarily serve as a last-mile payment system interconnected via Lightning channels. Instead, it argues that Ark might be better understood as **infrastructure for Lightning**, specifically as a **channel factory** enabling efficient reconfiguration of liquidity. The focus is on trade-offs, feasibility constraints, and open research questions.

**Important**: This document explicitly does not advocate against Ark as Payments it just asks whether there may be a better usecase. 

## 1. Review of Payment Feasibility and Structural Limits in Lightning

A Lightning payment is feasible if the **minimum cut** between sender and receiver in the liquidity graph exceeds the payment amount. In practice, feasibility is difficult to determine due to [uncertainty about remote channel balances](https://arxiv.org/abs/2103.08576
). This uncertainty can be reduced using [probabilistic routing and optimally reliable payment flows](https://arxiv.org/abs/2107.05322).

However, while improved routing, fee updates, and rebalancing can increase utilization, [they do not change global feasibility unless the channel graph itself is modified](https://github.com/renepickhardt/Lightning-Network-Limitations/blob/f670738cd2af93a55c3c919c9a864015f6dd042a/Limits%20of%20two%20party%20channels/paper/a%20mathematical%20theory%20of%20payment%20channel%20networks.pdf
).

In particular:

* Off-chain rebalancing redistributes liquidity within existing channels.
* It does *not* create new connectivity or directional capacity.
* When no feasible flow exists, an **on-chain transaction** (open, close, splice, etc.) is required.

This distinction is central. Even if a node can move liquidity from one channel to another via circulations, such operations also affect remote channels and therefore do not selectively increase feasibility for a desired payment. Changing feasibility locally without altering the rest of the network requires on-chain intervention.

Formal analysis in [A Mathematical Theory of Payment Channel Networks](https://github.com/renepickhardt/Lightning-Network-Limitations/blob/f670738cd2af93a55c3c919c9a864015f6dd042a/Limits%20of%20two%20party%20channels/paper/a%20mathematical%20theory%20of%20payment%20channel%20networks.pdf
) shows that the **maximum supported payment rate** depends critically on:

* fixed available on-chain bandwidth, and
* the expected rate of infeasible payments.

The constraints on two-party channel networks are sufficiently severe that, under realistic assumptions, the expected rate of infeasible payments becomes too high for such an off-chain network to scale Bitcoin payments without substantial on-chain support. This limitation motivates exploring coordination mechanisms that can restructure the graph topology more efficiently utilizing less on chain footprint.


## 2. Ark: Rounds and Virtual UTXOs

Ark introduces **rounds** in which participants exchange **virtual UTXOs (vTXOs)** coordinated by an Ark Service Provider (ASP). These vTXOs represent off-chain value commitments that can be reassigned over time. 

It seems that Ark style systems reduce the coordination and interactivity requirements of the coin pool by introducing the ark service provider as a coordinator who overprovisions liquidity for a fixed time window so that not everyone has to agree on every new state immediately but only eventually. This is a very nice property.

However, when Ark is used *directly* as a payment system, three properties become relevant:

1. **Liquidity lock-up due to expiry**
vTXOs release liquidity only after their timeout, binding the ASP’s capital for the duration of the expiry window.
2. **Change amplification**
Payments typically destroy one input and create multiple outputs (recipient and change), increasing the amount of liquidity the ASP must front.
3. **Trust during inter-round settlement**
Payments settle conclusively only at round boundaries. Between rounds, spent vTXOs could theoretically be double-spent, raising questions about custody and regulatory interpretation of the ASP’s role.

Under frequent payment patterns within the expiry window, the ASP’s required working capital can [grow substantially relative to net payment volume](https://bitcoin.stackexchange.com/questions/128113/how-well-does-ark-scale-bitcoin-payments). This complicates claims that Ark alone provides a low-overhead, scalable payment layer.


## 3. Interpreting Ark as a Channel Factory and LN Infrastructure Layer

An alternative interpretation is to treat Ark not as a payment system competing with Lightning, but as **infrastructure beneath it**. More specifically it can be understood as a **channel factory or multi-party channel mechanism**.

In this framing:

* vTXOs correspond to Lightning channels,
* an Ark round can open, close, or reshape many channels atomically,
* a **single on-chain transaction** can **reconfigure a large fraction** of the channel graph.

This differs fundamentally from routing or rebalancing. Rather than optimizing flows *on* a fixed graph, Ark enables structural changes to the topology of the graph itself, which potentially makes previously infeasible payments feasible.

Multiple channel operations - funding, closing, or splicing - may be compressed into a single Ark round transaction. Since Lightning already requires on-chain confirmations for such changes, Ark does not worsen latency from this perspective as ark rounds could eventually take place in every block.

## 4. Liquidity Reconfiguration and Operational Considerations

Lightning node operators already need to:

* monitor channels,
* respond to on-chain events,
* occasionally rebalance or close channels.

Rolling vTXOs in Ark rounds (e.g., a few times per month) is operationally comparable. Channels with higher utilization may require more frequent rollovers, which can be coordinated while the channel is actively used.

Failure assumptions change in detail but not in kind: operators require periodic engagement rather than continuous supervision.


## 5. Liquidity Pooling and Dynamic Allocation

In a channel factory or multi-party channel:

* liquidity is pooled across participants,
* allocation can be adjusted round by round,
* demand can be forecast or reacted to dynamically.

This contrasts with the current LSP model, where liquidity is provisioned per customer and often remains idle or asymmetrically distributed.

Pooling may improve capital efficiency - particularly but not only for mobile clients. in the case of Ark it does however introduce trade-offs involving expiry selection, coordination overhead, and ASP liquidity management. Determining optimal timeout parameters for the LSP to reclaim vTXOs remains an open problem.

## 6. Integration with Lightning and Custody Considerations

When Ark is used as infrastructure:

* payments occur over Lightning,
* settlement remains atomic and end-to-end,
* the ASP coordinates liquidity but does not intermediate payments.

This preserves Lightning’s non-custodial real-time settlement properties. In contrast, using Ark directly for payments introduces trust assumptions between rounds.

The separation of concerns - Lightning for payments, Ark for liquidity coordination - maintains Lightning’s core guarantees like instant settlement of payments with strong privacy while addressing structural scalability limits.


## 7. Routing, Gossip, and Open Questions

Channels funded via vTXOs lack on-chain funding transactions and therefore cannot currently be announced via Lightning gossip. This raises several open questions:

* How should such channels be represented to routers?
* Should routing operate at the factory level?
* Are new abstractions needed for liquidity advertisement?
* How do these mechanisms affect privacy and reliability?
* Should vTXO-backed channels exist only between ASP and users, or also directly between users?

## 8. Extended Open Questions

Viewing Ark as infrastructure for Lightning rather than as a standalone payment system clarifies some trade-offs, but also raises additional open questions that merit further study.

* **Incentives and ASP behavior:**
How should incentives be aligned such that an ASP’s liquidity management decisions improve Lightning-wide feasibility rather than only local profitability? How does competition between multiple ASPs affect liquidity allocation and pricing?

* **Centralization pressure:**
Does pooling liquidity in multi-party constructions introduce economies of scale that favor a small number of large factories? How does this compare to existing hub and LSP dynamics in Lightning?

* **Failure modes and exits:**
Following [Peter Todd's article on layer 2 review](https://petertodd.org/2024/covenant-dependent-layer-2-review): What are the on-chain and operational consequences of ASP failure or mass exits? How gracefully does the system degrade under stress, and what are the worst-case on-chain costs?

* **Latency versus feasibility:**
Ark enables structural reconfiguration, but only at round boundaries. How should round frequency and expiry windows be chosen to balance payment feasibility, capital efficiency, and user experience?

* **Privacy considerations:**
Does round-based coordination leak information about demand patterns or user activity over time? How do anonymity sets compare to those of bilateral Lightning channels? 

* **Interoperability and routing abstractions:**
How should vTXO-funded channels be represented to routers given the lack of on-chain funding transactions? Are new gossip or factory-level abstractions required?

These questions are not specific to Ark, but arise naturally whenever multi-party liquidity coordination is introduced. Addressing them is essential to understanding whether such mechanisms can sustainably complement the Lightning Network

## References

1. Pickhardt et al., *On the Uncertainty of Lightning Network Channel Balances*
https://arxiv.org/abs/2103.08576
2. Pickhardt & Richter, *Optimally Reliable Payment Flows on the Lightning Network*
https://arxiv.org/abs/2107.05322
3. Pickhardt  *A Mathematical Theory of Payment Channel Networks* (draft)
https://github.com/renepickhardt/Lightning-Network-Limitations/blob/f670738cd2af93a55c3c919c9a864015f6dd042a/Limits%20of%20two%20party%20channels/paper/a%20mathematical%20theory%20of%20payment%20channel%20networks.pdf
4. Pickhardt *How well does Ark scale Bitcoin payments?*
[https://bitcoin.stackexchange.com/questions/128113/how-well-does-ark-scale-bitcoin-payments](https://bitcoin.stackexchange.com/questions/128113/how-well-does-ark-scale-bitcoin-payments)
5 Todd, *Covenant dependent layer 2 review*  https://petertodd.org/2024/covenant-dependent-layer-2-review. 
6. BTC++ Talk on Lightning scaling and limitations
https://www.youtube.com/watch?v=c3AuaHJordg
7. Bitcoin Amsterdam LN vs Ark panel (2025)
https://www.youtube.com/watch?v=AU52kQz2zIM

---

## Acknowledgements

Discussions with peers and feedback from presentations at BTC++ and Bitcoin Amsterdam helped clarify the arguments and trade-offs presented here. This research was sponsored through opensats and patreons.

-------------------------

instagibbs | 2025-12-31 13:55:37 UTC | #2

I’ve been interested in this direction for a little under a year and have been doing exploratory work towards this recently.

My focus is under the assumption that routing nodes are already highly provisioned, but end-nodes such as mobile clients and related make LSP economics difficult at best due to inbound liquidity requirements, which could be highly optimized by layering it on Ark. Given my focus on LSP+mobile client style arrangements, this also lets me punt on the gossip/routing question for now.

This is essentially a Timeout Tree with the ability to “teleport” the end-channels between trees, at the cost of the additional wait at the leaf’s “forfeit” stage to ensure the ASP isn’t being double-spent (which imo is immaterial to user incentives).

With the hash-based Ark (hArk), I think this architecture becomes quite plausible with a lightly modified variant of today’s LN:

1. Normal channel operation in a tree
2. Client+LSP request refresh to new tree
3. Tree with refreshed vtxo is generated / confirmed (can even be done predictively before 2!)
4. Client+LSP quiesce old channel
5. Client+LSP do new tree vtxo channel setup (clone channel)
6. Client+LSP sign forfeit tx ← Old channel revoked, new channel can only be claimed if revoke spent
7. ASP sends hash preimage ← new channel under control of Client+LSP
8. Client-LSP unquiesce channel in new tree, continue operation

Note that with predictive tree generation and confirmation, this results in the client being able to refresh their channel nearly instantaneously, even if they only unlock their mobile phone once before old tree expiry.

Note that connector-based Ark would make the interactivity story significantly more daunting, or force significant channel downtime during round refreshes.

There are lots of details to sort out clearly, but I just wanted to sketch out what this could practically look like with no consensus or mempool changes, nor major BOLT rewrites.

> Since Lightning already requires on-chain confirmations for such changes, Ark does not worsen latency from this perspective as ark rounds could eventually take place in every block

One UX consideration here is that with today’s LN, 0-conf channels allow the client to see the fully signed transaction right away, while if you were waiting for batching rounds, this is another step removed. The LSP is promising that that ASP will emit a transaction with the appropriate structure to generate a new channel vtxo.

-------------------------

jgmcalpine | 2026-01-03 22:17:28 UTC | #3

Re: Economic crossover of Ark-funded vs. On-chain funded topologies

You make a compelling case for utilizing Ark to improve payment feasibility via compressed liquidity management.

I’m trying to model the economic crossover point for an LSP managing this topology.

In the Ark-funded model (the Factory), the LSP minimizes liquidity mismatch but incurs a perpetual "Liveness Tax"—the backing vTXOs must be refreshed/rolled over at every expiry (e.g., every 2-4 weeks). While efficient, this incurs recurring ASP fees and operational requirements.

In the On-chain funded model (Legacy channels), the LSP pays a high opportunity cost for locked capital (inefficient allocation) but incurs near-zero maintenance costs.

Does your model suggest that the entire network graph should remain Ark-funded to maximize payment feasibility? Or is there a threshold of predictable demand where a user achieves balanced flows (or high velocity) such that the operational overhead of the Factory outweighs the benefits of dynamic resizing, making a static channel the economically superior choice?

Effectively: Is the Factory a permanent home, or a staging ground for Liquidity Discovery?

-------------------------

vincenzopalazzo | 2026-01-04 13:07:54 UTC | #4

Thanks for this write-up! I was interested in ark when the protocol was proposed and started to be implemented to understand the interaction between lightning and how it is possible to use it to solve lightning problems.

I like to define ark as a technique, and not as a layer, just because it fits very well in many use cases, and we may not need to classify ark in a category, but just use the underlying technique to fix some problems that we are having and in this case with lightning scalability.

---

So I will add my considerations that I had when I presented a PoC of an ark channel factory in Prague last year with an LDK full node implementation (See the draft PR in https://github.com/vincenzopalazzo/lampo.rs/pull/452)

I will try to add the design decisions that I was thinking about while implementing it regarding some of your points.

\>Interoperability and routing abstractions

My idea on how to use the ark channel factory on today’s model is more like implementing a phoenix model LSP, where the mobile is having a channel with one of the LSPs. Here there is an assumption that the LSP for the lightning node is also an Ark operator, but in the big picture, I think this assumption can be removed.

At the implementation level, if today you want to implement this, it is already possible because mobile + LSP are already 0 conf channels, so you can just assume this is a confirmed channel for the ark channel.

Then, due to the ark has a liveness “issue” that requires the mobile wallet to be online at some point to refresh the vtxo, you can use this time to “refresh” the channel by basically closing and opening a new channel or splicing (depending on how complicated you want to make the mobile wallet + LSP relationship).

## Consideration at the lightning script level.

At the script level as of today, we can already implement it somehow by just having some extra path for every script (I need to double check this with the unilateral exit) and here is an unreviewed funding transaction https://github.com/vincenzopalazzo/lampo.rs/blob/63027fea90e7292d32a4e5c9465848b06339aa38/lampo-ark-wallet/src/lib.rs#L173 example

The general idea is that for every lightning script you will have two cases:

* `<lightning script> + ASP signature`
* `<lightning script> + timeout for the ark unilateral exit`

---

I also considered the case where a public channel that impacts the public network will be based on an Ark channel factory, and with my current understanding of the Ark protocol, I think it will create overcomplicated stuff at the gossip level & interconnection between lightning nodes that have channels with different ASPs, but I can be wrong. It has been a while since I thought about this problem, so I probably need too refresh my mind on this last consideration.

-------------------------

renepickhardt | 2026-01-05 09:25:37 UTC | #5

@instagibbs Thanks for the detailed sketch. I agree I may be doing premature optimization by trying to include “all the routing” immediately, and that the LSP+mobile case is likely the most pressing place to start.

I found your refresh/migration approach really thought-provoking. In particular, I liked the idea of treating the transition as a controlled “revocation” of the old vTXO state, rather than requiring the ASP to be robust against two fully live states for an extended overlap period.

My current mental model is: the forfeit + preimage gating makes the “new” allocation safe only once the “old” allocation is rendered economically/cryptographically non-viable, and the extra leaf-level waiting is the price for that safety i.e., consistent with your point that the forfeit stage is what prevents the ASP from being double-spent across trees/rounds. If that’s correct, then the benefit is not “less liquidity in general”, but specifically reduced *worst-case overlap exposure* (not provisioning as if both old and new claims could be exercised). Is that the intended interpretation, or am I misunderstanding your approach?

@jgmcalpine I like the “staging ground vs permanent home” framing, but I’m not sure I’d subscribe to it as the dominant mental model.

I’d rather separate two things that seem to get conflated: (1) a protocol liveness requirement (you can’t ignore expiry forever), and (2) an economic “liveness tax”. I agree there is a *baseline* recurring refresh at expiry, but I think it can often be amortized heavily via batching/round design. The stronger cost pressure seems to come from the service level being demanded: how often rollovers/top-ups are needed **early** relative to the agreed expiry, and what responsiveness guarantees users want. In other words, the ASP is effectively selling a liquidity/responsiveness option, and the reserve/overprovisioning behind that can be priced (routing fees, explicit “fast refresh/top-up” fees, or utilization-based charges). I do agree the pricing model needs to be derived carefully (including for the case I’m describing). But as I just learned using the techniques proposed by @instagibbs we may not even need to price the overprovisioning if users are willing to accept the additional wait.

On the “crossover” question: if a relationship is genuinely stable and rarely needs resizing, I can imagine there being a point where a long-lived on-chain channel is cheaper. My hunch, though, is that this set may be smaller than intuition suggests, because “balanced flows” are a poor baseline in practice. With linear fees and selfish sender behavior, there’s systematic depletion pressure along cheaper gradients, so instead of expecting stable balance, it may be more realistic to make reconfiguration cheap and continuous. (This discussion also resonates with the liquidity dynamics described here: [https://bitcoinops.org/en/podcast/2024/12/12/](https://bitcoinops.org/en/podcast/2024/12/12/)). (I will soon provide an updated version of the mathematical theory of payment channel networks where I even have explicite proofs for that phenomenon)


@vincenzopalazzo Thanks for sharing the PoC - having a concrete implementation is incredibly helpful for grounding this discussion.

I’m curious about your view on standardization and usefulness: do you think this could plausibly converge into something interoperable at the spec level (e.g., a BOLTs extension), even if initially scoped to LSP-like deployments and a particular Ark implementation? Or do you think it’s more likely to remain an implementation technique that varies too much across operators to standardize?

Where do you see the clearest advantage over existing channel-management tools (splicing, dual-funding, or current LSP patterns)? Conversely, what do you see as the biggest blocker to making it genuinely useful in production - UX/liveness constraints, script complexity, multi-operator interoperability/gossip, or something else?

-------------------------

instagibbs | 2026-01-05 12:53:27 UTC | #6

[quote="renepickhardt, post:5, topic:2179"]
If that’s correct, then the benefit is not “less liquidity in general”, but specifically reduced *worst-case overlap exposure* (not provisioning as if both old and new claims could be exercised). Is that the intended interpretation, or am I misunderstanding your approach?

[/quote]

There are two components to the liquiditiy requirements:

1. ASP: worst-case overlap exposure, where refreshed vtxos are in a forefeited yet not viable to sweep in a vbytes-cheap state(tree timeout). With LN on top, the velocity of these Ark refreshes can go down.
2. LSP: Inbound liquidity: Either during construction or over time, LSPs choose to have money parked into channels, and in many contexts, are never used. If the mobile user is online they could splice it out (for vbytes cost), but the user may almost never come back online. Layering LN on Ark allows this “targeting” to more tightly track practical usage. If too much is allocated, it can be reclaimed as the tree times out. If  too little, a new channel can simply be instantiated in a new tree (concurrent channels, or consolidating via revocation of old vtxo, at additional liquidity cost of refresh!)

The challenge here is that while marginal vbytes have dropped to \~0, there is the ASP/LSP liquidity optimization challenge left and that is what (someone) should explore.

I also have to note that the ASP/LSP have to be the same identity or trust each other otherwise the LSP is on the hook for unrolling channels when the mobile client turns off their phone, otherwise they hand all their money to the ASP at tree timeout.

-------------------------

ErikDeSmedt | 2026-01-06 09:20:42 UTC | #7

> I also have to note that the ASP/LSP have to be the same identity or trust each other otherwise the LSP is on the hook for unrolling channels when the mobile client turns off their phone, otherwise they hand all their money to the ASP at tree timeout.

I do wonder if this is a problem that can be fixed.

A model where the Ark Server and LSP is hosted by the same entity really helps users on moible phones today. It is great and I’ll be happy to contribute to building this.

However, I still feel it can’t bring us to full pie Rene is talking about. I’ve been trying to figure out if LN-symmetry can fix the situation. I would expect that with a rebind-able signature we can move the channel state from one-round to another. However, I cannot figure out to process the forfeits.

-------------------------

instagibbs | 2026-01-06 14:16:50 UTC | #8

Bringing the state from one tree to another isn’t that hard (though it becomes easier with rebindable signatures). My concern is that if the LSP is a separate entity, they will be forced to roll out the tree to unilaterally close the channel in the case where client is nonresponsive.

If the channel participants are presumed to come back online later it works fine. They can jointly do the forfeit dance and migrate.

-------------------------

