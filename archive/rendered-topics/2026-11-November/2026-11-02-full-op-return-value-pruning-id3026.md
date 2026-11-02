# Full OP_RETURN value pruning

bkeroack | 2026-11-02 20:30:37 UTC | #1

satd is the first Bitcoin node implementation to allow optional full OP_RETURN value pruning. It is controlled by these options:

* `-pruneopreturn=N` (N is the number of blocks behind tip, 288 minimum to match NODE_NETWORK_LIMITED guarantee)
* `-pruneopreturnsize=X` (X is the value size in bytes, greater than or equal to X will be pruned)
* `-restoreopreturns` (stop pruning and restore all pruned blocks; mutually-exclusive with above options)

Why do OP_RETURN value pruning? The recent BIP-110 controversy made it apparent that a certain proportion of node runners care deeply about OP_RETURN value sizes and what they see as spam on the blockchain. My personal view is that an objective analysis of mainnet shows that OP_RETURN value footprint is negligible currently. That said, the philosophical argument that non-financial data is spam and doesn't belong on the chain is real and resonates with some people. Since value pruning can be implemented without breaking consensus or most full node functionality, I believe it's better to allow this flexibility in node software to allow operators to express their preferences. If all major implementations adopt this type of flexibility hopefully we can avoid future distracting controversies.

When you combine pruning with the [Transaction Filter Policy language](https://delvingbitcoin.org/t/cost-bounded-transaction-filtering-language/2966) you can also prevent matching transactions from being relayed or added to block templates. Use a conditional like sum `outputs (out.op_return_size) > 80` in your filter rule (see the [manual](https://epochbtc.github.io/satd/policy.html) for details). This addresses the "I don't want my node assisting or storing large OP_RETURN values" concern. 

The implementation punches holes in the on-disk block files, redacting values that match the threshold. This uses the FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE (Linux) and F_PUNCHHOLE (MacOS) syscalls to redact values in place.

There are some consequences of pruning OP_RETURN values:

1. Your node advertises NODE_NETWORK_LIMITED just like a standard pruned node, which means you won't serve blocks to most peers (even if you have them intact, but see below for mitigation).
2. The disk usage will go **up**, not down. Hole-punching only returns storage to the system when it meets or exceeds the block size, typically 4kb. The vast majority of OP_RETURN values are less than this, so very little space reclaiming will happen. On top of that, every pruned block requires extra index entries to preserve txids and allow reindexing. So net disk usage tends to go UP with value pruning, not down.
3. Your node can no longer serve raw transaction bytes when values are pruned. Those JSON-RPC requests are refused with an error (for example, "-1 Block not available (pruned data)"). Blocks without pruned values are returned normally. `getblock` at verbosity 1 returns an additional custom field, ex: `"opreturn_stripped": { "transactions": 2, "outputs": 2, "bytes": 18432 }`. Verbosity 2 returns original txid, but no hex and the additional field `"opreturn_stripped": true.`
4. SSDs make it essentially impossible to securely delete anything, so the pruned values are likely forensically recoverable for a period of time.
5. Snapshots on CoW filesystems like ZFS, btrfs will retain the bytes despite pruning.

satd supports a special option: `-pruneopreturnpeers=intact` (default: "pruned") which causes the node to advertise NODE_INTACT_BLOCKS (service bit 28, within bits 24-31 that are reserved by Core for temporary experiments). This allows satd to serve whatever intact non-pruned blocks it has to peers (pruned blocks return `notfound`). Other satd 0.7.0+ peers will recognize this service bit and will request blocks even though the node is NODE_NETWORK_LIMITED. Core peers will likely continue to treat the peer as fully pruned and will not request blocks.

OP_RETURN pruning is reversible. Use the `-restoreopreturns` option to trigger the node to re-download all pruned blocks as background restore process (node stays NODE_NETWORK_LIMITED during this process). Once complete, the node switches back to NODE_NETWORK and fully serves peers blocks as before. Then the option can be dropped and the node works normally from that point forward.

You can download and install [satd 0.7.0 here](https://github.com/epochbtc/satd/releases/tag/v0.7.0). The manual talks about OP_RETURN pruning in the [Disk Footprint chapter](https://epochbtc.github.io/satd/disk-footprint.html#op_return-data-pruning). Any feedback or questions are welcome!

-------------------------

