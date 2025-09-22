# LN routing and caveats

aqua | 2025-09-22 06:30:52 UTC | #1

Need someone to tell if anything wrong with my understanding and fill the gaps

I want to send 1 BTC from **Alice** to **Bob** using Lightning, **Alice** does not have a direct channel with **Bob**. In practice me (**Alice**) need to open ANY channel to LN in Electrum first, then pay **Bob**’s invoice. Electrum requires to fund an entry channel to LN with I suppose **ACINQ** node by default. **ACINQ node** is also a *Trampoline Node* which has full map of LN nodes, so routing is done on **ACINQ** side and the node takes additional fee for that. Technically routing is done by applying *Dijkstra’s algorithm* to all open channels of the network. I’ve opened a channel with **ACINQ**, funded it, now I’m, we can say, “connected to LN” like a mesh-network. 

After **Alice** put LN invoice from **Bob** to Electrum and pressed “send”, the following happens - Electrum generates and signs a contract between **Alice** and **ACINQ**, which transfers **1 BTC** to **ACINQ** once invoice is paid by someone(!). **ACINQ node,** which is Trampoline Node, chooses full path **P0** via Dijkstra’s: *Alice → ACINQ → N1 → N2 → … → Bob* and generates the same contract with the next node **N1**, **N1** does the same with **N2** and so on. Since **ACINQ, N1, N2, …** are all interconnected, they negotiate the full chain **P0** as follows: **N1** signs its part of a contract and sends it to **N2**, if **N2** agrees to route, **N2** sends to **N1** back its own part of the contract. As I understand, Alice (**ACINQ** in case of delegating this to Trampoline Nodes) decides the full path LN transaction, which means that **N1** will have to negotiate contract with **N2** and not(!) **N123**, even if that path **P2** is possible too.

### QUESTIONS

1. Is everything above correct?
2. Is it possible that **ACINQ node** will deny to open a channel with me? For instance, for AML reasons
3. Could interim nodes do the same when **P0** is negotiated? And do they on practice? e.g. **N1** thinks that **N2** has high AML risk and evades to participate in routing
4. Does sender enforce its routing path **P0** (mandatory) or **N1** can on its own choose a channel to route **Alice**’s payment from **ACINQ,** i.e. **N123** instead of **N2**? How technically these things are done? Couldn’t google anything easy to understand on routing algo but read that **Alice** / **ACINQ** do know(!) the full route, same as in Tor onion layering
5. What exactly the capacity term means in LN, i.e. **ACINQ** has \~424 BTC capacity - is it a sum of all opened channels? How can I evaluate it to something more practically meaningful, e.g. here capacity of \~424 BTC means sending 50 BTC has a chance of 30% to find a route to **Bob** who is connected only to **Binance** node. What is inbound / outbound efficiency?
6. Why some huge nodes are marked as hard to reach despite them being top 5 capacity? i.e. nicehash and Kraken nodes on https://terminal.lightning.engineering/ why is that? Nicehash has the channel with okx, so it should be basically reachable from everywhere
7. If **Alice** entry channel to LN has e.g. 0.1 BTC capacity and **Bob** has a channel of 50 BTC will it be possible to send 50 BTC to **Bob** in practice? Capacity of 50 BTC doesn’t guarantee that there will be a route summing up exactly to 50 BTC. I visualize it as a network of paired pipes, P1→P2 and P2→P1 where P is just an address/wallet. When sending from **Alice** to **Bob** we care only about the amount that all “pipes” of the network can send in **Bob**’s direction but how to calculate that? It’s exponentially hard, there is 20k channels open in LN, it seems impossible to find all route permutations. Maybe from those 50 BTC, 40 will be possible to route if given enough time calculating path, may be only 5
8. Is it better to send huge sum in small chunks or it doesn’t matter? Do Dijsktra’s in Electrum handle chunking automatically? e.g. sending 50 BTC but there is no channels with comparable capacity to send it with 2 intermediaries 25 BTC each, so the amount is split to multiple 0.01 BTC chunks and send with 5000 interim channels which almost guarantees the sum will go through
9. Closure related to questions 5 and 7: what are the real constraints I must consider when using LN regularly for B2B on a technical level? Let’s say I’m running an exchange and there can be a wide range of amounts sent from native Bitcoin network to LN, from 0.1 BTC to over 1000 BTC. I want to know a priori if swap even possible, what is the minimum entry channel capacity should be, what fees I’ll incur, what time it will take, which framework to use.

There is definitely a mess in the questions especially in 5 and 7, I couldn’t formulate them in more accessible manner but hope you guys got the idea of what I’m asking about. Any help is appreciated.

-------------------------

