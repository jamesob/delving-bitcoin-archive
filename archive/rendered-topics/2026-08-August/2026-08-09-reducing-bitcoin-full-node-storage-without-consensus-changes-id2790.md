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

