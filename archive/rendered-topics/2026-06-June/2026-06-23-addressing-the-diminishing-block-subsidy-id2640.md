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

show1225 | 2026-06-23 22:43:27 UTC | #4

Dear cmp_ancp,

The only justifiable situation could be that we decide to deprecate ECC signature in the future and make use of P2PK coins which cannot be recovered with any rescue protocols (ZKP of BIP32 etc.).
Sacrificing the eldest in the village to protect the virtue of 21M cap is far more unfair and grotesque than evenly tax everyone by inflation (possibly net zero or negative inflation).

Speaking of PQ, maybe the transition to PQ signature will make the block space scarcer and we only need to smoothen volatile fee market by 2030s. In that sense, we should stay focused on the PQ transition for now and see what Bitcoin will look like after Q-day. Although any scaling solution or invention of lightweight PQ signature might nullify the elevated block space demand.


I do agree that we have a long way to go to reach consensus on any solution. That is why we have to start thinking about it now.

-------------------------

show1225 | 2026-06-23 22:52:06 UTC | #5

Dear MrHash, 

Flaw of the fiat system is in its governance rather than inflation itself.

Also, if there is a fair and nice solution without breaking the 21M cap, I am happy to discard my idea and support it. Is there any?

-------------------------

sipa | 2026-06-23 23:54:53 UTC | #6

I consider this a complete non-starter.

Not because tail inflation is inherently a bad idea. It may well be better, but we don't know. The right place to try this is in another currency, where it can compete fairly.

Bitcoin started as an experiment for a novel type of money. It was designed with a very clearly stated and opinionated choice of a finite monetary supply. That is what every user, as far back as 2009, knew and signed up for. It made several such choices, and they made Bitcoin what it is today. I suspect that if Bitcoin had launched with permanent inflation from the start, maybe it wouldn't have taken off, and maybe cryptocurrency wouldn't exist at all today.

What distinguishes Bitcoin from every other attempt at creating a currency, digital or otherwise, is not the optimality of those decisions, but confidence that its properties are reliable and beyond influence of any actors, malicious or not. This undermines that completely.

-------------------------

ajtowns | 2026-06-24 02:59:49 UTC | #7

[quote="show1225, post:1, topic:2640"]
As the block subsidy continues to diminish
[/quote]

While the subsidy is certainly decreasing in BTC terms, in USD terms it still seems to be increasing:

