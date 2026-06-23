# Addressing the Diminishing Block Subsidy

show1225 | 2026-06-23 15:33:24 UTC | #1

## **Introduction: The Elephant in the Room**

As the block subsidy continues to diminish, we are hoping that transaction fees will eventually replace the subsidy to sustain miner incentives. However, **[historical fee markets have disproved this hypothesis](https://bitinfocharts.com/comparison/bitcoin-median_transaction_fee.html#alltime)**. Normally, we see only a few sats/vB, and periods of high fee revenue are notoriously volatile and short-lived. Projecting this trajectory forward, it is difficult to see a future where transaction fees alone can sustainably secure the network.

While numerous studies have attempted to quantify an appropriate security budget for Bitcoin, there is no definitive consensus or mathematical formula. What is clear is that relying on current fee averages is dangerous post subsidy.

A few sats per vB is only a few thousand dollars per block when block subsidy is gone: Current Bitcoin Cash's security level with much more value on the chain. Even if Bitcoin price goes up significantly over the coming decades, this budget : value proportion will remain (and hoping for an infinite BTC price appreciation is a risky bet).
As we do not know the right amount, it is prudent to secure the network with a (wide) margin of safety. We cannot afford to risk the network becoming functionally insecure or vulnerable to deep reorgs.

In this article, I tentatively set **0.25 BTC per block** (roughly equivalent to 25 sats/vB without block subsidy) as a security budget since I am not intending to discuss the right level of security budget itself.

## **Proposed Solution: Tail Emission + EIP-1559 Style Fee Burn**

To ensure a permanent security budget and incentivize transaction inclusion, I propose introducing a perpetual tail emission combined with an [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559) style base fee burn and priority tipping model.

Tail Emission: 0.25 BTC per block (equates to approximately 25 sats/vB)
Activation Target: Year 2040 at block height 1,680,000, when the block reward is scheduled to halve from 0.390625 BTC to 0.1953125 BTC

Gross inflation of **13,140 BTC per year (annual inflation rate of just 0.06%) will be offset by burnt base fee**.

If Bitcoin's transaction demand is high, the fee burn will exceed the tail emission (net negative). If transaction demand is low, the supply may exceed 21 million BTC, but this is a necessary trade-off to maintain a network where final settlement can actually be trusted.

Benefits:

**1. Reduce Miner Revenue Volatility**
A predictable baseline income reduces the volatility of miner revenue, which in turn lowers the risk premium for mining operations. This will result in a higher hash rate.

**2. Correct the Value Incentive Model**
Currently, the system expects the network's security to be subsidized entirely by active senders. However, [users who use Bitcoin purely as a store of value (passive holders) also benefit immensely from the network's hashpower without paying for it](https://www.youtube.com/watch?v=YDSRdGXY6cQ).
Tail emission distributes the cost of security more fairly. Passive holders pay for security via a nominal inflation "tax," while active transactors effectively offset that inflation by burning fees.

## FAQs:

**Q. Will breaking the 21M cap undermine Bitcoins' narrative as a store of value, driving users back to gold?**
No. Even after thousands of years of human history, today's physical gold above-ground supply inflates by roughly **1.7%** annually (nearly 3,600 tonnes mined per year). Furthermore, as science evolves, gold could be mined (or even manufactured) indefinitely. It costs a lot more to store or buy and sell gold than Bitcoin.
A digitally verifiable asset with a worst-case inflation rate of 0.06% remains structurally superior to gold as a store of value.

**Q. Will this open a Pandora's box of monetary policy?**
There is a concern that miners might lobby for higher emissions once the cap is broken.
But Bitcoin is not governed like fiat. It does not have any ties to any national budget. Too much emission will not be widely supported even with a strong advocacy from certain actors.

Conversely, one day if we find our tail emission is not enough or too much, we could adjust it. We do not have such a lifeguard if we sorely rely on transaction fees. Bitcoin's transaction fee market is not meant for keeping the Bitcoin network secure. Senders just want to pay as little fee as possible. The fee market could happen to make the network secure nicely, but we are not sure about it.

It is also worth noting that Ethereum has had no issue with EIP-1559 although their governance model is different from Bitcoin's.

**Q. Why don't we make Bitcoin proof of stake?**
Staking will cost more than mining. Nobody stakes for APR 0.06% (0.25 BTC per block). Staking will add unnecessary security assumptions to Bitcoin.

**Q. How do we know the right amount of security budget?**
We could think of the current halving block subsidies era as a journey (chicken game) to find an appropriate minimum amount of budget to secure the Bitcoin network. A few halvings ago, we had no idea that only 3.125 BTC per block would keep the network state healthy contrary to academic warnings in the 2010s.

Current block subsidy of 3.125 BTC demonstrates that the network remains highly secure at this level. We could safely say that we can reduce the security budget to 0.X BTC per block zone (tens of sats/vB post subsidy). However, 0.0X BTC territory (a few sats/vB post subsidy) could be dangerous.

If we find in the 2030s that even 0.0X BTC / block could secure the network, we can postpone the tail emission. However, we must plan to exit the chicken game (How far can we reduce block rewards while keeping the network sound?).

**Q. Why not simply set a floor sats/vB?**
The current low-fee environment is the result of market mechanisms. I admit the price elasticity of block space is low, but setting an artificial floor may just result in empty blocks. In addition, floor fees could be confiscatory for tiny Bitcoin holders who can wait many blocks for lower fees but cannot afford to pay higher transaction fees.

**Q. How much will Bitcoin transactions cost if the fee market naturally reaches 25 sats/vB (1 BTC = 100,000)?**
Today's 208vB transaction will cost $5.6
PQ stateful hash based signature transaction (358vB) will cost $8.95
PQ stateless hash based signature transaction (1,446vB) will cost $36.15

-------------------------

cmp_ancp | 2026-06-23 16:01:49 UTC | #2

I fear that any solution will face a strong resistance by the community, specially if the 21M cap is broken.

Something I thought, reading your position on "hodlers paying a tax by using the hashrate", is to tax the oldest UTXOs on the set if the block fees don't beat a threshold. In that way, we can maintain the total supply, secure miners revenue and bring lost coins back to circulation. My only doubt would be on how to partially tax a UTXO and maintain its possession to the original owner. Maybe the UTXO could be consumed by a TX and a new UTXO is generated with the same spend script, however, that would break possible covenants constructions. A tax on oldest UTXOs would promote UTXO refreshes, wich also helps the network with fees.

Anyway, I think no idea would have the public appeal on the short run. Maybe the community will change its mind after a few halvings, when we start to face the real impacts on the end of the subsidy. These impacts could be greater or lesser, depending on future adoption. In the best scenario, the increase in adoption would be sufficient for network security.

-------------------------

MrHash | 2026-06-23 16:48:47 UTC | #3

Any inflation is arbitrary and mirrors the fiat system. If it is 2%, it can be 1%. If it is 1%, it can be 0.5%. Hence 0% is the only logically defensible position.

-------------------------

