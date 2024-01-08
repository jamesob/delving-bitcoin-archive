# [BUG]: spammers get Bitcoin blockspace at discounted price. Let's fix it

GregTonoski | 2023-12-27 20:54:30 UTC | #1

Blockspace price for data of a simple transaction is higher than the one for data of other ("complex") transactions, e.g.
 
3=616 weight / 205 bytes [aabbcce67f2aa71932f789cac5468d39e3d2224d8bebb7ca2c3bf8c41d567cdd](https://mempool.space/tx/aabbcce67f2aa71932f789cac5468d39e3d2224d8bebb7ca2c3bf8c41d567cdd)

vs

1.49=1140 weight / 767 bytes [1c35521798dde4d1621e9aa5a3bacac03100fca40b6fb99be546ec50c1bcbd4a](https://mempool.space/tx/1c35521798dde4d1621e9aa5a3bacac03100fca40b6fb99be546ec50c1bcbd4a).

As a result, there are incentives distorted and critical inefficiencies/vulnerabilities (e.g. misallocation of block space, blockspace value destruction, disincentivised simple transaction, centralization around complex transactions composers).

**Blockspace price shouldn't be higher for a simple transaction (price discrimination against simple txs)**
Price of blockspace should be the same for any data (1 byte = 1 byte, irrespectively of location inside or outside of witness), e.g. 205/205 bytes in the example above. 

Perhaps, the solution (the same price, "weight" of any byte of data in a transaction) could be introduced as part of the next version of Segwit transactions.

Do you agree?

-------------------------

murch | 2023-12-28 13:55:39 UTC | #2

Your analysis disregards that various parts of transactions do not have the same resource footprint in our system. I would suggest that you further explore the reasoning that led to the current state of affairs if you wish to convincingly motivate an intervention.

-------------------------

moonsettler | 2023-12-30 17:47:32 UTC | #3

if you want to do it without screwing over the normal users of bitcoin, and only make disproportionate use of witness space less competitive:

`WUtx = 4·size(tx_data|segwit_data) - 3·min(3·size(tx_data), size(segwit_data))`

however most of the spam is brc-20 and that already would not be significantly affected. in the future alternative assets likely will be either using runes or rgb. they can get a lot more efficient with block space. you won't likely achieve anything with this approach.

-------------------------

GregTonoski | 2024-01-06 19:49:07 UTC | #4

I haven't found evidence of validity of the "reasoning that led to the current state of affairs". To the contrary:
1. the ongoing (and a year long already) spam mania showed that people were led "to use the witness space to store general purpose data on the block chain, or adversarial miners may fill excess space with random junk to defeat fast relay schemes that rely on similar mempools" (SegWit Resources, "[Why a discount factor of 4? Why not 2 or 8?](https://medium.com/segwit-co/why-a-discount-factor-of-4-why-not-2-or-8-bbcebe91721e)").
2. Allegedly, another reason was to increase capacity (to 4MB "in a pathological case when you have a single transaction"  - [Pieter Wuille](https://www.youtube.com/live/NOYNZB5BCHM?feature=shared&t=2105)) and so increase the number of transactions which isn't achieved. The transactions are outcrowded by spam. 
3. "[-50% discount in transaction fee due to the use of the cheaper space for witness data](https://youtu.be/T1fqOEhFP40?feature=shared&t=2577)" is not achieved. As a matter of fact, fees are at all time high.
4. UTXO set grows despite fewer simple (not spam) transactions,
5. Consolidation incentive is not effective as many small UTXOs are stuck due to high fees (so-called "[unspedendable dust](https://bitcoin.stackexchange.com/a/108336/135945)" caused by SegWit mispricing).

Does the analysis address your concern, perhaps?

-------------------------

GregTonoski | 2024-01-06 21:15:19 UTC | #6

[quote="moonsettler, post:3, topic:327"]
you won’t likely achieve anything with this approach.
[/quote]

Can I ask you to clarify your stance on whether “Blockspace price shouldn’t be higher for a simple transaction (price discrimination against simple txs)”, please?

-------------------------

moonsettler | 2024-01-07 18:13:04 UTC | #7

i was talking about messing with the witness discount not achieving anything when most of the truly fee competitive spam is brc-20 like s***coin protocols. they already have a fairly dense on-chain footprint and they certainly can get more compressed. which means they can get even more competitive in bidding for block space in their own hype cycle.

high fees will make users of bitcoin generally miserable, but also hardly can be avoided if such a bullish external demand for the same blockspace is present. or in general long term at some significant level of adoption. that's why i said it likely won't achieve anything positive for the native economic users of bitcoin. they would just be worse off then they are rn.

-------------------------

GregTonoski | 2024-01-07 20:24:49 UTC | #8

[quote="moonsettler, post:7, topic:327"]
that’s why i said it likely won’t achieve anything positive for the native economic users of bitcoin. they would just be worse off then they are rn.
[/quote]

I think we may not be on the same page. Can I ask you to verify if there isn't misunderstanding, please? I proposed that the blockspace price was equalized for any type of transaction. As a result, the simple type of transactions ("native economic") wouldn't be put at disadvantage. Doesn't it sound like a positive change?

There is the breakdown and cost comparison of the two actual transactions: [https://gregtonoski.github.io/bitcoin/segwit-mispricing/comparison-of-costs.html](https://gregtonoski.github.io/bitcoin/segwit-mispricing/comparison-of-costs.html).

-------------------------

DoctorBuzz1 | 2024-01-07 20:55:01 UTC | #9

I believe that changing IsDust (or GetDustThreshold, rather) to reflect the intent of the code comments 

(if Tx < TxFee) {IsDust = True;}

... would solve a lot of the BRC-20 style b.s.

Murch mentioned to me elsewhere that Child - Parent Txs are perhaps the biggest issue with this idea (outside of Lightning, but LN should easily be able to adapt).


![1000000600|446x500](upload://1fR1n6dTIKkaIX0r2EmD6YOIcSA.png)

-------------------------

moonsettler | 2024-01-07 22:15:10 UTC | #10

[quote="GregTonoski, post:8, topic:327"]
As a result, the simple type of transactions (“native economic”) wouldn’t be put at disadvantage.
[/quote]

imo it's a huge mistake to think native economic transactions can't involve complex script. a multisig typically would involve a sizeable witness. for example a simple typical 2-of-3 with change could be 125 bytes main block data with 254 bytes witness.

but even if you only make native asset transactions marginally more expensive, while forcing alternative assets to be more economic than they are (say via taptweak and client side validation), the users would still need to bid a higher fee rate for the same relative demand.

it really doesn't sound positive to me.

-------------------------

GregTonoski | 2024-01-08 08:20:24 UTC | #11

[quote="moonsettler, post:10, topic:327"]
even if you only make native asset transactions marginally more expensive, while forcing alternative assets to be more economic than they are (say via taptweak and client side validation)
[/quote]

Indeed, we were not on the same page. I don't suggest making "native asset transactions marginally more expensive". On the contrary, I suggest that they should be less expensive. 

For example, 6000 simple transactions in a 2 MB block should not cost more sats than 1 complex transaction of the same (2 MB) size.

-------------------------

moonsettler | 2024-01-08 10:19:13 UTC | #12

[quote="GregTonoski, post:11, topic:327"]
Indeed, we were not on the same page. I don’t suggest making “native asset transactions marginally more expensive”. On the contrary, I suggest that they should be less expensive.

For example, 6000 simple transactions in a 2 MB block should not cost more sats than 1 complex transaction of the same (2 MB) size.
[/quote]

that's not how it works imo. the subsidy is a consequence of the space available for various parts. if you reduce the subsidy it means there is less block space. it gets more expensive with the same demand present. the witness bytes become more expensive so native economic transactions get marginally more expensive as a direct consequence. also the cost is highly related to creating new outputs. 6000 transactions creating ~12000 new outputs should be more expensive than a large script transaction creating 1 or 2 outputs.

but like i said, if you only want to remove the unproportional advantage from segwit discount and willing to throw things like BitVM under the bus, you can use the formula i posted above. that likely won't harm 'normal' users of bitcoin.

-------------------------

GregTonoski | 2024-01-08 10:49:03 UTC | #13

[quote="moonsettler, post:12, topic:327"]
6000 transactions creating ~12000 new outputs should be more expensive than a large script transaction creating 1 or 2 outputs.
[/quote]

What makes you think so?
[quote="moonsettler, post:12, topic:327"]
throw things like BitVM under the bus
[/quote]

I just want not to throw simple transactions under the bus.

-------------------------

