# Pruning non-transaction data instead of pruning whole blocks?

cooltexture | 2025-01-14 10:43:44 UTC | #1

Currently bitcoind's pruning option, simply removes all block data before a certain block height, only keeping the block headers, as far as i understand it.
Would it be possible to have a new pruning mode where all non-transaction data, like data pushes in unspendable outputs, is removed? I wonder if that would actually save much data, but i know some protocols like ordinals, have single-handedly filled entire blocks, so that would be a few megabytes at least. Would pruning this data have any drawbacks compared to regular pruned nodes or SPV clients?
As far as i know, protocols like ordinals allow, storing arbitrary data into the blockchain, which might also contains content that some node owners might not want to store on their hard drive, like doxxing information or pornography. 
Pruning this kind of data could also help, as a node operator would then not have to store these kinds of data.

-------------------------

sjors | 2025-01-14 13:37:09 UTC | #2

Removing parts of a block creates a fragmentation issue. Additionally there's not much utility in keeping incomplete blocks around, as you can't serve those to peers.

There's one exception though: you _can_ serve blocks without witness data. The following pull request could be revived to implement that:

https://github.com/bitcoin/bitcoin/pull/27050

> content that some node owners might not want to store on their hard drive

Something you don't want on your harddrive could be embedded in the non-witness part of a block. E.g. Counterparty, used more recently with STAMPS, puts data in a bare multisig `scriptPubKey`. Even pruning won't get this removed from your hard disk, because it's part of the UTXO set, which is stored in the `chainstate` directory. And these outputs may never get spent.

A project like Utreexo _would_ remove them from your hard disk (if that is your goal).

-------------------------

GaloisField2718 | 2025-01-15 23:35:07 UTC | #3

Hi, 
I understand in the way that can we prune the witness data as they are not "required" part of the transaction? Once the validity is proven are we able to prune it from the node? 
I feel that your question @cooltexture was about this.

-------------------------

GaloisField2718 | 2025-01-15 23:37:40 UTC | #4

Ah my bad it was in substance contained into your link sjors: 
> This PR does two things when running in prune mode: (a) assume witness merkle roots to be valid for assumed-valid blocks and (b) request assumed-valid blocks without `MSG_WITNESS_FLAG`.

>In theory this is a good idea, because witnesses are not validated (i.e. they are assumed valid up to a certain block `-assumevalid`) and get pruned anyway when running in prune mode, so not downloading them (for assumed-valid blocks) reduces the bandwidth that is needed for IBD (don't have any numbers on this yet).

> One downside is that nodes serving blocks without witnesses can't serve them directly from disk. They have to un-serialize and re-serialize without the witnesses before sending them."

-------------------------

