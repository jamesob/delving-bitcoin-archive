# Fee Selection for CTV-Based Mining Pool Fanouts

djh58 | 2026-07-08 18:35:58 UTC | #1

Inspired by [Scaling Noncustodial Mining Payouts with CTV](https://delvingbitcoin.org/t/scaling-noncustodial-mining-payouts-with-ctv/1753), I've been working on a qbit pool implementation. qbit is a UTXO chain with post-quantum signatures and OP_CTV, so it has been a useful testbed for CTV-based mining payouts.

The pool uses [TIDES](https://ocean.xyz/docs/tides)-style share accounting and routes recipients through direct coinbase outputs, carry-forward balances, or CTV fanouts. An initial implementation is live at https://prismpool.io, and the full source is at https://github.com/Qbit-Org/qbit-mining-bootstrap .

Direct coinbase outputs are scarce because the coinbase has to stay small enough for stock miner firmware to handle. So most recipients settle through precommitted CTV fanouts. The fanout's fee is fixed by the covenant hash at template-build time, \~17 hours (1,000-block coinbase maturity) before it can be broadcast. How to choose that fee is the decision I'd like feedback on.

We ruled out the zero-fee-parent + P2A/CPFP route and instead build the fee into the fanout, with every recipient paying a proportional share of it, for two reasons:

1) **No funded relayer.** Anyone holding the published artifact bytes can broadcast a fee-paying fanout at maturity, with no wallet, no capital, and no package relay. A zero-fee parent instead needs a funded broadcaster around for as long as any fanout is unclaimed.

2) **Dust defense.** Every payout must cover its fee share and still clear the economic floor, or it's pruned back to carry-forward. Sybil swarms of tiny mining identities therefore can't force uneconomic outputs onto the pool or other miners. The floor is unusually high on qbit because post-quantum P2MR inputs cost \~3,680 bytes to spend, so keeping uneconomic outputs off the chain matters even more here than it would on Bitcoin.

We also learned live that the obvious hybrid of built-in fee plus a 0-sat P2A anchor for top-ups is nonstandard: qbit inherits Bitcoin Core's ephemeral-dust policy (\`dust, tx with dust output must be 0-fee\`), so the same limitation applies to any Bitcoin CTV pool.

That makes the committed fee the only fee. Today we use the node's fee rate estimate × 1.2. If it proves too low, recipients can CPFP their own outputs.

**The question:** how would you price a fee committed \~17 hours before broadcast? Spot estimate × fixed premium, a multi-day fee average to smooth out spikes, or just a larger fixed premium?

Curious how others would weigh this, thanks in advance!

-------------------------

vnprc | 2026-07-09 02:01:53 UTC | #2

[quote="djh58, post:1, topic:2698"]
1. **No funded relayer.** Anyone holding the published artifact bytes can broadcast a fee-paying fanout at maturity, with no wallet, no capital, and no package relay. A zero-fee parent instead needs a funded broadcaster around for as long as any fanout is unclaimed.

[/quote]

It sounds like you are optimizing for the initial onboarding transaction over the steady state of payouts. This feels like a strange choice to me. Do you expect most miners to churn out after their first payout? I would expect the opposite and optimize for the 99.9% case instead of the 0.1% case. The downside of having users pay the fee to mine the fanout transaction doesn't seem that bad either. A miner who can't pay the fee can simply wait for someone else to fee bump the fanout transaction.

My approach in the original design was to attach a 1sat/vb fee, funded from the coinbase output, and also an anchor output for users to fee bump if necessary. This way they can simply wait for the fanout to get mined during a low fee period or if someone is motivated enough to fee bump the whole fanout transaction they can pay to accelerate everyone's payouts. So low time preference miners always get paid eventually, high time preference miners are forced to subsidize the fee bump of everyone else in that fanout. Seems fair to me.

I also ran into the standardness problem when building the fanout transaction. I limited it to 319 outputs due to one rule and a \~300 sat anchor output due to another rule. (This was before zero-fee anchors were implemented.) This is very frustrating for a mining pool because block producers don't need to follow standardness rules.

Your pool can include non-standard payout transactions in the blocks it mines with zero problems. You only need to produce standard fanout transactions if you want every block producer to have a chance to mine those transactions.

Standardness rules are not really bitcoin rules. They are an attempt to control (or censor 🤭) transactions that flow through the node network before they are mined. But, as Peter Todd has shown with Libre Relay, these rules are easy to loosen with a preferential peering subnet of nodes and as [slipstream](https://slipstream.mara.com/) has shown, they are not necessary at all to get your transaction mined.

-------------------------

vnprc | 2026-07-09 12:12:32 UTC | #3

Another thought occurred to me overnight. If you host the non-standard fanout transaction somewhere your miners can access them, probably via an API, the miners can simply attach their own fee to the 0-sat anchor and broadcast it to the mempool. Once the fee is attached the pair is now a standard 1p1c package that can be relayed across the node network.

-------------------------

vnprc | 2026-07-09 15:34:02 UTC | #4

> Once the fee is attached the pair is now a standard 1p1c package that can be relayed across the node network.

Shoutout to the core devs and maintainer who put in years of work to make this possible. Gone but not forgotten. :pouring_liquid:

-------------------------

