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

