# Analysis of attempting to imbue LN commitment transaction spends with v3 semantics

sdaftuar | 2024-02-07 16:47:29 UTC | #1

## Overview
There has been some discussion about taking the v3 policy proposal, which is an opt-in policy for transaction setting `nVersion=3`, and trying to directly apply it to transactions that appear to be LN commitment transaction spends, based on matching characteristics of such transactions.  This would allow the LN project to adopt v3 without making any explicit changes to their software.  

This raises questions about what effects such a change would have, so I ran a simulation[^1] over all transaction data that I recorded in 2023 to log which transactions would have:
 1. Matched against our
LN commitment template (implemented [here](https://github.com/bitcoin/bitcoin/commit/a5fe461838e8759c9abd3ac764162c271f97d50b)), and
 1. Failed to be accepted to the mempool under the proposed v3 rules (1 parent-1 child, 1000vb max child size, implemented in https://github.com/bitcoin/bitcoin/pull/28948).

Results:
* Total transactions that would have been imbued with v3 validation rules: **14124**
* Total transactions that would have failed validation using the proposed rules: **856 (6.06%)**

### Question 1: Is the template matching catching more than just LN commitment transaction anchor spends?

Specifically, are any of the transactions that fail validation likely to be something other than LN anchor spends?

To answer this question, for each of the 856 failed transactions, I ran decodescript on the last stack element
in each input's witness, to see if any of those stack elements appeared to be scripts matching the LN anchor spend template
described [here](https://github.com/bitcoin/bitcoin/issues/29319#issuecomment-1916166471).

Of the 856 failed transactions, 173 were never mined [^2]; 508 had an input matching that script template, and the remaining 175 had
a direct parent which matched the LN anchor spend template.  So assuming that script template is uniquely in use by LN 
implementations, then it would appear no other transactions that would fail these rules would have been picked up by the template.

### Question 2: What are the reasons that these transactions are rejected by the v3 validation rules?

* 595 of the transactions failed due to an ancestor count limit being hit (eg batch CPFP or > 1 descendant tx in chain)
* 251 of the transactions failed due to a descendant count limit being hit (multiple anchor spends of the same commitment tx)
* 10 transactions failed due to a size limit being hit. Here are the observed sizes:

> 0b78e1df093b1f621296d37d26bb8e8ab98d8319e0305cc8df513ca68f840d01 failed (v3 child tx is too big: 1044 > 1000 virtual bytes)
> 186bbee98e83093a19536d07493cbac81cce7ce04b00bd0ce492c346604baa01 failed (v3 child tx is too big: 1352 > 1000 virtual bytes)
> 64141aa64f9beb3d67eb9f7b68946cc4aa7b8644faf022ca08c986b9cc004287 failed (v3 child tx is too big: 1304 > 1000 virtual bytes)
> 709f6bfb9f0522fb2a3543fab6a0b49e4fd8044318e685c2457f7971010628f9 failed (v3 child tx is too big: 1827 > 1000 virtual bytes)
> 73bae426287150bdc922e9681c81fb8a96b1cc07ebaa97fb60b0fe32849375c4 failed (v3 child tx is too big: 1555 > 1000 virtual bytes)
> 8effcb0a127a8e1538c7bcb1f4244da5d8eb5943c07cfe91e2a7c0d9c954636d failed (v3 child tx is too big: 1504 > 1000 virtual bytes)
> 98c5faa56bdafdad6fc568c3be5abeb43b64948539af4c83e6ee75924b44797d failed (v3 child tx is too big: 1216 > 1000 virtual bytes)
> a936cf2c40af49c10a167e0d6a2b3bf18e256cbd0921d5ca1ca385ade32660ef failed (v3 child tx is too big: 1210 > 1000 virtual bytes)
> d357ed9634a80320cd68c7dfe46912824cd44328d79f3aae1c12e96c6cf61d56 failed (v3 child tx is too big: 1221 > 1000 virtual bytes)
> e7f32d8286686443341f93177c67147e5d06df868b11baf7a00034ddd871d981 failed (v3 child tx is too big: 1137 > 1000 virtual bytes)


### Question 3: How can we characterize the transactions that fail due to ancestor count limit?
Of the 595 transactions rejected for this reason:
 * 99 were never mined.
 * 175 of these transactions are because the rejected transaction would have created a length 2 descendant chain, ie the rejected transaction was a spend of an anchor spend.
 * 302 appear to be batched CPFP transactions, where multiple anchors are spent in the same tx. I looked at how many inputs matched the anchor spend script template and observed they ranged in size from 2-6 anchor inputs:
   | Number of anchors spent in a single tx | Number of txs |
   | -----------------------|-------------------------|
   | 2 | 252 |
   | 3 | 37 | 
   | 4 | 8 |
   | 5 | 3 |
   | 6 | 2 |
   | **Grand total** | **302** |
  - The remaining 19 transactions had only 1 anchor spend; these transactions appear to have been spending an anchor output as well as the output of some other unconfirmed transaction, though I haven't exhaustively analyzed these. Here are the txids of the 19 transactions in question:

> 0f4fb5de29eb33061376eda09d3beab22cf839efb21e5723559bab00f9008ddd
> 19996cb0fa68f5f712c9c41920ae2bf7c6f643544922a2dd7f0ff631817ab98d
> 1a098e6320eb9a430906df82356b3a7441874c503284aea1bd91e16c77b35be7
> 1fdacc8695460ce0177acadb01e80e2dffb80a11e85fae6e3d73b354f02205d3
> 21daa691908ff940b1e8b6aa71b8452796f39584cc1e3bfe2a746a74907d827e
> 2d429872c753fa26349d18c6cab25dc19ec7fd6870b51344f07b9bdec72450c2
> 2e871b6d2a27069cc4ba7ce98cc0858d41444df08bd71622c6c4de92178dedd7
> 3650677716c8a99ad89cc6f0ffd5a66136f43454458f05f165ba95dcca0ab503
> 3be9d7f294616fefe213ea50ffcb6d1ad2fbd1aebe127350d96b7f1e34b01b77
> 4106b6dde4ac651268759c0a52ec23871fa0f2c5dabd368c3119e40566287b62
> 4e2da138435ca950afa51f684977ac9fe85f10abecb7bd3ae4d9186a0cf3972b
> 7d4ec17ea3fefbf20d6072a42b4eb1dd65d5a099166f0300c8ea83a8af268a4e
> 82626a3491d7fb4ce4da7df848205950304d55f5089eedec42142469c2b79a7a
> 82d1ad4f2657183b1f2caaac5877fcf14b5cdb78d55f89fa9f49910d5061e49b
> 86d616ae6beb8b28f443bb86a48834f983934dce2574ac5b1a1def0b698e9ddd
> 8b4a7bab2556389eca15af7aed3260d3f8c5190627ff8a362cde3bd4a379fdc2
> 917c22ee5b0db446b5d8b9d18974efb0c557b19bd27f89fd9e2a0ea47c19bcb3
> c428c5b9baba65e9a8d531abe862b9f5529413af9df879b5d5a660a8c1b8949c
> d08995dfd33104c153c6f57573dd5d4e66287870641404457eab0e3516b84f0a


### Question 4: Across all transactions that matched the template for imbued v3 semantics, what is the distribution of child transaction vsizes?

Here is a histogram I generated of all 14124 transactions (each bin is a range of 25 vbytes). Almost all transactions are small, so this is on a log scale:

![histogram](upload://9gnW5FbIRkwMxz5MbkZXJrhn8ad.png)




[^1]:The code I ran is here: https://github.com/sdaftuar/bitcoin/tree/2024-01-sim-imbued-v3
[^2]:I think it's not a concern if the policy rule would prevent some transactions from being relayed which were never mined anyway, so I have not done any further analysis on these transactions.

-------------------------

t-bast | 2024-02-08 08:16:12 UTC | #2

[quote="sdaftuar, post:1, topic:527"]
The remaining 19 transactions had only 1 anchor spend; these transactions appear to have been spending an anchor output as well as the output of some other unconfirmed transaction, though I haven’t exhaustively analyzed these. Here are the txids of the 19 transactions in question:
[/quote]

This could be eclair's behavior: when spending an anchor, we currently allow using a safe unconfirmed wallet input. But we can easily change that behavior, and even without changing anything, if the transaction is rejected from our mempool, we automatically retry funding it at the next block, so we would very likely eventually fund with a confirmed input (unless none are available of course).

-------------------------

