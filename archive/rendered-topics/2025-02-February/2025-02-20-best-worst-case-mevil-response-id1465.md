# Best-(Worst-)Case MEVil Response

MattCorallo | 2025-02-20 16:08:54 UTC | #1

Hi all,

@7d5x9 and I spent some time thinking about what the best we can do in response to a future where MEVil on Bitcoin proliferates and recently posted our results at [1]. Basically, we propose an extension of the PBS direction ethereum went down where instead of proposers proposing full block templates they only bid for individual (or group) transaction inclusion with constraints (like "I want to be the first transaction included which interacts with the DEX identifiable by transactions which include the string X"). This ensures that the public mempool continues to be a reliable way to get a transaction included (as miners would fill the their block from both the public mempool and any private orders), providing censorship resistance similar to today, while still allowing MEVil-extraction experts to pay miners for specific transaction inclusion.

We also propose that the marketplaces offer private transaction inclusion by either having miners trust the marketplaces or by using TEEs to "encrypt" transactions such that they can only be read after a valid block meeting the constraints has been found. Interestingly, this also allows users of MEV-exposed systems to submit their transactions directly to miners for inclusion, reducing their exposure to frontrunning and other MEV issues.

Obviously this design is highly centralizing compared to today's bitcoin, and a reasonable response might be "we should strive for a world where we don't need this garbage", but IMO we should be clear-eyed about what a bitcoin looks like if MEVil becomes a major part of the mining process, and build for that outcome just in case it comes to pass.

Let us know what you think.

[1] https://github.com/mevpool/mevpool/blob/0550f5d85e4023ff8ac7da5193973355b855bcc8/mevpool-marketplace.md

-------------------------

ajtowns | 2025-02-20 17:57:10 UTC | #2

> For example, a Marketplace would allow for bids in the form of “I’ll pay X to include transaction Y as long as it comes before any other transactions which interact with the smart contract identified by Z”.

I think a "bid" of that form would be unpleasant if passed all the way through to miners -- accepting the bid but still including transaction W that interacts with smart contract Z means (in general) modifying transaction W to spend an output of Y (or perhaps some child of Y), and potentially updating the witness data of W to deal with the difference in value of Y's output vs Z's output. To me, those seem like things that the "proposer" should be doing, rather than the "relay" or the "builder".

Would it be plausible to make the "mevpool marketplace" operate solely on packages of transactions, that is:

```mermaid height=148,auto
 flowchart LR
    Extractor1 --tx pkg--> Marketplace
    Extractor2 --tx pkg--> Marketplace
    Marketplace --{tx pkgs}--> Miner1
    Marketplace --{tx pkgs}--> Miner2
    Miner2 --block--> Network
```

So that extractors who want their tx Y to be first monitor the mempool for other txs that interact with smart contract Z, setup a tx pkg that does that and submits it to the marketplace. Other extractors might submit competing packages, bidding fees up, and the marketplace sends the proposed packages to miners who build block templates, etc.

For sealed bids, I think you could reveal with utxos the package spends, the txids and wtids, and the total sigops, weight, and fee, and then either rely on trusting the marketplace to reveal the tx contents or a trusted execution environment in a similar way.

This assumes miners can do package RBF in a fairly optimal manner, and that we can add "sealed txs" to the mempool, but otherwise seems to keep things fairly simple on the miner/marketplace side.

Revealing the utxos that sealed txs are spending might be something of a give away to competitors, but seems unavoidable if you want miners to be building the block templates.

-------------------------

ariard | 2025-02-20 18:09:09 UTC | #3

Sorry, what is a MEVil exactly ?

"*Specifically, MEV which results in a financial incentive for miners to employ sophisticated technology in order to ensure the transactions they include in the next block have the maximal value is MEVil*”.

This is neither a definition in terms of structural aspects (e.g exploiting a base-layer policy mechanism against the interest of L2 counterparties), neither a quantitative premium in terms of block template efficiency construction and neither said what is "sophisticated technology" exactly. Is applying rigorously high school integral calculus to build your block template considered as “sophisticated technology” ?

