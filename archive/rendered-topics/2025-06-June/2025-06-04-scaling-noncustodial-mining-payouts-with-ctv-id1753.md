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