```plotly
data:
        - x: ['2010-07', '2010-08', '2010-09', '2010-10', '2010-11', '2010-12', '2011-01', '2011-02', '2011-03', '2011-04', '2011-05', '2011-06', '2011-07', '2011-08', '2011-09', '2011-10', '2011-11', '2011-12', '2012-01', '2012-02', '2012-03', '2012-04', '2012-05', '2012-06', '2012-07', '2012-08', '2012-09', '2012-10', '2012-11', '2012-12', '2013-01', '2013-02', '2013-03', '2013-04', '2013-05', '2013-06', '2013-07', '2013-08', '2013-09', '2013-10', '2013-11', '2013-12', '2014-01', '2014-02', '2014-03', '2014-04', '2014-05', '2014-06', '2014-07', '2014-08', '2014-09', '2014-10', '2014-11', '2014-12', '2015-01', '2015-02', '2015-03', '2015-04', '2015-05', '2015-06', '2015-07', '2015-08', '2015-09', '2015-10', '2015-11', '2015-12', '2016-01', '2016-02', '2016-03', '2016-04', '2016-05', '2016-06', '2016-07', '2016-08', '2016-09', '2016-10', '2016-11', '2016-12', '2017-01', '2017-02', '2017-03', '2017-04', '2017-05', '2017-06', '2017-07', '2017-08', '2017-09', '2017-10', '2017-11', '2017-12', '2018-01', '2018-02', '2018-03', '2018-04', '2018-05', '2018-06', '2018-07', '2018-08', '2018-09', '2018-10', '2018-11', '2018-12', '2019-01', '2019-02', '2019-03', '2019-04', '2019-05', '2019-06', '2019-07', '2019-08', '2019-09', '2019-10', '2019-11', '2019-12', '2020-01', '2020-02', '2020-03', '2020-04', '2020-05', '2020-06', '2020-07', '2020-08', '2020-09', '2020-10', '2020-11', '2020-12', '2021-01', '2021-02', '2021-03', '2021-04', '2021-05', '2021-06', '2021-07', '2021-08', '2021-09', '2021-10', '2021-11', '2021-12', '2022-01', '2022-02', '2022-03', '2022-04', '2022-05', '2022-06', '2022-07', '2022-08', '2022-09', '2022-10', '2022-11', '2022-12', '2023-01', '2023-02', '2023-03', '2023-04', '2023-05', '2023-06', '2023-07', '2023-08', '2023-09', '2023-10', '2023-11', '2023-12', '2024-01', '2024-02', '2024-03', '2024-04', '2024-05', '2024-06', '2024-07', '2024-08', '2024-09', '2024-10', '2024-11', '2024-12', '2025-01', '2025-02', '2025-03', '2025-04', '2025-05', '2025-06', '2025-07', '2025-08', '2025-09', '2025-10', '2025-11', '2025-12', '2026-01', '2026-02', '2026-03', '2026-04']
          y: ['4.50', '3.00', '3.00', '3.00', '10.00', '11.50', '15.00', '35.00', '46.00', '38.50', '151.50', '478.50', '770.00', '654.50', '410.50', '251.50', '157.50', '153.00', '263.50', '304.00', '246.00', '241.50', '250.00', '263.50', '331.50', '477.50', '498.50', '620.00', '528.50', '314.00', '332.50', '512.50', '862.50', '2600.00', '2909.50', '3232.50', '2115.25', '2410.50', '3206.50', '3137.25', '4962.75', '23673.00', '19261.00', '21325.50', '14093.50', '11968.00', '11406.75', '15725.50', '15889.75', '14877.00', '11853.25', '9533.25', '8134.75', '9466.00', '7848.00', '5660.00', '6468.50', '6163.75', '5803.00', '5582.75', '6441.50', '7001.00', '5683.75', '5939.25', '8127.00', '9068.25', '10861.50', '9281.25', '10841.00', '10425.25', '11326.00', '13410.50', '16913.00', '7592.12', '7146.13', '7674.12', '9115.88', '9415.62', '12471.12', '12341.88', '15375.25', '13618.88', '17526.00', '30652.25', '30752.50', '34194.88', '61884.00', '54933.00', '84377.12', '135744.50', '167655.50', '113157.25', '136344.88', '85209.25', '113346.37', '93978.00', '79348.25', '95046.88', '87654.23', '82821.24', '78988.26', '49738.44', '46444.56', '42997.57', '47645.15', '51689.81', '66224.28', '107355.99', '130683.55', '129815.01', '122258.35', '104171.11', '114142.63', '94277.95', '89749.47', '117361.01', '107994.89', '80580.27', '109595.91', '58922.94', '57177.78', '71083.23', '72989.83', '67134.87', '86820.11', '121139.75', '181947.01', '206796.06', '281830.04', '367029.15', '358141.54', '230809.23', '217847.78', '261684.70', '293246.95', '273239.86', '383589.24', '356082.39', '288801.32', '240624.23', '277216.48', '289260.28', '240431.84', '186244.25', '120433.54', '145713.75', '125794.63', '120700.60', '128032.96', '106044.58', '103906.75', '148273.56', '147790.94', '177568.97', '175572.30', '167624.83', '191187.99', '185473.33', '161254.53', '174898.44', '221482.84', '241804.69', '276045.83', '269223.58', '390253.96', '435638.43', '182043.79', '211584.18', '196412.44', '204242.19', '179543.75', '190565.62', '217468.75', '304265.62', '296115.62', '314687.50', '269943.75', '266240.62', '301459.38', '330062.50', '331000.00', '354687.50', '341906.25', '370843.75', '344656.25', '270228.12', '277737.50', '240665.62', '205600.00', '212731.25']
          mode: lines
layout:
  title: Block subsidy overall
  xaxis:
    title: "date"
  yaxis:
    title: "USD"
    type: log
    autorange: true
```