"*We believe that these two issues can be addressed through the construction of a "mevpool marketplace" standard that takes the place of accelerator/private mempool services, applying PBS only for a narrow subset of transactions in a block. This enables the miner to remain the block "builder" while still allowing for end users or third-parties specialized in MEV extraction to directly bid for individual transaction inclusion. The Marketplaces will host order books containing bids with varying properties, including position within blocks, restrictions in relation to individual smart contracts, etc.*”

How can you be sure that you're receiving the block template in real-time from the marketplace operators, not even saying how do you verify builder's block templates have not been selectively marginally downgraded in terms of feerate by weight unit ?

No proposed seal-based solutions are realistic, for the 1st one the only dry escrow mechanism is (1) old school bank's letter of credits... and for cypherpunk trust-minized escrow (2) anything which is bitcoin-script based though any 2-phases commit protocol would have to fit in 2 transactions included in 2 blocks at least, so in 20 min in average to settle a dispute on the correctness of a data structure (i.e block-template) that can happen every 10 min in average in the worst-case.

For the 2nd one, running templating infrastructure in a thing like a TEE is the last thing I would do as in block templating it's a latency-race as anyone who has worked on BIP152 knows it. That's just so many more kernel context-switch due to enclave transisition that you're better off to re-design your own commitment attestation scheme to fit bitcoin block entropy structure and get native support on enclave HW...Good luck with that.

I do not disagree on the problem, and I do think it's an important problem.

However, I don't think the proposed solutions are improving the problem at all, or even worsen it.

-------------------------

MattCorallo | 2025-02-20 19:23:29 UTC | #4

[quote="ajtowns, post:2, topic:1465"]
I think a “bid” of that form would be unpleasant if passed all the way through to miners – accepting the bid but still including transaction W that interacts with smart contract Z means (in general) modifying transaction W to spend an output of Y (or perhaps some child of Y), and potentially updating the witness data of W to deal with the difference in value of Y’s output vs Z’s output.
[/quote]

Mmm, this is a good point, and I had mostly been thinking of this in the context of things like CSV (and CSV-based rollup) protocols, where this likely doesn't apply (at least if they're EVM-like). However,

[quote="ajtowns, post:2, topic:1465"]
Would it be plausible to make the “mevpool marketplace” operate solely on packages of transactions, that is:
[/quote]

Yea, sorry, somewhat shorthand here this is indeed how I've been thinking about bids. A proposer would be able to submit one or more transactions in a single bid, and indeed would be able to do what you describe here.

[quote="ajtowns, post:2, topic:1465"]
For sealed bids, I think you could reveal with utxos the package spends, the txids and wtids, and the total sigops, weight, and fee, and then either rely on trusting the marketplace to reveal the tx contents or a trusted execution environment in a similar way.
[/quote]

Yep, though you can pass this all the way through to miners just as well. Basically you'd move bids to always be "the full set of transactions which interact with a contract" rather than ever being a subset of them (for contracts that rely on state passed via UTXOs) and then you can send the whole tx bundle through to the miner's TEE (or, indeed, just reveal the metadata if you're using a trusted marketplace).

[quote="ajtowns, post:2, topic:1465"]
This assumes miners can do package RBF in a fairly optimal manner
[/quote]

Indeed, though luckily this is somewhat simplified in that the miner is required to only either include or not-include the whole package, they cant split it into parts. They just have to be able to compare that to their existing mempool, which is hard but if we're talking about a server-class processor with SGX you can throw some CPU at it and get close enough to optimal that I'm not worried :).

[quote="ajtowns, post:2, topic:1465"]
Revealing the utxos that sealed txs are spending might be something of a give away to competitors, but seems unavoidable if you want miners to be building the block templates.
[/quote]

This is part of the reason for the TEE - if we can take the whole block building logic and shove it in SGX, we can run it locally at the miner but also send it Sealed transaction (packages) which can be considered for the block template but won't be extractable (at least for reasonable cost).

-------------------------

7d5x9 | 2025-02-20 19:44:44 UTC | #5

Hello ariard, thank you for your comments.

> Sorry, what is a MEVil exactly ?

For some further specificity, please refer to the mevpool writeup linked in the OP:

> Instead of consensus-computable variables (fees, weight, sigops) being the sole data required for block templating, they become a subset. Miners would require in-house financial engineers capable of analyzing on-chain contracts and designing and deploying value extraction schemes.

