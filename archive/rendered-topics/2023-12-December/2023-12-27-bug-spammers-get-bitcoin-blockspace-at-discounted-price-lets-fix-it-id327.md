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

