# A Mathematical Theory of Payment Channel Networks

renepickhardt | 2026-01-16 00:49:30 UTC | #1

Dear fellow lightning network developers

I’ve posted a [new paper on arXiv](https://arxiv.org/pdf/2601.04835) that formalizes several long-standing observations about payment channel networks (in particular with the lightning network in mind) under a single geometric framework:

https://arxiv.org/pdf/2601.04835

Many of the individual phenomena discussed in the paper are familiar to practitioners: 

* channel depletion, 
* capital inefficiency of two-party channels, 
* the benefits of channel factories, 
* and the idea that [feasibility rather than routing is the real bottleneck](https://delvingbitcoin.org/t/estimating-likelihood-for-lightning-payments-to-be-in-feasible/973?u=renepickhardt). 

The goal of this work was not to rediscover these effects, but to explain why they are structurally true and how they are connected.
A key outcome of the paper is the perspective that payments should be analyzed through the set of feasible wealth distributions rather than individual paths. Liquidity states differ only by circulations within fibers over a polytope. A payment is feasible if and only if the resulting wealth vector remains inside that polytope and offchain rebalancing does not change this (as was noted by others in 2018 on lightning-dev). This leads to:

* a simple throughput law `S=c/r` linking the supported off-chain payment bandwidth `S` to the rate `r` of infeasible payment attempts and onchain transaction bandwidth `c`
* a cut-based characterization of feasibility,
* a formal explanation of why multi-party channels (coinpools / channel factories) are structurally more capital-efficient,
* and a geometric explanation of [why linear asymmetric fees generically lead to channel depletion](https://delvingbitcoin.org/t/channel-depletion-ln-topology-cycles-and-rational-behavior-of-nodes/1259).

These insights directly motivated the [recent Delving Bitcoin article, which explores Ark as a channel factory](https://delvingbitcoin.org/t/ark-as-a-channel-factory-compressed-liquidity-management-for-improved-payment-feasibility/2179) and its implications for payment feasibility:

https://delvingbitcoin.org/t/ark-as-a-channel-factory-compressed-liquidity-management-for-improved-payment-feasibility/2179

The paper also analyzes [mitigation strategies for depletion](https://delvingbitcoin.org/t/mitigating-channel-depletion-in-the-lightning-network-a-survey-of-potential-solutions/1640?u=renepickhardt) such as symmetric fees, [convex/tiered fee schedules](https://github.com/renepickhardt/Lightning-Network-Limitations/blob/paper/Limits%20of%20two%20party%20channels/Effect%20of%20Convex%20Fee%20Functions%20on%20Flow%20Control%20and%20Channel%20Depletion.ipynb), and [coordinated off-chain replenishment](https://delvingbitcoin.org/t/research-update-a-geometric-approach-for-optimal-channel-rebalancing/1768) while explaining why some of these are difficult to deploy under today’s source-routing model.

For readers interested in simulations and executable intuition, much of the reasoning [was developed alongside code and notebooks](https://github.com/renepickhardt/Lightning-Network-Limitations/blob/main/README.md), which are collected here:

https://github.com/renepickhardt/Lightning-Network-Limitations/blob/main/README.md

Feedback and Questions are very welcome, especially on how these results should influence protocol design and whether multi-party channels should primarily serve as channel factories or as long-lived payment venues.

With kind regards Rene Pickhardt

-------------------------

