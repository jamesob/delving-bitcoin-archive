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

