# Lehar / Parlour Paper

micah541 | 2026-02-03 23:19:08 UTC | #1

There’s an academic  paper [“Market Power and the Bitcoin Protocol”](http://ewfs.org/wp-content/uploads/2025/03/tx_fees-18.pdf) that claims to have strong empirical evidence that miners are strategically leaving fees on the table as this drives up average fees over time. 

Is there criticism or agreement with this conclusion?  It certainly contradicts that general wisdom that miners are fee-maximizing with every single block.

-------------------------

ajtowns | 2026-02-04 15:26:34 UTC | #2

They define a block as "full" if "if at most an additional 2,000 weight units could have been processed in that block."

For comparison, there are two values that control how much free capacity a block has in the code; there's the `-blockreservedweight` which defaults to 8000 weight units and is limited to a minimum of 2000 units, which is reserved capacity for the coinbase transaction, and the hardcoded `BLOCK_FULL_ENOUGH_WEIGHT_DELTA` fixed at 4000 weight units which indicates when the block is "full enough" and should limit how many more txs are tried.

With those figures, I think if miners were not tweaking the defaults and thus were optimising fee income on a per-block basis, that you'd expect that the free space in a block plus the size of the block's coinbase transaction to be around 8000 weight units, and that seems to be what you observe in practice:

```sh
$ # one-liner to report (block-height coinbase-weight block-freespace sum)
$ (b=$(bitcoin-cli getblockcount); for a in `seq 0 500`; do bitcoin-cli getblock $(bitcoin-cli getblockhash $(($b-$a)) ) 2 | jq -j '.height, " ", .tx[0].weight, " ", 4000000-.weight, " ", 4000000-.weight+.tx[0].weight, "\n"'; done)
934948 1404 6394 7798
934947 1404 6544 7948
934946 1776 5970 7746
934945 1404 6896 8300
934944 2032 1836 3868
934943 1404 6297 7701
934942 1404 6603 8007
...
```

They also define "priority violations" in a way that seems like it will hit any time a miner uses "prioritisetransaction": "A priority violation occurs if a block contains a transaction and there are at least five transactions waiting in the mempool that [have a higher fee]". If the minimum feerate for a block is 5 sats/vb, and there are 4 sats/vb transactions available, then including a 1sat/vb (even if it has been paid for out of band via mempool.space accelerator) would match under this scheme, which doesn't seem appropriate. The definition is also written in terms of absolute fees, rather than fee rates, which would also lead to meaningless results, when large transactions paying modest fees are ignored in favour of smaller transactions with smaller fees on an individual basis, but that pay a higher fee when aggregated.

There is some later talk about stale blocks ("orphan blocks" in the paper) that uses a sample of 57 stale blocks from 2016-2019. That seems a pretty small number; the [stale-blocks](https://github.com/bitcoin-data/stale-blocks/) repo has 290 stale blocks (202 with full block data) over roughly the specified period (blocks 391000 to 590000?). They compare the orphan block to a window of "8000 blocks (approximately 4 days)", however 8000 blocks is actually about two months (4 days would be ~600 blocks), and in any event, it's not clear what relevance a block even a few hour later has to a stale block.

So, I don't find the conclusions about what miners are doing in practice to be well founded.

-------------------------

micah541 | 2026-02-04 18:16:44 UTC | #3

Thanks!  Seems like some important mechanics to miss.  It appears then that their observations are completely in line with a null hypothesis.

Is there an easy explanation for all the empty blocks that appear several minutes or more since the previous block?

-------------------------

AntoineP | 2026-02-04 22:45:27 UTC | #4

[quote="micah541, post:3, topic:2225"]
Is there an easy explanation for all the empty blocks that appear several minutes or more since the previous block?
[/quote]

I assume you are talking about the results presented on page 60:
> There is no direct connection between the probability of a block being empty and the time
elapsed since the previous block was mined, which would be the case of latency was a
problem. Starting in 2013 we find that the average time after which an empty block was
mined to be 9.69 minutes compared to 9.49 minutes for a full block. We find 54,966 non-
empty blocks mined within less than minute, and 2,505 non-empty blocks mined within less than 5 seconds. Similarly we find 2,156 empty blocks mined more than 10 minutes
after their predecessor.

I would be very surprised if ~4% of empty blocks nowadays arrived 10 minutes after their parent. So my guess would be that this is largely skewed toward a decade ago when empty blocks were a lot more frequent, and mining pools a lot less competent.

I suppose another hypothesis could be that, assuming the authors used the block's timestamp as time source instead of the time of arrival, the miner of these empty blocks was using timestamp rolling. Or the parent of these empty blocks had a timestamp unusually set further in the past.

To try and get some data supporting my hunch i ran this hacky command on a node to inspect which blocks were empty between height 800,000 (July 2023) and 900,000 (June 2025), and how much higher is their timestamp set compared to the previous block in the chain:
```
(ts_last="0"; for h in `seq 800000 900000`; do res="$(./bitcoin-cli getblock $(./bitcoin-cli getblockhash $h) 1)"; ts="$(echo $res | jq -r .time)"; interval="$(($ts - $ts_last))"; echo "$res" | jq --arg interval "$interval" -j '(if .tx | length == 1 then "block at \(.height) is empty, its timestamp is \($interval) seconds later than the previous block\n" else empty end)'; ts_last="$ts"; done)
```

The result is that only 278 blocks during this period were empty, confirming that empty blocks are a lot less common nowadays. The vast majority of them have a timestamp within a minute of their parent, and none have a timestamp over 10 minutes after their parent.

<details>
<summary>Expand results</summary>

```
block at 800486 is empty, its timestamp is 36 seconds later than the previous block
block at 800569 is empty, its timestamp is 12 seconds later than the previous block 
block at 800789 is empty, its timestamp is 18 seconds later than the previous block
block at 801325 is empty, its timestamp is 32 seconds later than the previous block
block at 801628 is empty, its timestamp is 12 seconds later than the previous block
block at 802331 is empty, its timestamp is 10 seconds later than the previous block 
block at 802594 is empty, its timestamp is 28 seconds later than the previous block
block at 803543 is empty, its timestamp is 22 seconds later than the previous block
block at 804683 is empty, its timestamp is 43 seconds later than the previous block
block at 804739 is empty, its timestamp is 48 seconds later than the previous block 
block at 805253 is empty, its timestamp is 26 seconds later than the previous block 
block at 806043 is empty, its timestamp is 30 seconds later than the previous block 
block at 806424 is empty, its timestamp is 25 seconds later than the previous block
block at 806429 is empty, its timestamp is 18 seconds later than the previous block
block at 807374 is empty, its timestamp is 19 seconds later than the previous block
block at 807520 is empty, its timestamp is 21 seconds later than the previous block
block at 808357 is empty, its timestamp is 23 seconds later than the previous block
block at 809586 is empty, its timestamp is 14 seconds later than the previous block 
block at 810755 is empty, its timestamp is 32 seconds later than the previous block
block at 810811 is empty, its timestamp is 11 seconds later than the previous block
block at 811426 is empty, its timestamp is 11 seconds later than the previous block
block at 811458 is empty, its timestamp is 21 seconds later than the previous block
block at 811881 is empty, its timestamp is 25 seconds later than the previous block
block at 812736 is empty, its timestamp is 5 seconds later than the previous block
block at 813365 is empty, its timestamp is 24 seconds later than the previous block 
block at 814393 is empty, its timestamp is 26 seconds later than the previous block
block at 814423 is empty, its timestamp is 26 seconds later than the previous block
block at 814510 is empty, its timestamp is 25 seconds later than the previous block
block at 816278 is empty, its timestamp is -64 seconds later than the previous block
block at 816841 is empty, its timestamp is 12 seconds later than the previous block
block at 816870 is empty, its timestamp is 7 seconds later than the previous block 
block at 816929 is empty, its timestamp is 14 seconds later than the previous block
block at 816977 is empty, its timestamp is 23 seconds later than the previous block
block at 817214 is empty, its timestamp is 37 seconds later than the previous block
block at 817573 is empty, its timestamp is 5 seconds later than the previous block 
block at 817861 is empty, its timestamp is 8 seconds later than the previous block 
block at 817980 is empty, its timestamp is 50 seconds later than the previous block
block at 818036 is empty, its timestamp is 20 seconds later than the previous block
block at 818116 is empty, its timestamp is 24 seconds later than the previous block
block at 818882 is empty, its timestamp is 64 seconds later than the previous block
block at 818904 is empty, its timestamp is 12 seconds later than the previous block
block at 818960 is empty, its timestamp is 14 seconds later than the previous block
block at 819045 is empty, its timestamp is 5 seconds later than the previous block  
block at 819056 is empty, its timestamp is 15 seconds later than the previous block
block at 819384 is empty, its timestamp is 36 seconds later than the previous block
block at 819448 is empty, its timestamp is 5 seconds later than the previous block 
block at 819601 is empty, its timestamp is 24 seconds later than the previous block
block at 819831 is empty, its timestamp is 4 seconds later than the previous block  
block at 820037 is empty, its timestamp is 21 seconds later than the previous block
block at 820645 is empty, its timestamp is 13 seconds later than the previous block
block at 820856 is empty, its timestamp is 18 seconds later than the previous block
block at 820874 is empty, its timestamp is 25 seconds later than the previous block
block at 820877 is empty, its timestamp is 23 seconds later than the previous block
block at 821060 is empty, its timestamp is 22 seconds later than the previous block
block at 821241 is empty, its timestamp is -28 seconds later than the previous block
block at 821293 is empty, its timestamp is 29 seconds later than the previous block
block at 821507 is empty, its timestamp is 4 seconds later than the previous block 
block at 821539 is empty, its timestamp is 30 seconds later than the previous block
block at 821741 is empty, its timestamp is 4 seconds later than the previous block  
block at 821785 is empty, its timestamp is 13 seconds later than the previous block
block at 821933 is empty, its timestamp is 5 seconds later than the previous block 
block at 822046 is empty, its timestamp is 38 seconds later than the previous block
block at 822128 is empty, its timestamp is 12 seconds later than the previous block
block at 822254 is empty, its timestamp is 105 seconds later than the previous block
block at 822472 is empty, its timestamp is 15 seconds later than the previous block
block at 822755 is empty, its timestamp is 28 seconds later than the previous block
block at 823176 is empty, its timestamp is 16 seconds later than the previous block
block at 823902 is empty, its timestamp is 17 seconds later than the previous block
block at 823925 is empty, its timestamp is 40 seconds later than the previous block
block at 825261 is empty, its timestamp is 28 seconds later than the previous block
block at 825493 is empty, its timestamp is 18 seconds later than the previous block
block at 825498 is empty, its timestamp is 20 seconds later than the previous block
block at 825561 is empty, its timestamp is 21 seconds later than the previous block
block at 825999 is empty, its timestamp is 16 seconds later than the previous block 
block at 826077 is empty, its timestamp is 27 seconds later than the previous block
block at 826284 is empty, its timestamp is 83 seconds later than the previous block
block at 827402 is empty, its timestamp is 21 seconds later than the previous block 
block at 827452 is empty, its timestamp is 33 seconds later than the previous block
block at 827453 is empty, its timestamp is 15 seconds later than the previous block
block at 827865 is empty, its timestamp is 51 seconds later than the previous block                                                                                                                                                                                                                                                                                                                                  
block at 828012 is empty, its timestamp is 516 seconds later than the previous block
block at 828594 is empty, its timestamp is 21 seconds later than the previous block
block at 828620 is empty, its timestamp is 16 seconds later than the previous block
block at 828651 is empty, its timestamp is 9 seconds later than the previous block 
block at 829466 is empty, its timestamp is 20 seconds later than the previous block
block at 829893 is empty, its timestamp is 9 seconds later than the previous block
block at 830534 is empty, its timestamp is 10 seconds later than the previous block
block at 831871 is empty, its timestamp is 41 seconds later than the previous block
block at 832105 is empty, its timestamp is -41 seconds later than the previous block
block at 832272 is empty, its timestamp is 42 seconds later than the previous block
block at 832637 is empty, its timestamp is 34 seconds later than the previous block 
block at 832870 is empty, its timestamp is 10 seconds later than the previous block 
block at 833061 is empty, its timestamp is 7 seconds later than the previous block 
block at 834025 is empty, its timestamp is 39 seconds later than the previous block 
block at 834122 is empty, its timestamp is 17 seconds later than the previous block
block at 834457 is empty, its timestamp is 44 seconds later than the previous block
block at 834526 is empty, its timestamp is 82 seconds later than the previous block 
block at 834771 is empty, its timestamp is 12 seconds later than the previous block
block at 835151 is empty, its timestamp is 67 seconds later than the previous block 
block at 835442 is empty, its timestamp is 23 seconds later than the previous block
block at 835747 is empty, its timestamp is -35 seconds later than the previous block
block at 836797 is empty, its timestamp is 33 seconds later than the previous block
block at 837015 is empty, its timestamp is 21 seconds later than the previous block
block at 837951 is empty, its timestamp is 6 seconds later than the previous block 
block at 838906 is empty, its timestamp is 402 seconds later than the previous block
block at 838949 is empty, its timestamp is 12 seconds later than the previous block
block at 839000 is empty, its timestamp is 3 seconds later than the previous block
block at 839026 is empty, its timestamp is 31 seconds later than the previous block
block at 839463 is empty, its timestamp is 27 seconds later than the previous block 
block at 839870 is empty, its timestamp is -83 seconds later than the previous block
block at 839972 is empty, its timestamp is -99 seconds later than the previous block
block at 840421 is empty, its timestamp is 27 seconds later than the previous block
block at 841056 is empty, its timestamp is 12 seconds later than the previous block
block at 841116 is empty, its timestamp is 9 seconds later than the previous block
block at 841127 is empty, its timestamp is 17 seconds later than the previous block
block at 841355 is empty, its timestamp is 11 seconds later than the previous block
block at 841873 is empty, its timestamp is 11 seconds later than the previous block 
block at 842220 is empty, its timestamp is 7 seconds later than the previous block
block at 842634 is empty, its timestamp is 21 seconds later than the previous block
block at 842828 is empty, its timestamp is 25 seconds later than the previous block
block at 842977 is empty, its timestamp is 12 seconds later than the previous block
block at 843027 is empty, its timestamp is 22 seconds later than the previous block
block at 843183 is empty, its timestamp is 1 seconds later than the previous block
block at 843192 is empty, its timestamp is -15 seconds later than the previous block
block at 843474 is empty, its timestamp is 36 seconds later than the previous block
block at 843728 is empty, its timestamp is 27 seconds later than the previous block
block at 843924 is empty, its timestamp is 36 seconds later than the previous block
block at 844043 is empty, its timestamp is 10 seconds later than the previous block
block at 844144 is empty, its timestamp is 17 seconds later than the previous block
block at 844177 is empty, its timestamp is 10 seconds later than the previous block
block at 844208 is empty, its timestamp is 39 seconds later than the previous block
block at 844293 is empty, its timestamp is 23 seconds later than the previous block
block at 844504 is empty, its timestamp is 17 seconds later than the previous block
block at 844689 is empty, its timestamp is 12 seconds later than the previous block
block at 844854 is empty, its timestamp is 31 seconds later than the previous block
block at 845143 is empty, its timestamp is 48 seconds later than the previous block
block at 845161 is empty, its timestamp is 31 seconds later than the previous block
block at 845213 is empty, its timestamp is 0 seconds later than the previous block 
block at 845297 is empty, its timestamp is 17 seconds later than the previous block
block at 845399 is empty, its timestamp is 22 seconds later than the previous block
block at 845484 is empty, its timestamp is 40 seconds later than the previous block
block at 845548 is empty, its timestamp is 17 seconds later than the previous block 
block at 845948 is empty, its timestamp is 27 seconds later than the previous block
block at 846112 is empty, its timestamp is 37 seconds later than the previous block
block at 846153 is empty, its timestamp is 25 seconds later than the previous block
block at 846235 is empty, its timestamp is 38 seconds later than the previous block
block at 846337 is empty, its timestamp is 18 seconds later than the previous block 
block at 846402 is empty, its timestamp is 31 seconds later than the previous block
block at 846960 is empty, its timestamp is 31 seconds later than the previous block
block at 847083 is empty, its timestamp is 26 seconds later than the previous block
block at 847230 is empty, its timestamp is 2 seconds later than the previous block 
block at 847274 is empty, its timestamp is 23 seconds later than the previous block
block at 847280 is empty, its timestamp is 50 seconds later than the previous block
block at 847288 is empty, its timestamp is 43 seconds later than the previous block
block at 847423 is empty, its timestamp is 40 seconds later than the previous block
block at 847447 is empty, its timestamp is 11 seconds later than the previous block
block at 847487 is empty, its timestamp is 26 seconds later than the previous block
block at 847615 is empty, its timestamp is 10 seconds later than the previous block 
block at 847870 is empty, its timestamp is 17 seconds later than the previous block
block at 847904 is empty, its timestamp is 29 seconds later than the previous block
block at 847936 is empty, its timestamp is 21 seconds later than the previous block
block at 847993 is empty, its timestamp is 19 seconds later than the previous block
block at 848117 is empty, its timestamp is 44 seconds later than the previous block
block at 848480 is empty, its timestamp is 23 seconds later than the previous block
block at 848804 is empty, its timestamp is 36 seconds later than the previous block
block at 849398 is empty, its timestamp is 30 seconds later than the previous block
block at 850026 is empty, its timestamp is 7 seconds later than the previous block
block at 850472 is empty, its timestamp is 39 seconds later than the previous block
block at 852710 is empty, its timestamp is 39 seconds later than the previous block
block at 852846 is empty, its timestamp is 11 seconds later than the previous block
block at 853861 is empty, its timestamp is 7 seconds later than the previous block 
block at 853919 is empty, its timestamp is 33 seconds later than the previous block
block at 853930 is empty, its timestamp is 37 seconds later than the previous block 
block at 855027 is empty, its timestamp is 15 seconds later than the previous block
block at 855601 is empty, its timestamp is 36 seconds later than the previous block
block at 855858 is empty, its timestamp is 9 seconds later than the previous block  
block at 856547 is empty, its timestamp is 29 seconds later than the previous block
block at 856856 is empty, its timestamp is 1 seconds later than the previous block 
block at 857072 is empty, its timestamp is 31 seconds later than the previous block
block at 857104 is empty, its timestamp is 16 seconds later than the previous block
block at 857116 is empty, its timestamp is 22 seconds later than the previous block
block at 859194 is empty, its timestamp is 42 seconds later than the previous block
block at 859621 is empty, its timestamp is 17 seconds later than the previous block
block at 859659 is empty, its timestamp is 24 seconds later than the previous block
block at 859735 is empty, its timestamp is 8 seconds later than the previous block
block at 860183 is empty, its timestamp is 26 seconds later than the previous block
block at 860242 is empty, its timestamp is 26 seconds later than the previous block
block at 860667 is empty, its timestamp is 0 seconds later than the previous block
block at 860932 is empty, its timestamp is 20 seconds later than the previous block
block at 861645 is empty, its timestamp is 317 seconds later than the previous block
block at 861725 is empty, its timestamp is 332 seconds later than the previous block
block at 861849 is empty, its timestamp is 30 seconds later than the previous block
block at 862999 is empty, its timestamp is -38 seconds later than the previous block
block at 863118 is empty, its timestamp is 13 seconds later than the previous block
block at 863720 is empty, its timestamp is 11 seconds later than the previous block
block at 863779 is empty, its timestamp is -19 seconds later than the previous block
block at 863830 is empty, its timestamp is 24 seconds later than the previous block
block at 863854 is empty, its timestamp is -17 seconds later than the previous block
block at 864948 is empty, its timestamp is 5 seconds later than the previous block
block at 865065 is empty, its timestamp is 2 seconds later than the previous block
block at 866163 is empty, its timestamp is 25 seconds later than the previous block
block at 866230 is empty, its timestamp is 12 seconds later than the previous block
block at 866508 is empty, its timestamp is 36 seconds later than the previous block
block at 866579 is empty, its timestamp is 21 seconds later than the previous block
block at 866666 is empty, its timestamp is 34 seconds later than the previous block
block at 867274 is empty, its timestamp is 2 seconds later than the previous block
block at 867332 is empty, its timestamp is 15 seconds later than the previous block
block at 867552 is empty, its timestamp is 389 seconds later than the previous block
block at 867641 is empty, its timestamp is 18 seconds later than the previous block
block at 867644 is empty, its timestamp is 6 seconds later than the previous block
block at 867688 is empty, its timestamp is 45 seconds later than the previous block
block at 868250 is empty, its timestamp is 16 seconds later than the previous block
block at 868281 is empty, its timestamp is 7 seconds later than the previous block
block at 868291 is empty, its timestamp is 35 seconds later than the previous block
block at 868394 is empty, its timestamp is 28 seconds later than the previous block
block at 868504 is empty, its timestamp is -12 seconds later than the previous block
block at 868597 is empty, its timestamp is 5 seconds later than the previous block
block at 868664 is empty, its timestamp is 42 seconds later than the previous block
block at 868711 is empty, its timestamp is 17 seconds later than the previous block
block at 868784 is empty, its timestamp is 34 seconds later than the previous block
block at 869757 is empty, its timestamp is 44 seconds later than the previous block
block at 870057 is empty, its timestamp is 3 seconds later than the previous block
block at 871528 is empty, its timestamp is 12 seconds later than the previous block
block at 871642 is empty, its timestamp is 9 seconds later than the previous block
block at 871732 is empty, its timestamp is 6 seconds later than the previous block
block at 872266 is empty, its timestamp is 34 seconds later than the previous block
block at 873409 is empty, its timestamp is 11 seconds later than the previous block
block at 873480 is empty, its timestamp is 29 seconds later than the previous block
block at 873904 is empty, its timestamp is 31 seconds later than the previous block
block at 874540 is empty, its timestamp is 66 seconds later than the previous block
block at 875173 is empty, its timestamp is 14 seconds later than the previous block
block at 875322 is empty, its timestamp is 15 seconds later than the previous block
block at 875663 is empty, its timestamp is 35 seconds later than the previous block
block at 875997 is empty, its timestamp is 10 seconds later than the previous block
block at 876695 is empty, its timestamp is 4 seconds later than the previous block
block at 877397 is empty, its timestamp is 7 seconds later than the previous block
block at 878233 is empty, its timestamp is 10 seconds later than the previous block
block at 878609 is empty, its timestamp is 2 seconds later than the previous block
block at 878703 is empty, its timestamp is 12 seconds later than the previous block
block at 878848 is empty, its timestamp is 13 seconds later than the previous block
block at 879275 is empty, its timestamp is -41 seconds later than the previous block
block at 879461 is empty, its timestamp is 23 seconds later than the previous block
block at 879535 is empty, its timestamp is 8 seconds later than the previous block
block at 879706 is empty, its timestamp is 29 seconds later than the previous block
block at 880260 is empty, its timestamp is 20 seconds later than the previous block
block at 880496 is empty, its timestamp is 448 seconds later than the previous block
block at 881763 is empty, its timestamp is 4 seconds later than the previous block
block at 881869 is empty, its timestamp is 4 seconds later than the previous block
block at 882062 is empty, its timestamp is 21 seconds later than the previous block
block at 882764 is empty, its timestamp is 54 seconds later than the previous block
block at 883304 is empty, its timestamp is 3 seconds later than the previous block
block at 883793 is empty, its timestamp is 10 seconds later than the previous block
block at 886950 is empty, its timestamp is 22 seconds later than the previous block
block at 886964 is empty, its timestamp is 8 seconds later than the previous block
block at 887302 is empty, its timestamp is 63 seconds later than the previous block
block at 887605 is empty, its timestamp is 9 seconds later than the previous block
block at 887980 is empty, its timestamp is -14 seconds later than the previous block
block at 888416 is empty, its timestamp is 26 seconds later than the previous block
block at 889122 is empty, its timestamp is 17 seconds later than the previous block
block at 889406 is empty, its timestamp is 14 seconds later than the previous block
block at 889993 is empty, its timestamp is 19 seconds later than the previous block
block at 890103 is empty, its timestamp is 23 seconds later than the previous block
block at 890714 is empty, its timestamp is 13 seconds later than the previous block
block at 890847 is empty, its timestamp is 13 seconds later than the previous block
block at 891326 is empty, its timestamp is 23 seconds later than the previous block
block at 892159 is empty, its timestamp is 2 seconds later than the previous block
block at 892518 is empty, its timestamp is 10 seconds later than the previous block
block at 893822 is empty, its timestamp is 3 seconds later than the previous block
block at 893984 is empty, its timestamp is 10 seconds later than the previous block
block at 894052 is empty, its timestamp is 17 seconds later than the previous block
block at 894091 is empty, its timestamp is 11 seconds later than the previous block
block at 895175 is empty, its timestamp is 252 seconds later than the previous block
block at 895649 is empty, its timestamp is 13 seconds later than the previous block
block at 895978 is empty, its timestamp is 8 seconds later than the previous block
block at 896857 is empty, its timestamp is -38 seconds later than the previous block
block at 897689 is empty, its timestamp is 11 seconds later than the previous block
block at 898321 is empty, its timestamp is 12 seconds later than the previous block
block at 899717 is empty, its timestamp is 3 seconds later than the previous block
```

</details>

Running this command for the height range 350,000...450,000 (March 2015 - Jan 2017) returns *a lot* of empty blocks, with a much higher variance in timestamps (for instance block 410,214 has a timestamp almost 10 minutes *before* its parent).

EDIT: here are the results for this range. There were 2523 empty blocks during this period. 99 of them have a timestamp at least 10 minutes later than their parent. Results are available [here](https://gist.github.com/darosior/fb29bb79f0e841d5b8843372ab8f68b6), as there were too many lines to fit in a Delving post.

The highest difference in timestamp is from empty block 375,153 that is close an hour after its parent!

-------------------------

AntoineP | 2026-02-05 14:44:30 UTC | #5

@0xB10C added an [empty blocks dashboard](https://mainnet.observer/charts/blocks-empty/) to mainnet.observer. It also links to this [2017 post](https://www.bitmex.com/blog/empty-block-data-by-mining-pool) from Bitmex Research, which surfaces another hypothesis for the "long" empty blocks circa 2015-2016: a covert ASIC boost optimisation from Bitmain.

-------------------------

