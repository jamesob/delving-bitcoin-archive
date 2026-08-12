# Reducing Bitcoin Full Node Storage Without Consensus Changes

BrokenMachine | 2026-08-09 18:34:24 UTC | #1

I’ve been mulling over the blockchain size “problem” for a while now, particularly with an eye toward being able to continue running a full node comfortably well into the future. I wanted to see whether there was a useful middle ground between the “disk space is cheap, who cares?” position and the much more aggressive approach of changing consensus to constrain growth.

That led me to build a customized implementation forked directly from Bitcoin Core. What started as a personal experiment ended up producing results that I thought were interesting enough to share, so I’ve pushed the source to GitHub rather than keeping it to myself.

I’m posting it here specifically because I’d like it reviewed by people with more experience in Bitcoin Core internals than I have. I’ve tested it fairly extensively and tried to be conservative about what I changed, but I’m under no illusion that my own testing is a substitute for independent review. If I’ve made a bad assumption, missed an edge case, or done something stupid, I’d much rather have it pointed out now.

BitcoinRocks v31.1.1 — Bitcoin Core v31.1 with RocksDB, compressed block storage, and a few forward-ported optimizations

I’ve published the first source release of **BitcoinRocks v31.1.1**, based directly on Bitcoin Core v31.1:

https://github.com/BitcoinRocks-Core/BitcoinRocks

BitcoinRocks is **not a separate chain or cryptocurrency**. It follows Bitcoin consensus and is intended as an experimental alternative Bitcoin full-node implementation focused primarily on storage/database behavior and node-side performance.

The main changes in v31.1.1 are:

* **LevelDB → RocksDB** for chainstate and database-backed indexes, with workload-specific tuning and hardware-aware automatic cache allocation. Explicit `-dbcache` remains available as an override.
* **Compressed `blk*.dat` records** using Zstandard. Each block is compressed independently, with transparent support for legacy raw records and automatic raw fallback when compression is not worthwhile.
* **Parallel prevout fetching during `ConnectBlock`**, backported from the post-v31.1 Bitcoin Core development work. BitcoinRocks exposes this through `-prevoutfetchthreads` (`8` default, `16` maximum, `0` disables it).
* **User-selectable relay-policy profiles** (`core`, `conservative`, and `strict`). These alter local relay/mempool policy only and do **not** change consensus.
* Depends builds pin the intended **RocksDB, LZ4, and Zstandard** dependencies.

Before tagging the release, I also performed an exhaustive chain cross-check against Bitcoin Core: every main-chain block from Genesis through a snapshot height above 961k was independently compared, including serialized blocks/headers, transaction ordering, sampled raw transactions, and Merkle proofs. The test completed with **zero mismatches**, and both nodes subsequently converged on and byte-verified the same live chain tip.

Storage testing has also been encouraging. On the dataset used for the published comparison, compressed BitcoinRocks block files saved roughly **171 GiB** relative to an uncompressed Bitcoin Core node, while the RocksDB-backed optional indexes were about **9.2 GiB smaller overall**. The repository includes the raw comparison methodology and results rather than just the headline figures.

This is currently a **source-only release**. I’m particularly interested in review of the RocksDB integration, block-record format, cache/tuning decisions, and anything I may have overlooked in the interaction between compressed block storage, reindexing, pruning, and indexes.

The relevant implementation notes and storage results are under `doc/` in the repository.

-------------------------

reardencode | 2026-08-11 17:14:24 UTC | #2

Cool. How big is your total store size after sync?

This has been a focus of my work on rbitcoin as well. Storage may be cheap, but RAM is expensive an many operations require loading parts of the storage into RAM.

-------------------------

BrokenMachine | 2026-08-11 21:50:46 UTC | #3

I just reran the storage accounting against the fully synced node today, at approximately **block 962,055**. One block arrived while the audit was running, so the record count moved underneath the snapshot slightly.

The **entire BitcoinRocks datadir is currently 769.904 GB / 717.029 GiB apparent size** (**717.218 GiB actually allocated on disk**).

That includes all three optional indexes I run:

* `txindex`: **58.391 GiB**
* `blockfilterindex`: **12.228 GiB**
* `coinstatsindex`: **0.089 GiB**

Those optional indexes total **70.707 GiB**, so without them the base full-node datadir is about **693.983 GB / 646.322 GiB**.

The main components are:

* compressed `blk*.dat`: **537.005 GiB**
* `rev*.dat`: **98.763 GiB**
* `chainstate`: **10.396 GiB**
* `blocks/index`: **0.108 GiB**