```plotly
data:
        - x: ['2019-01', '2019-02', '2019-03', '2019-04', '2019-05', '2019-06', '2019-07', '2019-08', '2019-09', '2019-10', '2019-11', '2019-12', '2020-01', '2020-02', '2020-03', '2020-04', '2020-05', '2020-06', '2020-07', '2020-08', '2020-09', '2020-10', '2020-11', '2020-12', '2021-01', '2021-02', '2021-03', '2021-04', '2021-05', '2021-06', '2021-07', '2021-08', '2021-09', '2021-10', '2021-11', '2021-12', '2022-01', '2022-02', '2022-03', '2022-04', '2022-05', '2022-06', '2022-07', '2022-08', '2022-09', '2022-10', '2022-11', '2022-12', '2023-01', '2023-02', '2023-03', '2023-04', '2023-05', '2023-06', '2023-07', '2023-08', '2023-09', '2023-10', '2023-11', '2023-12', '2024-01', '2024-02', '2024-03', '2024-04', '2024-05', '2024-06', '2024-07', '2024-08', '2024-09', '2024-10', '2024-11', '2024-12', '2025-01', '2025-02', '2025-03', '2025-04', '2025-05', '2025-06', '2025-07', '2025-08', '2025-09', '2025-10', '2025-11', '2025-12', '2026-01', '2026-02', '2026-03', '2026-04']
          y: ['46444.56', '42997.57', '47645.15', '51689.81', '66224.28', '107355.99', '130683.55', '129815.01', '122258.35', '104171.11', '114142.63', '94277.95', '89749.47', '117361.01', '107994.89', '80580.27', '109595.91', '58922.94', '57177.78', '71083.23', '72989.83', '67134.87', '86820.11', '121139.75', '181947.01', '206796.06', '281830.04', '367029.15', '358141.54', '230809.23', '217847.78', '261684.70', '293246.95', '273239.86', '383589.24', '356082.39', '288801.32', '240624.23', '277216.48', '289260.28', '240431.84', '186244.25', '120433.54', '145713.75', '125794.63', '120700.60', '128032.96', '106044.58', '103906.75', '148273.56', '147790.94', '177568.97', '175572.30', '167624.83', '191187.99', '185473.33', '161254.53', '174898.44', '221482.84', '241804.69', '276045.83', '269223.58', '390253.96', '435638.43', '182043.79', '211584.18', '196412.44', '204242.19', '179543.75', '190565.62', '217468.75', '304265.62', '296115.62', '314687.50', '269943.75', '266240.62', '301459.38', '330062.50', '331000.00', '354687.50', '341906.25', '370843.75', '344656.25', '270228.12', '277737.50', '240665.62', '205600.00', '212731.25']
          mode: lines
layout:
  title: Block subsidy since 2019
  xaxis:
    title: "date"
  yaxis:
    title: "USD"
    type: log
    autorange: true
```

I would expect that when that changes, miners will make more effort to optimise fee revenue.

-------------------------

show1225 | 2026-06-24 06:40:37 UTC | #8

Dear Pieter,

I would be very interested to hear your napkin-sketch ideas on how to address this problem.


While I fully support the community's effort to naturally drive up block space demand, given the fact that a low-fee environment has historically been the norm, we should have a plan.

-------------------------

show1225 | 2026-06-24 07:06:09 UTC | #9

Dear Anthony,

Miners are making bloody efforts to optimize their profit. If they could squeeze tens of thousands of dollars by optimizing fee revenue, they should have done so already (I am not a mining insider though).

-------------------------

cmp_ancp | 2026-06-24 11:15:27 UTC | #10

Raises in the BTC price may diminish the halving impact. If the BTC price doubles, then the halving is totally mitigated.

However, that doesn't count for the last halving, when subsidy disappear as a whole, independent on BTC price.

BTC price also doesn't help with raising the revenue. We should expect that, with BTC valuation, fees should fall, if the willingness of the users to spend value in tx fees stay the same. I mean, the raise in BTC price may value the revenue from yesterday if it was stored in BTC though.

-------------------------

sipa | 2026-06-24 13:38:08 UTC | #11

[quote="show1225, post:8, topic:2640"]
I would be very interested to hear your napkin-sketch ideas on how to address this problem.
[/quote]

I already gave you my answer: experiment with different inflation schedules (including tail emission) on in another currency, and compete with Bitcoin. It is very possible that in the long term it wins in the market.