You might also find further understanding by studying the MEV vectors available across the Ethereum ecosystem which are fairly well documented. But more generally you can think of MEVil, distinct from pure MEV (e.g. RBF), as transaction sequencing that relies on generalized financial sophistication (i.e. what you might find at a HFT firm with quants), domain-specific understandings of smart contracts on the Bitcoin blockchain, as well as access to significant liquidity to take advantage of MEVil opportunities.

> For the 2nd one, running templating infrastructure in a thing like a TEE is the last thing I would do as in block templating it’s a latency-race as anyone who has worked on BIP152 knows it.

Unlike Intel SGX, modern TEEs (e.g. TDX) are near-native guest performance so the latency issue described is negligible. There is some research available on this, an example of which you might find here: https://arxiv.org/html/2408.00443v1

> How can you be sure that you’re receiving the block template in real-time from the marketplace operators

Yes this is a significant hurdle to Proposal One, and thus in our opinion it is sub-optimal. That said, if the Marketplace's payout depend on the block being accepted by the network, you might assume the incentives are aligned to ensure quick delivery.

-------------------------

ariard | 2025-02-21 23:58:35 UTC | #6

Hello 7d5x9, thanks for your comments.

(— and Welcome ! As there is an automatic message in delving forum saying we should welcome newcomers, and not just “trash” them for being newbies)

[quote="7d5x9, post:5, topic:1465"]
For some further specificity, please refer to the mevpool writeup linked in the OP:
[/quote]

I did `cat mempool-marketplace.md | grep -C 0 “MEV”` and `cat mempool-marketplace.md | grep | -C 0 extraction”` I didn’t find *any definition*. More, it’s underlying that in what is generally understood as *miner-extracted value* there is a lot of repetition in the markdown write-up of the word “extraction in expression like “MEV extraction”.

Traditionally in physics, an idea with some heuristic worthiness is a *clear* *definition* we can *measure*.

[quote="7d5x9, post:5, topic:1465"]
You might also find further understanding by studying the MEV vectors available across the Ethereum ecosystem which are fairly well documented. But more generally you can think of MEVil, distinct from pure MEV (e.g. RBF), as transaction sequencing that relies on generalized financial sophistication (i.e. what you might find at a HFT firm with quants), domain-specific understandings of smart contracts on the Bitcoin blockchain, as well as access to significant liquidity to take advantage of MEVil opportunities.
[/quote]

Never seen an Ethereum paper with a consistent def of MEV so far, though if there is an existent one, feel free to share it. That’s the issue you’re pointing by saying there is MEVil on one side and MEV on the other side by giving RBF as an exemple, as any block template optimization can make genuine assumptions on RBF hard constants bip125 rules. There is no need to be a quant at a HFT firm and even there when someone says “generalized financial sophistication” most of the time it’s not serious math, just being a vetted liquidity provider in a regulated market class.

*Domain-specific* understanding of smart contracts on the Bitcoin blockchain. I don’t know what is a "smart contract" on Bitcoin, generally I prefer a bitcoin contract to be *dumb* i.e the execution of which being understood by its own designers…Though let’s say very roughly assume 3 to 6 months to learn the in and outs of a Bitcoin contracting protocol, of a class of complexity under Lightning.

Significant liquidity this is something we can *measure*, though you don’t give any n*umerical value* denominated in bitcoin units here.

