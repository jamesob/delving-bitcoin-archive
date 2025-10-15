# LN routing and caveats

aqua | 2025-09-22 13:20:14 UTC | #1

Need someone to tell if anything wrong with my understanding and fill the gaps

I want to send 1 BTC from **Alice** to **Bob** using Lightning, **Alice** does not have a direct channel with **Bob**. In practice me (**Alice**) need to open ANY channel to LN in Electrum first, then pay **Bob**’s invoice. Electrum requires to fund an entry channel to LN with I suppose **ACINQ** node by default. **ACINQ node** is also a *Trampoline Node* which has full map of LN nodes, so routing is done on **ACINQ** side and the node takes additional fee for that. Technically routing is done by applying *Dijkstra’s algorithm* to all open channels of the network. I’ve opened a channel with **ACINQ**, funded it, now I’m, we can say, “connected to LN” like a mesh-network.

After **Alice** put LN invoice from **Bob** to Electrum and pressed “send”, the following happens - Electrum generates and signs a contract between **Alice** and **ACINQ**, which transfers **1 BTC** to **ACINQ** once invoice is paid by someone(!). **ACINQ node,** which is Trampoline Node, chooses full path **P0** via Dijkstra’s: *Alice → ACINQ → N1 → N2 → … → Bob* and generates the same contract with the next node **N1**, **N1** does the same with **N2** and so on. Since **ACINQ, N1, N2, …** are all interconnected, they negotiate the full chain **P0** as follows: **N1** signs its part of a contract and sends it to **N2**, if **N2** agrees to route, **N2** sends to **N1** back its own part of the contract. As I understand, **Alice** (**ACINQ** in case of delegating this to Trampoline Nodes) decides the full path LN transaction, which means that **N1** will have to negotiate contract with **N2** and not(!) **N123**, even if that path **P2** is possible too.

### QUESTIONS

1. Is everything above correct?
2. Is it possible that **ACINQ node** will deny to open a channel with me? For instance, for AML reasons
3. Could interim nodes do the same when **P0** is negotiated? And do they on practice? e.g. **N1** thinks that **N2** has high AML risk and evades to participate in routing
4. Does sender enforce its routing path **P0** (mandatory) or **N1** can on its own choose a channel to route **Alice**’s payment from **ACINQ,** i.e. **N123** instead of **N2**? How technically these things are done? Couldn’t google anything easy to understand on routing algo but read that **Alice** / **ACINQ** do know(!) the full route, same as in Tor onion layering
5. What exactly the capacity term means in LN, i.e. **ACINQ** has \~424 BTC capacity - is it a sum of all opened channels? How can I evaluate it to something more practically meaningful, e.g. here capacity of \~424 BTC means sending 50 BTC has a chance of 30% to find a route to **Bob** who is connected only to **Binance** node. What is inbound / outbound efficiency?
6. Why some huge nodes are marked as hard to reach despite them being top 5 capacity? i.e. nicehash and Kraken nodes on https://terminal.lightning.engineering/ why is that? Nicehash has the channel with okx, so it should be basically reachable from everywhere
7. If **Alice** entry channel to LN has e.g. 0.1 BTC capacity and **Bob** has a channel of 50 BTC will it be possible to send 50 BTC to **Bob** in practice? Capacity of 50 BTC doesn’t guarantee that there will be a route summing up exactly to 50 BTC. I visualize it as a network of paired pipes, P1→P2 and P2→P1 where P is just an address/wallet. When sending from **Alice** to **Bob** we care only about the amount that all “pipes” of the network can send in **Bob**’s direction but how to calculate that? It’s exponentially hard, there is 20k channels open in LN, it seems impossible to find all route permutations. Maybe from those 50 BTC, 40 will be possible to route if given enough time calculating path, may be only 5
8. Is it better to send huge sum in small chunks or it doesn’t matter? Does Dijsktra’s in Electrum handle chunking automatically? e.g. sending 50 BTC but there is no channels with comparable capacity to send it with 2 intermediaries 25 BTC each, so the amount is split to multiple 0.01 BTC chunks and send with 5000 interim channels which almost guarantees the sum will go through
9. Closure related to questions 5 and 7: what are the real constraints I must consider when using LN regularly for B2B on a technical level? Let’s say I’m running an exchange and there can be a wide range of amounts sent from native Bitcoin network to LN, from 0.1 BTC to over 1000 BTC. I want to know a priori if swap even possible, what’s the minimum entry channel capacity should be, what fees I’ll incur, what time it will take

There is definitely a mess in the questions especially in 5 and 7, I couldn’t formulate them in more accessible manner but hope you guys got the idea of what I’m asking about. Any help is appreciated.

-------------------------

aqua | 2025-09-28 15:09:26 UTC | #2

this is a bump
real bump
whats wrong with my questions?

-------------------------

garlonicon | 2025-09-29 11:30:33 UTC | #3

> whats wrong with my questions?

There is nothing wrong, I guess people are just focused on different things.

> How technically these things are done?

> Does Dijsktra’s in Electrum handle chunking automatically?

