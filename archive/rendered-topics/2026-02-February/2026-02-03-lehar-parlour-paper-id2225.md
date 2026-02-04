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