[quote="7d5x9, post:5, topic:1465"]
Unlike Intel SGX, modern TEEs (e.g. TDX) are near-native guest performance so the latency issue described is negligible. There is some research available on this, an example of which you might find here: [An Experimental Evaluation of TEE technology Evolution: Benchmarking Transparent Approaches based on SGX, SEV, and TDX](https://arxiv.org/html/2408.00443v1)
[/quote]

No way for the latency penalty, with the flush of the memory pages tables. The paper doesn’t even say in its index how much memory-management is the significant problem in virtualization. So I’m not sure the author of the papers understands well themselves the perf problem.

[quote="7d5x9, post:5, topic:1465"]
Yes this is a significant hurdle to Proposal One, and thus in our opinion it is sub-optimal. That said, if the Marketplace’s payout depend on the block being accepted by the network, you might assume the incentives are aligned to ensure quick delivery.
[/quote]

Again, that’s where there is a fundamental bottleneck with Proposal One. If you go for a cyperpunk trust-minimized escrow system, the best people have come up in decades of Internet is Bitcoin Script on the bitcoin blockchain, and here you have to do a 2-phases commit protocol on the *result* of a block template commitment at tip N that can only be *arbitrated* at best at tip N+1 in a *probabilistic* fashion not theory number-based sec model (let’s wave for now *selfish mining* that could let the Marketplace operator do “insider trading” against the miners participating in the marketplace).

More fundamentally, you’re pointing out a problem which you’re defining in HFT-style fashion, of which liquidity is an advantage factor and you’re brining as a solution a Marketplace solution, where miners are going to *front* liquidity for complete uncertain results. There is something that doesn’t hold, and miners are better to invest their existent liquidity in improving their chips.

At the end, it’s all like just re-inventing the centralization pressure of mining pools in face of the payout variance issue affecting bitcoin miners.

All that said - I do agree that block template construction or its potential jamming is a problem, though so far the only line of solution I’m convinced of can be fruitful is *pure algorithmic one* like the cluster mempool idea in core aims to do.

-------------------------

Laz1m0v | 2025-03-10 12:32:46 UTC | #7

# Agent Networks: An Alternative Approach to MEV Mitigation

Hi all,

I've been thinking about Matt and @7d5x9's marketplace proposal, and while it makes a lot of sense as a pragmatic approach, I wonder if we're overlooking an alternative that might better preserve Bitcoin's decentralization properties.

## Agent-based MEV mitigation

What if, instead of creating marketplaces that centralize order flow, we deployed networks of specialized software agents that could collectively identify and neutralize MEV opportunities? The basic architecture would look like:

1. **Monitoring agents** deployed across the network that detect potential MEV extraction patterns
2. **Response agents** that dynamically adjust transaction parameters to make extraction unprofitable 
3. **Coordination protocols** that allow these agents to share information without creating central points of failure

This approach leverages the same insights from the marketplace proposal - that MEV requires visibility into the transaction flow and the ability to sequence transactions - but addresses these at the protocol layer rather than through intermediaries.

## How this would work in practice

Imagine a user submitting a DEX transaction that might be vulnerable to frontrunning. Instead of routing through a marketplace, their wallet would coordinate with a network of monitoring agents that:

1. Scan the mempool for patterns indicating potential extraction opportunities
2. Compute optimal fee strategies to make extraction unprofitable 
3. Adjust transaction timing and parameters automatically

Users gain MEV protection without having to trust a marketplace operator. The system becomes more effective over time as agents learn extraction patterns and develop countermeasures.

## Comparison to the marketplace approach

Compared to Matt's proposal, this approach:

- **Advantages**: Preserves decentralization, doesn't require trusting marketplace operators, potentially more adaptive to new extraction techniques
- **Disadvantages**: More complex implementation, likely slower to deploy, requires coordination among multiple agents

## Technical feasibility

Many of the components required for this already exist in some form - mempool monitoring tools, statistical analysis of transaction patterns, and dynamic fee estimation. The challenge is integrating these into a cohesive system.

We could start by developing simple monitoring agents that only identify the most common extraction patterns, then gradually expand capabilities as the system proves effective.

## Next steps?

If people think this approach has merit, I'd be happy to work on a more detailed specification. We might be able to implement a basic version without any consensus changes, similar to how Matt's marketplace could operate outside the core protocol.

What do you think? Is this agent-based approach worth exploring as an alternative or complement to the marketplace model?

-------------------------

ajtowns | 2025-03-11 03:19:59 UTC | #8

[quote="MattCorallo, post:1, topic:1465"]
We also propose that the marketplaces offer private transaction inclusion
[/quote]

One thing I wonder is if a structure like this could also be used to mitigate some mempool/policy attack patterns.

If you have the same setup of a "marketplace" recommending packages of txs to miners, perhaps as well as MEV extractors proposing packages of txs, you could have "watchtowers" proposing transactions to the marketplace.

That could be an improvement on what watchtowers can do with existing p2p communication if the marketplace offers direct feedback to bidders when their proposed tx package is replaced in the marketplace's recommendation to miners. That would perhaps be better than just having a watchtower on its own by allowing some specialisation: marketplaces are good at monitoring the mempool and not having their view of transactions censored by an attacker, while the watchtowers only need to be good at knowing how to deal with one specific protocol.

The marketplace could potentially also track recently conflicted bids and reinclude them if they cease being conflicted, which might be an effective mitigation to replacement cycling attacks all on its own. That's possibly easier to do with bids that go through some limiting auth/payment process, rather than generic txs that appear on the p2p network.

(That wouldn't help if all marketplaces were colluding in the replacement cycling attack, of course; it may also not be great for anonymity, if you need to dox yourself in some way to interact with the marketplace)

-------------------------

instagibbs | 2025-03-11 15:29:15 UTC | #9

[quote="ajtowns, post:8, topic:1465"]
The marketplace could potentially also track recently conflicted bids and reinclude them if they cease being conflicted, which might be an effective mitigation to replacement cycling attacks all on its own.
[/quote]

Building the machinery to track these cycles is basically all the work and shouldn't be that prohibitive to do p2p. I think more interesting might be side-stepping RBF incremental fee pinning, presuming the actual incentives for those replacements are worked out to a miner's satisfaction.

-------------------------

sjors | 2025-05-21 13:55:10 UTC | #10

[quote="ariard, post:6, topic:1465"]
Traditionally in physics, an idea with some heuristic worthiness is a *clear* *definition* we can *measure*.
[/quote]

A reasonable definition of MEV could be along the lines of, a miner:
1. choosing between two or more transactions that spend the same inputs; or
2. withholding a transaction from the public mempool, only revealing it by (optional) inclusion in their own block

MEVil then is the subset that can't be handled by Bitcoin Core and/or alternative open source software with similar resource consumption. Those extra resources could just be more powerful computers, or - worse - human expertise (not just financial engineers, even a mere devops person).

How much more? Perhaps such that for a small 10kW miner the additional capital expense is less than 5% and operating expense less than 1%.

So several hundred dollars extra equipment and 0.1 kW extra power consumption. At  $0.05 / kWh such an operation might spend $4K per year on electricity, which I use as a proxy of revenue. That's nowhere near enough to hire a devops person. Perhaps a 20 MW operation could find and afford a skilled parttime devops person in a low wage country for $10k per year and stay under this 1%.

One could argue whether this skill needs to exist at the individual mining operator level or if it's enough that (many, small) pools have them. In that case adjust the above, taking into account that pools have tighter (?) margins.

(and you'd lose Stratum v2 style miner block template creation)

The MeVil threshold then, is a parameter space that one should specify.

This definition would exclude large (non-standard) `OP_RETURN` transactions, since regardless of what Bitcoin Core does, there's going to be a patched version of Bitcoin Core that can relay and mine them at negligible extra resource cost. 

Some drivechains could be MeVil, if they require software that processes each individual drivechain in order to make economically optimal decisions, e.g. to decide whether to vote for or against an attempted theft withdrawal. In the above example threshold, it wouldn't be if it had trivial consensus rules with small and slow blocks.

But it would be MeVil, depending on your choice of parameters, but certainly in my example, if the Drivechain required the equivalent of fully validating the Ethereum blockchain.

A similar analysis applies to Bitcoin miners that are also STX miners.

Having to run a bunch of these extra nodes could quickly become an operational headache even if the computers required aren't huge, and even if there was a simple script out there to install them all. Each such node could have its own DoS (and theft) issues that add risk to the Bitcoin mining operation if not properly isolated.

Very quickly you need a devops person (having these skills yourself still involves opportunity cost). The STX example also suggests that sometimes you need to be in the know of shenanigans that you can perform that your software won't do by default. This may require hiring an expert, or maybe a SAAS subscription to a company that remotely configures your tooling.

The withholding aspect of my definition (2) seems more difficult to parameterise. The software doing this doesn't have to be complicated; it can just take a payment over lightning. But it requires some sort of (reputation) marketplace to coordinate. If such a market is permissioned like MEVpool, that's not ideal.

From the proposal:

> With or without covenants, protocols with MEV are likely to increase in the Bitcoin network.

This touches the big open question, to me anyway, as to whether any of the new functionality provided by (powerful) covenant opcodes such as OP_CCV require significant extra resources, compared to what's already possible.

I'd like to see more concrete examples of that, with the caveat that for any example one can dismiss as not MeVil, there might be an unknown unknown other system.

Although one needs to be careful with fatalistic reasoning.

-------------------------