You can see it by running regtest, and setting up a network, where you can have more than one client, and then you can confirm it experimentally, which node can see what things, and what decisions can be made by which participant.

> Is it possible that ACINQ node will deny to open a channel with me? For instance, for AML reasons

Of course. And exactly the same attack can be done by miners, because they decide, which transactions are included, and which are not. In case of main chain, you can in theory force the network to accept your transaction, by mining a valid block (as long as hashrate majority is not actively reorging your blocks). In case of LN, you can only choose a different node, because there is no mining (but it can be introduced, if needed).

> Could interim nodes do the same when P0 is negotiated?

Anyone can disturb LN, by simply being offline. I think you are worried about AML reasons, or similar things, but LN can stop working in a much easier way: some node operators could simply disconnect, and wait. To raise a panic, all that is needed, is just convincing enough nodes to go offline.

> And do they on practice?

There is no way to tell, because “being offline” and “rejecting connection for AML reasons” can give you the same end result: forcing you to settle things on-chain. And anyone can tell you “please check your connection”, instead of informing you about any AML rules.

> Capacity of 50 BTC doesn’t guarantee that there will be a route summing up exactly to 50 BTC.

LN payment can always fail. Some people prefer sidechains, instead of LN, because then, you can always send every coin to everyone else. In LN, you simply use the main chain to open and close channels, but contrary to the sidechains, it is hard to send and receive coins “inside LN”, without touching the main chain. It is possible to fix that by using [fake channels](https://petertodd.org/2025/fake-channels-and-rgb-lightning), but if LN is “experimental”, then fake channels are “double experimental”.

> it seems impossible to find all route permutations

The same is true with all transactions’ permutations within a block. So it is based on “best effort”, where you start with any solution, and improve it gradually. Because [Bitcoin mining is NP-hard](https://blog.citp.princeton.edu/2014/10/27/bitcoin-mining-is-np-hard/), and in case of LN, it is not any better than that, and LN nodes also solve NP-hard problems.

> Is it better to send huge sum in small chunks or it doesn’t matter?

It is a philosophical question, because there are also fees. In theory, you could make a lot of transactions, sending one satoshi each, and have a higher chances of sending a payment, which wouldn’t fail. But in practice, LN fees could make that approach impractical. So, the amount you can send, and the way of sending it, is driven by network conditions. Just like on-chain, you can have lower or higher fees, when mempools are more or less congested, the same is true in LN: you can find a single connection, which will handle it atomically, or your payment can fail, if it would be a single payment, so you will be forced to split it into N smaller payments, to have any chances to succeed.

Also, after reaching a certain amount, sending it on-chain may be cheaper, than sending it through LN. Because LN is made for small payments, where on-chain fees are too high to proceed. However, if you can see lower on-chain fees, than LN fees, then just use on-chain. Which is also a reason, why there is more on-chain than LN activity: after going beyond certain amount, it is more profitable, to close all channels, than to keep using them.

> what are the real constraints I must consider when using LN regularly for B2B on a technical level?

Mainly transaction fees, and the risk of nodes being offline. Because LN is not a sidechain. You cannot send anything anywhere. Your actions highly depend on current network state, at the moment when you want to transact (which is also why anchors were introduced, to allow adjusting on-chain fees). If anything goes wrong, then you have to touch on-chain coins, so LN is just layer “one and a half”, instead of being the proper “layer two”.

For example: some casinos officially rejected introducing LN, because they considered it “not worth the effort”. Also, some Bitcoin ATM company decided to not support LN, because they tried and failed. And you can read their statement here: https://shitcoins.club/resources/pdf/stand_LN_en.pdf

So, LN can help you, if you want to make a lot of transactions, between participants, which are almost always online, and where you don’t want to change network topology too much (which means, that the same amounts are constantly flying in both directions between similar participants, so you don’t have liquidity problems, and you can delay going on-chain forever, and do almost everything off-chain). If your business doesn’t meet this definition, then using LN may be more expensive, if you will be forced to go on-chain anyway, and then pay on-chain fees, and off-chain fees, instead of moving fractions of satoshis purely off-chain.

-------------------------

AdamISZ | 2025-10-15 11:23:50 UTC | #4

[quote="garlonicon, post:3, topic:2003"]
So, LN can help you, if you want to make a lot of transactions, between participants, which are almost always online, and where you don’t want to change network topology too much (which means, that the same amounts are constantly flying in both directions between similar participants, so you don’t have liquidity problems, and you can delay going on-chain forever, and do almost everything off-chain). If your business doesn’t meet this definition, then using LN may be more expensive, if you will be forced to go on-chain anyway, and then pay on-chain fees, and off-chain fees, instead of moving fractions of satoshis purely off-chain.

[/quote]

I think your characterization is much too pessimistic here. There are people using Lightning, while keeping self-custody, but not needing a node (see e.g. Phoenix), able to reliably pay bills with it (failure rates basically zero) for years now.

There are lots of merchants accepting bitcoin that succeed in adding Lightning as “just another payment option”. The infrastructure is there now, even if the mindshare isn’t (certainly outside of bitcoiners, it isn’t!).

(Apologies for the relatively irrelevant to the OP commentary)

-------------------------