The block audit currently represents **708.572 GiB of logical raw block payload stored in 536.988 GiB**, saving **171.584 GiB** on block payload, or **24.215%**.

If I add those saved bytes back to this exact datadir, the equivalent dataset with raw block records would be **888.613 GiB**, so compression is currently reducing the whole datadir by **19.309%**, even with undo data, chainstate, and optional indexes left untouched.

On the RAM side, some of that is deliberate. I made a conscious effort to safely make use of up to roughly **25% of available system RAM during IBD** in pursuit of faster synchronization from Genesis. One recurring complaint from end users is essentially, *“I don’t want to wait a week for Bitcoin to sync,”* so reducing IBD time was one of the implementation goals rather than minimizing memory use at all costs.

On this particular machine, a full synchronization from Genesis reached mainnet in roughly **30 hours**. The system remained responsive throughout IBD and was still perfectly usable for other work. Once IBD completed and the node settled into steady-state operation, its resource usage became practically unnoticeable to me, despite this being a machine I use fairly heavily.

That said, I would not treat that result as a definitive Bitcoin Core vs. BitcoinRocks performance comparison. A proper stock-Core-versus-BitcoinRocks benchmark would need to control hardware, configuration, cache allocation, enabled indexes, network conditions, and starting state. Ideally, I would also like to see those measurements independently reproduced by third parties before making strong claims about relative IBD speed, RAM use, or steady-state runtime performance.

For completeness, here is the terminal output from the accounting script:

[details="Full BitcoinRocks storage audit"]
```text
Data directory:               BitcoinRocks v31.1.1 Data Directory
Software profile:             BitcoinRocks v31.1.1
Database backend:             RocksDB
Block storage format:         ZSTD-compressed flat-file block records with raw fallback

Scanned 4,313/4,313 blk files; records=962,057

Block-file XOR:                    enabled
blk files:                         4,313
block records:                     962,057
ZSTD-compressed block records:     857,601
Raw fallback block records:        104,456
Logical raw block payload:         760,823,248,710 B    760.823 GB    708.572 GiB
Stored block payload:              576,586,602,905 B    576.587 GB    536.988 GiB
Block payload saved:               184,236,645,805 B    184.237 GB    171.584 GiB
Payload reduction:                 24.215%
Eight-byte record headers:             7,696,456 B      0.008 GB      0.007 GiB
blk apparent file size:            576,604,250,024 B    576.604 GB    537.005 GiB
blk allocated disk blocks:         576,620,773,376 B    576.621 GB    537.020 GiB
Current blk prealloc/slack:             9,950,663 B      0.010 GB      0.009 GiB

ENTIRE DATADIR                     769,903,870,039 B    769.904 GB    717.029 GiB  allocated=717.218 GiB  files=10,645
blocks/ total                      682,765,926,391 B    682.766 GB    635.875 GiB  allocated=635.926 GiB  files=8,641
  blk*.dat                         576,604,250,024 B    576.604 GB    537.005 GiB  allocated=537.020 GiB  files=4,313
  rev*.dat                         106,046,158,393 B    106.046 GB     98.763 GiB  allocated=98.788 GiB   files=4,313
  blocks/index                         115,517,966 B      0.116 GB      0.108 GiB  allocated=0.118 GiB    files=13
chainstate                          11,162,232,953 B     11.162 GB     10.396 GiB  allocated=10.423 GiB   files=194
indexes/ total                      75,920,971,074 B     75.921 GB     70.707 GiB  allocated=70.818 GiB   files=1,798
  indexes/blockfilter               13,129,328,599 B     13.129 GB     12.228 GiB  allocated=12.269 GiB   files=792
  indexes/coinstatsindex                95,041,703 B      0.095 GB      0.089 GiB  allocated=0.101 GiB    files=14
  indexes/txindex                   62,696,600,772 B     62.697 GB     58.391 GiB  allocated=58.449 GiB   files=992
```

The script's own estimate of the disk-space cost without block compression, which very closely matches what my Bitcoin Core v27.0 node with the same three optional indexes occupies on disk:

```text
Current apparent datadir:          769,903,870,039 B    769.904 GB    717.029 GiB
Add back blk savings:              184,236,645,805 B    184.237 GB    171.584 GiB
Same datadir with raw blks:        954,140,515,844 B    954.141 GB    888.613 GiB
Net whole-datadir reduction:       19.309%
Everything except blk files:       193,299,620,015 B    193.300 GB    180.024 GiB
```


\*edited for formatting

[/details]

-------------------------

