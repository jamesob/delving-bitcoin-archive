# Scaling Noncustodial Mining Payouts with CTV

vnprc | 2025-06-04 20:12:32 UTC | #1

GM

I have been thinking a lot about mining pools lately. It's fun because the design space of architectures, protocols, and payout mechanisms is wide open. One useful mental framework I have found is to categorize mining pools into two groups:
- pools that custody mining rewards
- pools that do not

A quick perusal of mempool.space makes it abundantly clear that every pool except Ocean is a custodial pool.

Here is a typical [custodial coinbase](https://mempool.space/tx/46f4013cefeab285fbd6bff2443070c1dfd2384284bfaa4edf531c30470be535) weighing in at 471 bytes. Notice they peel off the rare sat at the start of the block and the rest of the newly mined bitcoin flows into a single wallet address. If you click through to that address you can see it contained a whole bunch of coinbase outputs that were eventually consolidated.

![image|690x300](upload://aLavzHNTyLwIRYADOsZvewjBzZn.png)

Here is a 2.28 kilobyte [noncustodial coinbase](https://mempool.space/tx/6935c08c2ba63affab916b5d255471d30733278c5709ed3042b086339a9dcd59) containing 62 outputs.

![Screenshot from 2025-06-04 11-27-33|530x500](upload://coN1kzrC60QtBMD8pxWEx21cY25.jpeg)

At btc++ last month I attended this [excellent talk](https://www.youtube.com/watch?v=EKQvDfmQkt8&t=8910s) on the design of the DATUM protocol and I was very surprised to learn how difficult it is for Ocean to provide these noncustodial fanout transactions. The vast majority of the problem is Antminer firmware restrictions on the size of the coinbase.

In order to work around this limitation Ocean not only has to fingerprint miner hardware and keep track of multiple work templates per account but they are also unable to tightly control the coinbase of the blocks their miners produce. Open source ASIC firmware like Bitaxe's [ESP-miner](https://github.com/bitaxeorg/ESP-Miner) allows a very large coinbase transaction, limited only by the available memory on the ESP. But Antminer firmware is the most restrictive, allowing just over a dozen outputs in the coinbase.

Here is an example of a 784 byte Ocean [coinbase tx](https://mempool.space/tx/990c417e5bc97412a2e1f3197cde239e72f96e1c1a3f1fe81f48d3ba2571f4fb) containing 16 payouts. No doubt this block was produced by an ASIC appliance running handicapped firmware.

![image|690x300](upload://k2cjVyF4NG8jpu5K1hYZCeuKcAv.png)

It occurs to me that CTV nukes this problem from orbit. If CTV were active on mainnet the coinbase could include a single tiny consensus-enforced commitment to a fanout transaction of arbitrary size.

After the conference I took the time to hack together this [Coinbase Playground](https://github.com/vnprc/coinbase-playground) environment to explore the possibilities. It's rough around the edges because it was a hackathon project but I believe the concept is extremely sound.

Benefits:
1. Eliminate the ability of hostile ASIC firmware vendors to handicap non-custodial mining pools.
1. Enable non-custodial mining payouts of any size, down to the dust limit.
1. Maximize fee revenue in each block the pool mines. (The level of elegance of this incentive alignment pleases me greatly.)

Drawbacks:
1. Users must get additional transactions mined to claim their rewards
1. Someone must make the unroll transaction data available

The first transaction I scripted was designed to be a straightforward upgrade to Ocean's coinbase fanouts that prioritizes ergonomics and usability. It is a 179 byte coinbase transaction that spends to a CTV unroll transaction with up to 319 payment outputs and one anchor output. This was the maximum size fanout I could produce (10 kilobytes) before running into mempool policy limits. I set the fee rate to 1 sat/vb so the pool can immediately broadcast it to the mempool where it will sit for 100 blocks until it becomes valid to mine.

This design leverages the mempool to solve for data availability. Users can either wait for the mempool to empty out or use the anchor output to CPFP fee bump it. They could potentially use `SIGHASH_ANYONECANPAY` to crowdsource the fees. You can tweak this design in various ways, for example by dropping the anchor output or violating policy rules to add more outputs or remove the fee (although this reintroduces the data availability problem). Bike shedding aside, I think it lays a pretty solid foundation to build upon.

Next I began experimenting with nested CTV transactions. I built a very basic 4 leaf tree before I ran out of time. I think there is a very interesting design space here if we use n-of-n musig keys in the keypath. This would enable owners of the outputs to swap off-chain bitcoin for the signatures of their leaf siblings and collapse the subtrees into larger outputs with lower on-chain fees. This design, however, introduces a difficult coordination problem because you can only negotiate with the siblings of your leaf outputs.

Perhaps an even more promising avenue of research is to use CTV+CSFS to create rebindable commitments to the tree structure. In this way the output owners can spend the 100 blocks after confirmation auctioning their noncustodial coinbase outputs in exchange for off-chain payments. This would be a categorical improvement over traditional coin mixers because coinbase outputs, unlike coinjoin outputs, have no transaction history.

This design also brings significant coordination challenges because all signers must agree to state updates. Perhaps there is a middle ground using nested CTV commitments to limit the set of signers. I intend to continue exploring the possible designs in this space.

One criticism I received was that we should change the firmware rather than change bitcoin. I disagree with this argument because history has shown that it is, in fact, easier to change bitcoin than to change Bitmain hardware or firmware. (Shoutout gmax! 🐐ed.)

What do y'all think? I am very interested to receive constructive criticism or tips on designing transactions or bitcoin scripts since this is a new endeavor for me.

-------------------------

ErikDeSmedt | 2025-06-05 07:35:37 UTC | #2

We have been [thinking](https://x.com/2ndbtc/status/1926309267142832641) about a similar approach using Ark.

The key advantage of this approach is that we can reduce the transaction-cost. A small miner that operates in the scheme above will receive a clunky set of many small VTXOs. 

I don't want to hi-jack this post and make it about Ark. However, a short summary won't hurt anyone.

The mining pool can use OP_CTV to pay into a [transaction tree](https://docs.second.tech/protocol/intro/#utxo-sharing-using-transaction-trees). The leaf of this tree is called a VTXO. A miner can hold on to their VTXO's for a while until they have collected a good amount of VTXO's and offboard all funds at once.

This saves costs.

A small miner will not end up with 100 UTXO's of 0.0001 BTC.
Instead, they can offboard 100 VTXOs of 0.0001 BTC. Only a single UTXO worth 0.01 BTC will hit the chain.

-------------------------

AntoineP | 2025-06-06 14:01:07 UTC | #3

Thanks for the detailed writeup. I do not see how what you describe scales non-custodial mining payouts (the title of this post), or maximizes fee revenues (as you claim in your list of benefits of the scheme). It actually does the opposite as it increases the block space required for non-custodial payouts.

At best this is a congestion control claim, in that in a high fee period the marginal feerate earned by the pool for including non-payout transactions is so high that it can reasonably expect to maximize fee revenues by delaying the inclusion of payout outputs. But this is not a viable strategy under normal circumstances, where you would expect that increasing your block space usage would work against maximizing your fee revenues.

Using Ark for payouts instead introduces some additional assumptions but at least does make sense from a scalability perspective (as Erik points out), since you could reasonably expect the happy case to be more likely and payouts to actually end up using less block space on average. And CTV does help here in that it can [make the Ark payouts non-interactive](https://nehanarula.org/2025/05/20/ark.html#ctv): now the coordinator (the pool?) doesn't need a roundtrip with every single hasher to presign the payout tree beforehand.

If the intention of this post was to motivate a CTV soft fork, i think you should focus on how Ark could be used to scale mining pool payouts and how CTV (+CSFS) would improve it, rather than congestion control which was the original flagship usecase for CTV that failed at convincing people it's worth changing Bitcoin for.

[quote="vnprc, post:1, topic:1753"]
history has shown that it is, in fact, easier to change bitcoin than to change Bitmain hardware or firmware. (Shoutout gmax! :goat:ed.)
[/quote]

Side note: if your goal is to provide motivation for a CTV (+CSFS) soft fork, i would suggest you keep out unrealistic wild claims that twist reality to fit a certain narrative. There is an unlimited amount of interesting things to be thinking about, and your doing that may be used as a heuristic by the very people you are trying to convince to spend time on these other things rather than your ideas.

-------------------------

1440000bytes | 2025-06-06 18:11:31 UTC | #4

[quote="vnprc, post:1, topic:1753"]
At btc++ last month I attended this [excellent talk](https://www.youtube.com/watch?v=EKQvDfmQkt8&t=8910s) on the design of the DATUM protocol and I was very surprised to learn how difficult it is for Ocean to provide these noncustodial fanout transactions. **The vast majority of the problem is Antminer firmware restrictions on the size of the coinbase.**

In order to work around this limitation Ocean not only has to fingerprint miner hardware and keep track of multiple work templates per account but they are also unable to tightly control the coinbase of the blocks their miners produce. Open source ASIC firmware like Bitaxe’s [ESP-miner](https://github.com/bitaxeorg/ESP-Miner) allows a very large coinbase transaction, limited only by the available memory on the ESP. **But Antminer firmware is the most restrictive, allowing just over a dozen outputs in the coinbase.**
[/quote]

Thanks for writing this post. I have added Coinbase Playground in https://en.bitcoin.it/wiki/Covenants_Uses

IIUC this solves 2 problems without involving ASP:

1. Antminer [coinbase size limit](https://x.com/GrassFedBitcoin/status/1832473863537717308)
2. Leaves room to add more transactions in block (i.e. more fees)

Note: Miners can use [ark on-chain](https://docs.second.tech/getting-started/barking-on-signet/advanced/) addresses as payout addresses.

-------------------------

jamesob | 2025-06-08 16:51:36 UTC | #5

[quote="AntoineP, post:3, topic:1753"]
I do not see how what you describe scales non-custodial mining payouts
[/quote]

Without this technique, a pool operator must literally act as a custodian until the payout happens. L2 use is possible, but not necessarily economical depending on the cost for unilateral exit.

[quote="AntoineP, post:3, topic:1753"]
Using Ark for payouts instead introduces some additional assumptions but at least does make sense from a scalability perspective
[/quote]

We have yet to see how expensive in practice the unilateral exit case is from an Ark, which would determine this claim.

[quote="AntoineP, post:3, topic:1753"]
But this is not a viable strategy under normal circumstances, where you would expect that increasing your block space usage would work against maximizing your fee revenues.
[/quote]

"Normal circumstance" for the last few months has been that the mempool is empty during weekends. I'm not sure if you're familiar with the (old?) argument that when block revenue is dominated by fees and if the mempool goes empty, the chain will struggle to make progress as miners simply reorg one another to grab fees.

This strategy, and congestion control in general, helps that situation by allowing fee smoothing over some period of time without introducing a custodial reliance.

I'm not sure why you'd be so hostile to having another unique "tool in the toolbox" for not only decentralizing mining, but helping to address a profound and incoming issue with bitcoin as the block subsidy goes away.

-------------------------

AntoineP | 2025-06-08 22:18:03 UTC | #6

[quote="jamesob, post:5, topic:1753"]
I’m not sure why you’d be so hostile to having another unique “tool in the toolbox” for not only decentralizing mining
[/quote]

I am only hostile to bad arguments for potentially good ideas, of which your motivated reasoning here and on other issues (CTV ""vaults"") is an example, because it makes good ideas less likely to become reality and bad ideas more difficult to uncover as such.

[quote="jamesob, post:5, topic:1753"]
[quote="AntoineP, post:3, topic:1753"]
I do not see how what you describe scales non-custodial mining payouts
[/quote]

Without this technique, a pool operator must literally act as a custodian until the payout happens. L2 use is possible, but not necessarily economical depending on the cost for unilateral exit.
[/quote]

No, it can just include payouts in the block directly. Using CTV to commit to the payouts only defers the time at which the payout transaction hits the chain, at the cost of more block space usage. It is not a scalability solution.

[quote="jamesob, post:5, topic:1753"]
We have yet to see how expensive in practice the unilateral exit case is from an Ark, which would determine this claim.
[/quote]

Sure, it's not guaranteed but at least is not literally proposing to use more block space as a scalability solution.

[quote="jamesob, post:5, topic:1753"]
This strategy, and congestion control in general, helps that situation by allowing fee smoothing over some period of time without introducing a custodial reliance.
[/quote]

I am aware of the arguments for CTV-based congestion control. I was just pointing out that OP is merely proposing to use congestion control for mining pool payouts, and that it is incorrect to claim this is a scalability solution.

-------------------------

jamesob | 2025-06-09 15:22:36 UTC | #7

[quote="AntoineP, post:6, topic:1753"]
No, it can just include payouts in the block directly.
[/quote]

This steals fee revenue from otherwise revenue-generating blockspace (unless the pool constituents are paying the miner fees somehow), especially for large numbers of pool participants.

Using a CTV payout allows the pool to collect maximum fee revenue in the moment, and then allows each pool constituent to individually decide how much they're willing to pay in fees (and when) to take receipt of the payout.

-------------------------

vnprc | 2025-06-09 17:17:42 UTC | #8

> Thanks for the detailed writeup.

You are welcome! It's my pleasure to explore and write about how CTV materially improves bitcoin. Thank you for reading and commenting.

> I do not see how what you describe scales non-custodial mining payouts (the title of this post), or maximizes fee revenues

We can define non-custodial mining payouts as such: newly created UTXOs in the coinbase of a bitcoin block that are unable to be spent by any party other than the intended recipient of that payout. Every large pool today, with the exception of Ocean, sends the coinbase outputs to a custodial wallet and renders payouts from that wallet. This means the pool operator could rug their miners by simply not transferring the bitcoin from their wallet to the miner's wallet. In this sense, Ocean is a non-custodial pool for all miners who receive a coinbase output in a given Ocean block.

Ocean is limited in the number of these outputs they can produce each block by two factors. The first is a hard limit: Some ASIC appliance firmware places a limit on the size of the coinbase. The second is a soft limit: since the coinbase counts against the maximum block size limit, pools are incentivized to use the smallest possible coinbase in order to maximize fee revenue in each block.

CTV sidesteps both limitations. It enables an effectively unlimited number of non-custodial payouts in each block mined while improving pool profitability compared to the status quo. In other words, CTV scales non-custodial mining payouts.

> At best this is a congestion control claim

This is not a congestion control claim. The heart of my argument is that CTV unfetters trustless mining payouts. Trusted systems are a security vulnerability, as asserted by Satoshi in the first two sentences of the white paper. CTV can fundamentally improve the trust model of bitcoin mining. Proof of work mining is the foundation of bitcoin's trust model. Therefore, CTV can be used to fundamentally improve the trust model of bitcoin.

> increasing your block space usage would work against maximizing your fee revenues

I'm not sure how you reach this conclusion. You appear to be considering total block space usage for the bitcoin network in the context of a single mining pool. This doesn't translate. CTV can very slightly increase total blockspace usage but this is not a concern for a mining pool operator unless they are producing all of the blocks on the network. In practice, every custodial pool externalizes their payout costs to future blocks using fanout transactions. My design is only different in the trust model used.

> Using Ark for payouts instead introduces some additional assumptions but at least does make sense from a scalability perspective

I am very bullish on Ark for payments. I'm also very excited about how it can be used to improve mining operations and scalability. Again, you are talking about scalability of the entire bitcoin system. In this post I am focused on scaling trustless mining payouts beyond the limitations it faces under current consensus rules.

> If the intention of this post was to motivate a CTV soft fork

My intention is to improve bitcoin's chance of long-term success as a self-sovereign monetary commodity by improving the mining stack. I believe that mining pool centralization is currently the biggest threat to bitcoin and therefore this is the highest impact work I can do. I did not wade into the CTV debate earlier because I didn't have a concrete proposal in an area of domain expertise. (This was the case the last time we talked.)

I have chosen a domain. I am working hard to gain practical expertise. And I have found (I believe) a very compelling use for CTV within my domain. I plan to continue building use cases for CTV+CSFS, as indicated at the end of the github readme. If you still question my motives you can also reach out to me personally. I'm always happy to talk to a respected core developer such as yourself.

> unrealistic wild claims that twist reality to fit a certain narrative

Greg Maxwell backward engineered a Bitmain ASIC appliance to demonstrate how it was capable of covert ASIC boost. This is what settled the block size debate for me personally by revealing the motivations of the primary economic actor opposing the segwit soft fork.

Covert ASICboost was obsoleted by the segwit soft fork, demonstrating to me that it is easier to change bitcoin than to change Bitmain firmware or hardware. This is a subjective claim and, as such, it cannot be proven or disproven. You can choose to agree or disagree. I only made this statement in response to a subjective criticism of my proposal.

I don't believe we can fix the deep and long-running problems in the bitcoin mining stack by being ignorant of history or naive about the motivations of the largest actors in this space. My claims are not unrealistic or wild and they do not distort reality or promote a narrative other than that which is supported by the historical facts.

Here is some reading material if you don't believe me.

Greg's initial claim about reverse engineering the ASIC:
https://gnusha.org/pi/bitcoindev/CAAS2fgR84898xD0nyq7ykJnB7qkdoCJYnFg6z5WZEUu0+-=mMA@mail.gmail.com/

This reddit thread shows achow confirming with greg over IRC how he did it:
https://www.reddit.com/r/Bitcoin/comments/63soe3/comment/dfx9ncr/

The smoking gun, IMO, was when Braiins shipped the option to activate covert ASICboost on their S9 firmware.
https://www.ccn.com/under-pressure-bitmain-releases-patch-to-let-bitcoin-miners-activate-overt-asicboost/

-------------------------

vnprc | 2025-06-09 17:45:40 UTC | #9

@1440000bytes Thanks for adding this to your covenants wiki.  And thank you for taking on the job of cataloging this debate. I am very appreciative of your efforts!

> IIUC this solves 2 problems without involving ASP:

I would characterize it as solving three problems or perhaps solving the two problems you mentioned but having three distinct positive outcomes:
1. Eliminate the influence of ASIC firmware vendors on the viability of non-custodial mining payouts.
2. Remove the effective cap on the number of payouts in each coinbase. This serves to reduce the minimum on-chain payout threshold of non-custodial pools down to the dust limit.[^1] It is currently very much higher than the dust limit at Ocean.
3. Remove the disincentive for non-custodial pools to limit the number of payouts in order to increase transaction fee profits.



[^1]: For those who don't read carefully (you really should): "down to the dust limit" does not imply a preference for the size of these outputs. This phrase only establishes the practical lower limit of those payouts.

-------------------------

vnprc | 2025-06-10 17:04:12 UTC | #10

Oh hey you can link quotes to the original. Delving is pretty sweet!

[quote="AntoineP, post:3, topic:1753"]
i think you should focus on how Ark could be used to scale mining pool payouts and how CTV (+CSFS) would improve it
[/quote]

[quote="ErikDeSmedt, post:2, topic:1753"]
The mining pool can use OP_CTV to pay into a [transaction tree](https://docs.second.tech/protocol/intro/#utxo-sharing-using-transaction-trees). The leaf of this tree is called a VTXO. A miner can hold on to their VTXO’s for a while until they have collected a good amount of VTXO’s and offboard all funds at once.
[/quote]

I am convinced. I need to look more closely at the noncustodial nature of ark. If the security guarantees are comparable to a basic CTV payment tree in then it would be a natural upgrade to reduce the on-chain footprint of small payouts overall.

-------------------------

marathon-gary | 2025-06-10 20:14:16 UTC | #11

RE: congestion control

Isn't this what Lightning Network accomplished to a degree? It deferred on-chain settlement in payment channels so users would not have to use the base chain.

I also would imagine miners being able to "unroll" any tree they're in (Ark or bare CTV) with a 0 fee transaction in their block template.

-------------------------

AntoineP | 2025-06-10 21:22:12 UTC | #12

[quote="marathon-gary, post:11, topic:1753"]
RE: congestion control

Isn’t this what Lightning Network accomplished to a degree? It deferred on-chain settlement in payment channels so users would not have to use the base chain.
[/quote]

No, Lightning channels *compress* an infinite number of payments to a couple of onchain transactions (in the optimistic case). Congestion control keeps the same number of onchain transactions (in fact it adds some more) but allows their inclusion in the block chain to be deferred (and spread out).

-------------------------

marathon-gary | 2025-06-10 22:49:48 UTC | #13

I see the distinction you're making but a LN channel is still a deferment of final settlement. I think a better contrast would be LN channels defer an *unknown final state* within the bounds of channel capacity, while a CTV tree defers a *known final state*.  Although a CTV tree could also have within it [batch channels](https://utxos.org/uses/batch-channels/) as well, so a CTV tree could hold both known and unknown final state of the deferred transactions.

In the case of mining pool payouts, the coinbase CTV tree could be optimistically updated every block the pool wins, assuming interactivity of the participants (LN assumes interactivity of participants too) by leveraging MuSig for the various nodes of the CTV tree. If all participants are online and interacting, the entire CTV tree state could be updated without the need to exit from the construct. Since mining is boolean, you're mining or you're not, a miner is already active with the pool so could optimistically participate in any protocol that requires interactivity.

Furthermore, since miners have the ability to make their own block templates, a miner could always include a 0 fee transaction paying themselves out of the CTV tree in the next block that is found. 

[quote="vnprc, post:8, topic:1753"]
In practice, every custodial pool externalizes their payout costs to future blocks using fanout transactions. My design is only different in the trust model used.
[/quote]

-------------------------

plebhash | 2025-06-13 15:01:56 UTC | #14

Some words about how this relates to Stratum V2.

Recently, we had some discussions on Coinbase Output allocation on the spec repo: https://github.com/stratum-mining/sv2-spec/pull/138

We realized that there's a there's a chicken-and-egg problem that's very hard to solve without introducing new round-trip messages into the canonical Sv2 Job Declaration (JD) process.

In the bootstrapping phase of the Sv2 JD, the following Sv2 messages need to happen in sequence:
- [`AllocateMiningJobToken`](https://github.com/stratum-mining/sv2-spec/blob/66287a5d9e8d94a681febadcaffa6e141e3cf4b4/06-Job-Declaration-Protocol.md#642-allocateminingjobtoken-jdc---jds) (JD Client / Miner -> JD Server / Pool)
- [`AllocateMiningJobToken.Success`](https://github.com/stratum-mining/sv2-spec/blob/66287a5d9e8d94a681febadcaffa6e141e3cf4b4/06-Job-Declaration-Protocol.md#643-allocateminingjobtokensuccess-server---client) (JD Server / Pool -> JD Client / Miner)
- [`CoinbaseOutputConstraints`](https://github.com/stratum-mining/sv2-spec/blob/66287a5d9e8d94a681febadcaffa6e141e3cf4b4/07-Template-Distribution-Protocol.md#71-coinbaseoutputconstraints-client---server) (JD Client / Miner -> Template Provider / Miner)

`AllocateMiningJobToken.Success.coinbase_tx_outputs` is how JD Server / Pool informs the bare minimum information that it expects to see in the Coinbase Outputs in order to acknowledge work. At this point, neither Pool nor Miner know the template revenue, because the Template Provider still wasn't informed about the blockspace and sigops budget available for creating the template (that happens over `CoinbaseOutputConstraints`).

So we established a convention where the first output of `AllocateMiningJobToken.Success.coinbase_tx_outputs` dictates where the pooled mining goes. This output is informed as a 0 valued output, and it is up to the JD Client / Miner to fill it on subsequent JD messages (`DeclareMiningJob` and `SetCustomMiningJob`) with the amount of sats it wants to allocate towards pooled mining.

This is enough to establish a sequence of causality in the chain of messages listed above. JD Client has enough data to inform the Template Provider about the available blockspace and sigops budget via `CoinbaseOutputConstraints`, and later it can continue with the JD process without violating the agreements with the Pool.

This is however a compromise that sacrifices non-custodial payouts under the canonical Sv2 JD spec. Pool / JDS cannot determine non-custodial payouts without knowledge of template revenue. Miner / JDC cannot inform template revenue without knowledge of blockspace/sigops budget. 🐔🥚.

Some protocol extension would be needed to unblock non-custodial payouts, be it multi-output or CTV-based.

-------------------------

vnprc | 2025-06-13 16:11:25 UTC | #15

Thanks for looking into this @plebhash. I'm very interested in exploring this further because I view non-custodial payouts as a high priority feature for my project.

It seems like the pool's ability to require certain transactions in the template is interfering with the possibility of non-custodial payouts within the current spec.

If the pool simply forgoes this option and leaves the template wide open for miners to fill, wouldn't that solve the problem?

Miners could use the known fixed-size CTV coinbase to non-interactively calculate sigops and blockspace budget.

-------------------------

vnprc | 2025-06-13 16:33:29 UTC | #16

@marathon-gary I didn't respond to your LN post because it's off-topic in this thread and I think it deserves a dedicated Delving post.

The TL;DR of that post (which I haven't written yet) is that Poon-Dryja channels with block producers are fundamentally insecure. I think Spillman channels are a superior choice for pool payouts.

I'll take some time in the next few days to write up my thinking.

-------------------------

plebhash | 2025-06-13 20:17:11 UTC | #17

but then where would the consensus for reward distribution happen?

what you're saying sounds like decentralized consensus, but as far as I've looked into this, it's very non-trivial problem to solve, which p2pool and braidpool are doing their own compromises to achieve. 

also, those are projects where Sv2 JD don't really serve much of a purpose (Sv2 Mining Protocol could benefit them, but not really relevant on this discussion).

I heard the Ocean folks talking about moving DATUM in a direction where miners are spot-checking eachother work, but I'm yet to fully assimilate how that would look like, and also how that would scale up.

the point about having a centralized pool is that it dictates how the coinbase must ultimately look like (be it custodial or non-custodial payouts).

in canonical Sv2 JD, we try to simplify the problem as much as possible, by saying that there will be only one single output... it's up to the miner to take the template revenue and allocate how many sats it wants to this specific output.

a protocol extension would introduce new messages that (at the cost of extra round-trips) to allow Miner to inform the template revenue (or revenue to be allocated towards pooled mining), and then be informed about the non-custodial coinbase output that will qualify valid work.

I agree that the part about sigops + blocksize budget is relatively trivial, and could even be something the miner informs its TP without the need for any dedicated message (since it's kind of statically defined)... but the point is that would diverge from the message flow that's already established on the canonical Sv2 JD

-------------------------

