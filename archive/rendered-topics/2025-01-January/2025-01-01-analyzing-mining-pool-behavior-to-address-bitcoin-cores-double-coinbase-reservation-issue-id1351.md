# Analyzing Mining Pool Behavior to Address Bitcoin Core's Double Coinbase Reservation Issue

ismaelsadeeq | 2025-01-16 18:37:06 UTC | #1

In theory, Bitcoin Core's block-building algorithm reserves **4000 WU** for block header, transaction count and miners' coinbase transactions.
This means it should generate a block template with a weight of **3,996,000 WU**.  
However, in practice, this is not the case due to an accidental double-reservation bug ([see #21950](https://github.com/bitcoin/bitcoin/issues/21950)).  
As a result, Bitcoin Core generates block templates with a weight of **3,992,000 WU**.  

There is an ongoing attempt to fix this issue ([see #31384](https://github.com/bitcoin/bitcoin/pull/31384)), but there are [concerns](https://github.com/bitcoin/bitcoin/pull/31384#pullrequestreview-2489253785) that fixing the issue could lead to some miners generating invalid block templates.  
If the PR were to be merged, mining pools using the updated Bitcoin Core version must ensure the sum of the coinbase transaction weight and the weight of all additional transactions added to the block template must not exceed **4000 WU**.  
Otherwise, the block will exceed the consensus limit for the maximum block weight (**4,000,000 WU**) and be invalid.

The proposed PR fixes the double reservation issue, which means the block template size is increased by **4000 WU**, resulting in blocks with a weight of **3,996,000 WU** as expected and matches theory.

Miners or pools needing more than **4000 WU** for their coinbase transactions should configure `bitcoind` with a lower `-blockmaxweight` option.  
For example, starting Bitcoin Core with `-blockmaxweight=3999000` will free up an additional **1000 WU**.  

The purpose of this analysis is to examine mining pool behavior and determine the feasibility of this change.
- **Blocks Analyzed**: 107,313  
- **Period**: 24th December 2022 to 23rd December 2024
### Methodology

- I've used [libbitcoinkernel](https://github.com/stickies-v/py-bitcoinkernel/) to read blocks from disk.  
- Each block is deserialized, and [bitcoin-data/mining-pools](https://github.com/bitcoin-data/mining-pools) tags are used to match the block to a mining pool. The block weight, coinbase transaction, and mining pool information are then saved to disk.  
- After saving the data for the entire block interval, I generated the analysis results using [this script](https://github.com/ismaelsadeeq/mining-analysis?tab=readme-ov-file#3-generate_resultspy).  
- Charts were generated using a set of [scripts](https://github.com/ismaelsadeeq/mining-analysis?tab=readme-ov-file#4-graph-generation-scripts).  
- More details on the methodology can be found https://github.com/ismaelsadeeq/mining-analysis.


## Results

Note: Values in paranthesis represent block heights

### Pools Generating Blocks with Coinbase Weight > 4000 WU

|Pool Name | Average Coinbase weight  | Lowest Coinbase weight | Highest coinbase weight |
|--- | --- | --- | --- |
|Ocean.xyz | 6994 | 5308 (865820) | 9272 (863471) |
|Unknown pool | 7432 | 7000 (824116) | 7864 (785117) |

#### Figure 1: Box Plot of Coinbase Transaction Weights > 4000 WU
This box plot shows the distribution of coinbase transaction weights for pools generating blocks with coinbase weights greater than 4000 WU.

![Large Coinbase Transaction Weights|690x417](upload://txplM5i0pl6hVyorGFZL04MFxwL.png)

### Remaining mining pools coinbase stats

|Pool Name|Average Coinbase Weight|Lowest Coinbase Weight|Highest Coinbase Weight|
|---------|---------------------|---------------------|---------------------|
|ViaBTC|1,128|864 (778363)|1,284 (868424)|
|Foundry USA|834|748 (833764)|1,192 (868391)|
|F2Pool|1,610|1,176 (787112)|1,832 (868388)|
|AntPool|1,357|732 (814783)|1,612 (875136)|
|Binance Pool|1,151|684 (789485)|1,480 (810101)|
|SpiderPool|997|704 (817123)|1,296 (863176)|
|Braiins Pool|1,056|684 (853433)|1,232 (785389)|
|MARA Pool|815|720 (800883)|1,016 (833308)|
|SecPool|1,371|1,088 (808120)|1,460 (871380)|
|Poolin|1,110|728 (852695)|1,336 (795979)|
|Luxor|1,237|780 (770527)|1,480 (868404)|
|BTC.com|1,205|740 (796334)|1,476 (811469)|
|Pega Pool|688|688 (800892)|688 (800892)|
|Ultimus Pool|1,209|1,056 (853449)|1,444 (844214)|
|EMCDPool|1,082|1,080 (804771)|1,304 (795962)|
|SBI Crypto|716|716 (833247)|716 (833247)|
|NiceHash|786|780 (804872)|828 (870207)|
|Titan|696|696 (770484)|696 (770484)|
|WhitePool|887|676 (820630)|1,304 (865060)|
|KuCoin Pool|1,095|1,092 (777136)|1,316 (796122)|
|CleanIncentive|728|728 (819134)|728 (819134)|
|1THash|1,084|1,084 (837937)|1,088 (864350)|
|EclipseMC|860|860 (858559)|860 (858559)|
|Terra Pool|844|844 (780190)|844 (780190)|
|CKPool|856|856 (863890)|856 (863890)|

# Figure 2: Line Chart of Coinbase Transaction Weights
This line chart shows the average, minimum, and maximum coinbase transaction weights for various mining pools.
![Line Chart of Coinbase Transaction Weights|690x417](upload://9V3sKOZGNlGSZPRcm22NFSRKSyC.png)

### Pools with Blocks (> 3,996,000 WU)


| Pool Name | Average Block Weight | Lowest Block Weight | Highest Block Weight |
| --- | --- | --- | --- |
| F2Pool | 3,997,857 | 3,994,044 (828959) | 3,998,554 (779729) |

### Pools with Blocks (< 3,996,000 WU)

| Pool Name | Average Block Weight | Lowest Block Weight | Highest Block Weight |
| --- | --- | --- | --- |
| Foundry USA    | 3,993,030         | 3,992,749 (813879)       | 3,993,523 (874770) |
| Binance Pool   | 3,993,348         | 3,992,753 (784937)       | 3,993,768 (846996) |
| MARA Pool      | 3,993,017         | 3,992,721 (802653)       | 3,994,944 (859968) |
| Luxor          | 3,993,434         | 3,992,782 (768718)       | 3,993,811 (841510) |
| AntPool        | 3,993,554         | 3,992,868 (814783)       | 3,993,918 (873367) |
| ViaBTC         | 3,993,324         | 3,992,881 (782955)       | 3,993,612 (872513) |
| Poolin         | 3,993,308         | 3,992,987 (855538)       | 3,993,613 (795979) |
| Braiins Pool   | 3,993,251         | 3,992,695 (853433)       | 3,993,416 (853776) |
| SBI Crypto     | 3,992,911         | 3,992,717 (796840)       | 3,993,047 (834797) |
| Ultimus Pool   | 3,993,407         | 3,993,076 (852748)       | 3,993,768 (842345) |
| BTC.com        | 3,993,399         | 3,992,745 (796409)       | 3,993,781 (806063) |
| SpiderPool     | 3,993,190         | 3,992,912 (856599)       | 3,993,616 (864899) |
| WhitePool      | 3,993,072         | 3,992,677 (825268)       | 3,993,631 (872281) |
| Ocean.xyz      | 3,988,137         | 3,986,489 (865820)       | 3,990,301 (863471) |
| EMCDPool       | 3,993,273         | 3,993,081 (781934)       | 3,993,428 (796046) |
| Pega Pool      | 3,992,894         | 3,992,690 (783729)       | 3,993,018 (781576) |
| Titan          | 3,992,974         | 3,992,926 (768641)       | 3,993,020 (769939) |
| KuCoin Pool    | 3,993,282         | 3,993,101 (781030)       | 3,993,504 (796188) |
| Terra Pool     | 3,993,032         | 3,992,863 (776927)       | 3,993,172 (780396) |
| CleanIncentive | 3,992,946         | 3,992,748 (823029)       | 3,993,038 (822604) |
| 1THash         | 3,993,276         | 3,993,085 (837937)       | 3,993,412 (814830) |
| NiceHash       | 3,992,987         | 3,992,782 (789503)       | 3,993,148 (785085) |
| CKPool         | 3,993,102         | 3,992,897 (863890)       | 3,993,181 (822636) |

#### Figure 3: Line Chart of Mining pool block Weights
This line chart shows the average, minimum, and maximum block weights for various mining pools.

![Block Weights for Pools|690x417](upload://3Lq4pJ9Ur9jpFqOYzRXohjAclOb.png)


### Key Observations

- Most mining pools adhere to Bitcoin Core’s default setup, keeping coinbase transaction weights well below **4000 WU**. However, two pools stand out:
  - **Ocean.xyz** consistently uses larger coinbase transactions, with an average weight of **6,994 WU**. Their smallest coinbase weight was **5,308 WU** (in block 865820), and their largest reached **9,272 WU** (block 863471). This suggests Ocean.xyz configures its node with a reduced `-blockmaxweight` setting, as their average block size is **3,986,489 WU**, well below the default template size.
  - An **Unknown Pool/miner** also exceeded the default coinbase weight, averaging **7,432 WU**, with a maximum of **7,864 WU** but a very low block weight average which is unusual behavior.
- **F2Pool** blocks have an average weight of **3,997,857 WU**. For instance, block **779729** reached **3,998,554 WU** with a coinbase weight of just **1,588 WU**, showcasing F2 optimal use of the available block weight.
- The majority of pools—such as Foundry USA, Binance Pool, and AntPool—produce blocks averaging around **3,993,000 WU**, consistent with Bitcoin Core’s current behavior. However, this depicts underutilization which is a direct result of the double-reservation bug.
- ~~The small increase in the weights of the blocks among these pools suggests minor additions of transactions by miners after building a block template~~.
- The small increase in the weights of the blocks after selecting the transactions is accounted for the block headers, as noted by @0xB10C [here](https://delvingbitcoin.org/t/analyzing-mining-pool-behavior-to-address-bitcoin-cores-double-coinbase-reservation-issue/1351/4?u=ismaelsadeeq)

If the double-reservation issue is fixed, block templates will increase to **3,996,000 WU**, aligning theoretical and actual limits. This change benefits pools by allowing full utilization of the block weight while preserving **4000 WU** for coinbase transactions and other small additions.

Miners who need more space, such as Ocean.xyz, can still configure their nodes with a reduced `-blockmaxweight`. For example, setting it to **3,999,000 WU** frees up **1000 WU** for larger coinbase transactions, ensuring compliance with consensus rules.

I think addressing this bug will enhance fairness across the mining pools and fix the ambiguity for developers working with the bitcoin core codebase.

-------------------------

everythingSats | 2025-01-04 08:00:05 UTC | #2

How much of Ocean's data, puts into consideration, their DATUM server, which gives hashers opportunity to generate their own Block template, and edit the miner tag. This I believe is a nuance worthy of noting. Thoughts?

-------------------------

0xB10C | 2025-01-13 16:48:04 UTC | #3

> * The small increase in the weights of the blocks among these pools suggests minor additions of transactions by miners after building a block template.

I was surprised by this. I initially looked at SBI Crypto and noticed that their max block weight of 3,993,047 WU minus the max coinbase weight of 716 WU is 3,992,331 WU. That's 331 WU more than the soft-maximum of 3,992,000 WU I was expecting. Looking at all pools, it seems it's quite often around that 331 number, never more though.

|Pool Name|max block weight|max coinbase weight|weight above 3,992,000 |notes|
|---|---|---|---|---|
|Foundry USA|3993523|1192|331||
|Binance Pool|3993768|1440|328||
|MARA Pool|3994944|904|2040|block with big inscription|
|Luxor|3993811|1480|331||
|AntPool|3993918|1612|306||
|ViaBTC|3993612|1284|328||
|Poolin|3993613|1336|277||
|Braiins Pool|3993416|1124|292||
|SBI Crypto|3993047|716|331||
|Ultimus Pool|3993768|1440|328||
|BTC-com|3993781|1472|309||
|SpiderPool|3993616|1288|328||
|WhitePool|3993631|1304|327||
|Ocean.xyz|3990301|9272|-10971|Ocean doesn’t use the default|
|EMCDPool|3993428|1304|124||
|Pega Pool|3993018|688|330||
|Titan|3993020|696|324||
|KuCoin Pool|3993504|1316|188||
|Terra Pool|3993172|844|328||
|CleanIncentive|3993038|728|310||
|1THash|3993412|1084|328||
|NiceHash|3993148|820|328||
|CKPool|3993181|856|325||

I'd be surprised if this is really due to ALL pools adding extra transactions AFTER they got a template from Bitcoin Core. That seems risky (why not do it with `prioritizetransaction`?) and I haven't noticed this in any of the blocks processed by my [miningpool-observer](github.com/0xb10c/miningpool-observer) project.

Is there possibly more to it?

-------------------------

0xB10C | 2025-01-14 08:29:19 UTC | #4

[quote="0xB10C, post:3, topic:1351"]
Is there possibly more to it?
[/quote]

Yes :man_facepalming:, there is more to it. Especially to a serialized block...
- the header is 80 bytes, accounting for 320 WU
- usually, a 3 byte var_int for the number of transactions accounting for 12 WU (if a 1 byte var_int is enough, this is 4 WU)

This leaves us with 332 WU. As we check that the package weight is [NOT `>= 3,992,000`](https://github.com/bitcoin/bitcoin/blob/35bf426e02210c1bbb04926f4ca2e0285fbfcd11/src/node/miner.cpp#L200-L201) (i.e. we check that `< 3,992,000`), we actually have a soft-maximum of 3,991,999 WU.

This allows looking at how close the pools came to actually fill the block: Seems like Foundry, SBI Crypto, Braiins, and Luxor had 0 WU left in their blocks. This also indicates that all listed pools are currently using the default `-blockmaxweight=3999000`. 

|Pool Name | max block weight | max coinbase weight | min WU left to fill|
|--- | --- | --- | ---|
|Foundry USA | 3993523 | 1192 | 0|
|Binance Pool | 3993768 | 1440 | 3|
|Luxor | 3993811 | 1480 | 0|
|AntPool | 3993918 | 1612 | 25|
|ViaBTC | 3993612 | 1284 | 3|
|Poolin | 3993613 | 1336 | 54|
|Braiins Pool | 3993416 | 1124 | 39|
|SBI Crypto | 3993047 | 716 | 0|
|Ultimus Pool | 3993768 | 1440 | 3|
|BTC-com | 3993781 | 1472 | 22|
|SpiderPool | 3993616 | 1288 | 3|
|WhitePool | 3993631 | 1304 | 4|
|EMCDPool | 3993428 | 1304 | 207|
|Pega Pool | 3993018 | 688 | 1|
|Titan | 3993020 | 696 | 7|
|KuCoin Pool | 3993504 | 1316 | 143|
|Terra Pool | 3993172 | 844 | 3|
|CleanIncentive | 3993038 | 728 | 21|
|1THash | 3993412 | 1084 | 3|
|NiceHash | 3993148 | 820 | 3|
|CKPool | 3993181 | 856 | 6|


[quote="ismaelsadeeq, post:1, topic:1351"]
The small increase in the weights of the blocks among these pools suggests minor additions of transactions by miners after building a block template.
[/quote]

This is probably incorrect then.

-------------------------

ismaelsadeeq | 2025-01-13 20:21:16 UTC | #5

Thanks so much for looking into this in depth
[quote="0xB10C, post:3, topic:1351"]
I was surprised by this.
[/quote]

Same, but unfortunately, I did not account for the block header size.
It all makes sense now.

I’ll strike that through and note that the small increase is due to the size of the block headers.

-------------------------

0xB10C | 2025-01-15 08:16:48 UTC | #6

[quote="ismaelsadeeq, post:1, topic:1351"]
### Pools with Blocks (> 3,996,000 WU)

|Pool Name|Avg. Block Weight|Min. Block Weight|Min. Coinbase Weight (Height)|Max. Block Weight|Max. Coinbase Weight (Height)|
| --- | --- | --- | --- | --- | --- |
|**F2Pool**|3,997,857|3,994,044 (828959)|1724 (800183)|3,998,554 (779729)|1364 (786721)|
[/quote]

With your recent change to https://github.com/bitcoin/bitcoin/pull/31384#issuecomment-2591295705 to have a minimal reserved block weight of 2000 WU, I came back here to double check with the F2Pool's stats.
1. Can you clarify F2Pool's min and max coinbase weights? In the table above the min is larger than max.
2. For block 779729 with the max block weight of 3,998,554, a coinbase weight of ‎1588 WU and 332 WU for header+tx_count leaves me with a default of 3,996,634 WU. That seems to be an odd number to use as a custom `-blockmaxweight` (and only possible with patches to Bitcoin Core).
3. If you have the data, it might be interesting to calculate `block weight - coinbase weight - header weight` for the F2Pool blocks to get a better feeling about the weight they reserve.

-------------------------

ismaelsadeeq | 2025-01-16 18:43:50 UTC | #7

> Can you clarify F2Pool’s min and max coinbase weights? In the table above the min is larger than max.

This inconsistency is due to a mix-up when compiling the data.
Also, I meant the minimum block weight coinbase weight, not the minimum coinbase weight. I’ve separated the coinbase weight stats from the block weight stats and update the coinbase chart.

> * For block 779729 with the max block weight of 3,998,554, a coinbase weight of ‎1588 WU and 332 WU for header+tx_count leaves me with a default of 3,996,634 WU. That seems to be an odd number to use as a custom `-blockmaxweight` (and only possible with patches to Bitcoin Core).

>  If you have the data, it might be interesting to calculate `block weight - coinbase weight - header weight` for the F2Pool blocks to get a better feeling about the weight they reserve.

I think we are a certain that F2Pool is using a patch of Bitcoin Core after https://github.com/bitcoin/bitcoin/pull/31384#issuecomment-2593329870

However, it would be useful to review the data for confirmation.

-------------------------

