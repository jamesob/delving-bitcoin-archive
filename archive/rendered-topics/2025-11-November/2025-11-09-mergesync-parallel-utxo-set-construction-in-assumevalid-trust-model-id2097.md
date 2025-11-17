# MergeSync: Parallel UTXO-set construction in assumevalid trust model

ffwd | 2025-11-09 18:16:24 UTC | #1

As the Bitcoin block chain grows, the time required to set up a new node from scratch also increases.

At block height 910_000, just over 16 years into the Bitcoin experiment, more than 1.2 billion on-chain transactions have been recorded. Including witness data, this amounts to almost **700GiB** of data, enough to fill roughly 1000 CDs. Those transactions created over **3 billion UTXOs**, of which about **95%** have either already been spent or are provably unspendable. One goal of syncing a new node from scratch is therefore to determine *which* UTXOs survive valid transfers, so that new blocks can be checked quickly against the current UTXO-set.

Current implementations locate a valid block header chain and then download and validate each block sequentially. This process is slow; depending on CPU, RAM, disk, and network speed it can take anywhere from a few hours to several days.

Numerous ideas for improving Bitcoin scalability, specifically initial block download (IBD) speed, have been proposed over the years - some have even been implemented.

Earlier this year an observation was made that current bottleneck is not CPU speed but I/O. Because we process blocks in sequence, we constantly write, read, delete entries in the `chainstate` database on disk. Using an SSD for the `chainstate` folder and increasing `-dbcache` can accelerate sync considerably, but this remains a sequential process.
Some researchers explored a new approach called **SwiftSync**, originally published by Ruben Somsen. Reading about SwiftSync provides useful background, and inspired our work, but it is not required to understand the method described here.

---

## Algorithm

We propose a map/reduce approach that lets a `-assumevalid` trusting node speed up UTXO-set construction using a parallel data pipeline.

Let’s start with a simpler question: What is the cardinality of the UTXO-set at height 922_527?

block hash:`00000000000000000000a08e6723dc9eac9a108d50d05eb7c94885f3f10696bc`

0. Assume a valid copy of the block chain is available locally.
1. **Shard** all block data so that each of the **N** available cores processes roughly the same amount of data. For every transaction, **extract** every *OutPoint* (`txid:vout`) from both inputs and outputs. Each core appends its extracted OutPoints to a CSV file with two columns (`txid`, `vout`). Provably unspendable outputs and coin-base inputs are skipped.
2. **Sort** each CSV file in lexical order (by line) in parallel.
3. Perform a **single pass** over each sorted file to eliminate duplicate lines; because the files are presorted, duplicates are guaranteed to be adjacent.
4. **Recursively** merge two or more surviving CSV files (subject to available RAM), then repeat steps 2-4 until only one file remains.
5. After the final iteration, the line count of the remaining file is the answer to the question above.

To obtain a full UTXO-set in a usable format, additional metadata (such as amounts, output scripts, creation height, etc.) can be added as extra columns to the CSV files. Keep in mind that plain HEX/CSV is not the most efficient encoding, it only served as simple explanation of the algorithm. External sorting and other techniques can be employed to reduce memory usage on constrained systems. Implementation is left as an exercise for the interested reader.

### Known security tradeoffs

* No signatures are processed or verified. This is unsafe unless another trustless node has performed full sequential validation, thereby confirming that the `-assumevalid` block hash is indeed user-valid.

### Open questions

* Are there additional attack vectors we have not considered?

---

*MergeSync* demonstrates that, under an assumevalid trust model, parallel processing and simple set operations can reduce wall clock for UTXO-set construction. Further work is needed to quantify performance gains on various hardware and to formalize security assumptions.

-------------------------

murch | 2025-11-17 19:14:27 UTC | #2

Hey, thanks for sharing your idea. It seems to me that implementing this would suffer from the same problem that you mention in your motivation: we would need to write out all UTXO data and additionally all inputs’ outpoints. It’s not clear to me why you would expect this to reduce the I/O compared to the Bitcoin Core’s current approach.
Compared to SwiftSync which uses the hints to compress the checking of all spent UTXOs to a single accumulator (which can also be parallelized), this feels like a regression.
Perhaps I’m overlooking something. Could you clarify what the novel contribution here is?

-------------------------