In fact, that is also what your proposal practically amounts to, because as an inevitably controversial hardfork, adopters of it will find themselves operating a different currency anyway (just one whose initial distribution is bootstrapped with a snapshot of Bitcoin's UTXO set at fork time).

-------------------------

show1225 | 2026-06-25 00:40:17 UTC | #12

Dear Peter,

I appreciate your replies.
Yes. It will be a controversial hardfork and Bitcoin will be split into two if this kind of proposal gains a certain level of support.

I wanted know if you have any soft-forkable idea to address the security budget problem. The only idea I could come up with is introducing a floor fee.

-------------------------

statoshi | 2026-06-25 18:21:50 UTC | #13

I've noodled a bit on this over the years and one of the fundamental reasons I think most schemes to address decreasing block subsidies are non-starters is because there is no way for Bitcoin as a system to know its own value. Some say that this is a "fiat approach" but fiat is merely one way to measure value because it's a unit of account. Point being, it's very hard to algorithmically change the revenue afforded to miners if you are targeting a given level of thermodynamic security that's measured in any standard unit of account.

There is an alternative approach that I have thought about a bit and [gave a presentation](https://www.youtube.com/watch?v=cPPKok5luk4) about last year: introduce a dynamic block size that adjusts based upon the level of demand (fees paid) for block space. I lean more toward something like this that seeks to "kick start" the auction market for block space during periods when demand drops below available supply, causing the market rate for block space to crater to the fee floor. Though the devil would be in the details, of course.

This would be easily soft forkable if you introduced a dynamic limit that only adjusted the maximum block size within the current allow range of block sizes. If you always wanted it to be able to increase available block space beyond the current ceiling, it would require a much more convoluted soft fork or a simpler hard fork.

-------------------------

Choirzooh | 2026-06-26 14:25:50 UTC | #14

Great topic and imo, I think the options are getting mixed together. Separating them might help the "what else is there?" question. There are really three categories here, not two:

1. Change the issuance --- tail emission, taxing old UTXOs. sipa's point settles
   this one: a contested hardfork just creates a different coin bootstrapped from
   Bitcoin's UTXO set, so "go do it in another currency and compete" is what
   these really amount to.

2. Tweak the on-protocol fee market --- statoshi's dynamic block size, a floor
   fee. These keep the fee auction going, but as statoshi says, they can't aim
   at a specific security level --- and they add no new money, they just move the
   fee side around.

3. Pay miners from outside Bitcoin --- a reward in some other asset that doesn't
   touch consensus and leaves the 21M cap alone. The thread keeps pointing at
   this without naming it (it's exactly what "go compete in another currency"
   describes), but never treats it as its own option. Merged mining is the one
   accepted example --- small, but proof the category isn't empty.

Here's why #3 is worth pulling out, and it follows from statoshi's own point: if Bitcoin can't measure its own value from the inside, then the only place that price can form is outside it. So an outside, market-set reward is the only kind that even gets at the problem --- but it pays for that by giving up consensus enforcement.

What makes #3 worth taking seriously is that it's the only one of the three that needs no hardfork and no one's permission --- it doesn't change Bitcoin's rules or risk a chain split, and unlike fees it doesn't depend on a particular future showing up. Merged mining already shows the category works; it's small only because the chains doing it are small, not because the idea is capped --- the contribution scales with whatever market decides to fund it. The honest open questions are really about the ceiling: how much durable demand there is for funding security as its own thing (rather than it just flowing back into BTC), and whether a reward miners could in principle ignore actually moves their
behaviour in practice. Thoughts?

-------------------------

CubicEarth | 2026-06-26 20:35:33 UTC | #15

[quote="Choirzooh, post:14, topic:2640"]
Change the issuance --- tail emission, taxing old UTXOs. sipa's point settles this one: a contested hardfork just creates a different coin bootstrapped from Bitcoin's UTXO set, so "go do it in another currency and compete" is what these really amount to.

[/quote]

Well let's be clear here: A hard fork is not needed to do these things, or at least their functional economic equivalents, and therefore can't be relied on as the objection.

For instance, a soft fork could freeze 50% of all balances permanently, and if that was repeated every 4 years, it would effectively force deflation while leaving the block rewards a constant size relative to the usable money supply. Hence tail emissions without a hard fork, at least for the next 100 years.

A similarly aggressive soft fork could implement a transaction tax as a percentage of moved value. Now perhaps that would just push activity to layer 2s, and it might be a bad idea for other reason. But the point is it would only require a soft fork. 


It is also interesting to recall that the 1 MB block size limit was itself a soft fork down from an effective limit of 32 MB. Later on, many people strongly advocated for keeping the size small as a way to force scarcity and thereby generate a robust fee market to pay for Bitcoin's security. Anyone who advanced that reasoning, IMO, has revealed that they are in fact willing to adjust Bitcoin's fundamental economic parameters to try to pay for security. 

While I personally disagreed that very small blocks were the best way to maximize fee revenue (I think we choked off growth far too early and lost mindshare and economic activity to other coins), I think it is reasonable to use a political process to change the economic parameters to ensure Bitcoin's survival.

The beauty of Satoshi's system was never because it was perfect from the start and and would survive forever without change, but rather because of the way it aligns incentives, cryptography, auditability and permissionlessness such that changes can't be forced from a minority onto a majority, and that everyone can verify and participate equally without any credentialing.

But it is a market-majority system. Or perhaps market-supermajority. There is no reason we should let Bitcoin die if we discover something once worshipped is a flaw. The market can operate within and on Bitcoin too, not just by jumping to some other coin.

-------------------------

show1225 | 2026-06-29 05:51:49 UTC | #16

Dear Jameson,

My primary concern is misalignment between who pays for the network security and who benefit it in the post block subsidy era (considering subsidies as effective dilution / inflation).

However, deploying less controversial idea like dynamic block size first and see how fee market changes seems more realistic.

I do hope and would like to work to make this problem get more attention once quantum updates get settled. Quantum transition might change the whole land scape of the fee market anyway.

-------------------------

