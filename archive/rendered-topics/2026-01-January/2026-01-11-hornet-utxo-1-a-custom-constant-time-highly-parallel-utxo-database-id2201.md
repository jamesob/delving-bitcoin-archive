# Hornet UTXO(1): A custom, constant-time, highly parallel UTXO database

tobysharp | 2026-02-27 10:52:09 UTC | #1

Hi all,

I have developed a new UTXO database, custom designed for maximizing throughput on Bitcoin consensus validation. I call it Hornet UTXO(1) because the queries are constant-time. 

This is a new component of Hornet Node, my experimental Bitcoin client focused on declarative consensus specification and high performance. I intend to do a full write-up but I'm posting preliminary results here to get feedback from the community first. 

The database has been developed from scratch in modern C++. It has been designed for maximum parallelism so it's highly concurrent and **lock-free**. 

On my 32-core dev workstation, these are like-for-like results for re-validating the whole mainnet chain without script or signature validation: 

* Bitcoin Core: **167 minutes** \
`bitcoind -datadir="/nvme1/bitcoin/" -reindex-chainstate -dbcache=24000 -connect=0 -assumevalid=0000000000000000000019426fb9e93a5cfa7695f271d928beeddf39a1ef3f3f`
* Hornet UTXO(1): **15 minutes** 

In this benchmark, both processes perform all structural and contextual validation, build the UTXO database, and validate all tx inputs correspond to unspent outputs. 

The core operations of Append, Query, and Fetch can all be executed concurrently and lock-free. Blocks can be submitted for validation out of order as they arrive, and will be processed concurrently, with data dependencies automatically resolved. 

Entries in the database index are height-tagged so that state can be rewound to a fork point for reorgs without storing additional undo state. This also allows queries to be issued against a specific past chain state. 

The serial dependency on a previous block's transactions is minimized by:

* Allowing non-dependent work to progress concurrently;
* Performing pre-emptive Append operations (that can be unwound in a failure case);
* Allowing partial queries and fetches against the existing data. 

Disk access is optimized via asynchronous, high queue-depth requests. Memory is optimized via compact, cache-friendly data structures and fixed-size allocations. 

The implementation is succinct at only 3,000 lines of C++23 with no external dependencies. There are no hash maps involved as sorted arrays and LSM trees are preferred. 

For those who missed it, you can read about the broader Hornet Node project here: 

T. Sharp (2025). *Hornet Node: A Declarative, Executable Specification of Bitcoin Consensus.* [HTML](https://hornetnode.org/paper.html) // [PDF](https://arxiv.org/pdf/2509.15754) 

(Thanks to the people who got in touch to encourage me with this project. I really appreciate it.)

I would be interested to hear feedback, questions, or about related work that I’m probably unaware of. I can then make sure it all gets addressed in the formal write-up.

This is currently an unfunded personal project developed in spare time. If you would like to support continued development, you can donate here: `33Y6TCLKvgjgk69CEAmDPijgXeTaXp8hYd`. 

If you're in industry and you're interested in faster IBD validation times to bring nodes online, please reach out. 

Best wishes, 
T♯

-------------------------

jsarenik | 2026-01-17 18:04:48 UTC | #2

Reading about it at https://hornetnode.org/overview.html but can’t find any source code (repository) yet.

Another note is that I see bitcoin.pdf being mentioned but I couldn't find word “mempool” mentioned Hornet-wise yet. Satoshi’s paper says in section 5. Network that “1) New transactions are broadcast to all nodes.” (which is another way of writing about what's known today as mempool).

Thanks for sharing!

PS: For loading Bitcoin Core nodes with current mempool (dumped every 15 seconds for months now) on fresh start without any locally persisted `mempool.dat` I use https://anyone.eu.org/mymempool-log.txt and feel free to get the data there, too. HTH

-------------------------

setavenger | 2026-01-18 16:11:54 UTC | #3

Reading through the Hornet Node page, it seems like it's an entirely separate Node implementation. I find that very interesting and would love to read more about it and get my hands on the source code. I was also not able to find it after a brief search. From that a couple questions arise:

- How was the comparison structured? Was it Bitcoin Core vXX.X vs Hornet Node with Hornet UTXO(1) as a DB engine? Or were the 15min achieved by somehow porting the DB to Bitcoin Core?
- Have you made the same or a similar comparison to libbitcoin? The validation speed on that implementation was also quite astounding and was discussed on Delving Bitcoin as well.
- Generally speaking can the DB be somehow used for other purposes like an indexer or similar use cases, or is it constrained as a db engine for Hornet Node?

If I can get my hands on the source code I can give some more feedback and more qualified questions.

Cheers!

-------------------------

tobysharp | 2026-02-27 10:50:56 UTC | #4

> Reading through the Hornet Node page, it seems like it’s an entirely separate Node implementation.

That's correct, although the focus is on spec-based consensus, validation, and IBD. There is so far no mempool behavior, although maybe that will be added later.

> I find that very interesting and would love to read more about it 

The most complete documentation is here:

[Hornet Node Overview](https://hornetnode.org/overview.html) \
[Hornet Node Executable Declarative Specification](https://hornetnode.org/paper.html)

The Hornet UTXO(1) database is the latest addition to this workstream.

> and get my hands on the source code.

I'm not widely sharing the code yet, although I'm planning to do so once this IBD phase is complete. I'd be happy to give a presentation or demo too if you would like to connect via email. 

> How was the comparison structured? Was it Bitcoin Core vXX.X vs Hornet Node with Hornet UTXO(1) as a DB engine? Or were the 15min achieved by somehow porting the DB to Bitcoin Core?

Bitcoin Core v30.0 vs Hornet Node, yes. Bitcoin Core was run with the -assume-valid option to skip signature and script validation, so that the main cost is the UTXO database construction and querying. 

> Have you made the same or a similar comparison to libbitcoin? The validation speed on that implementation was also quite astounding and was discussed on Delving Bitcoin as well.

I haven't but I agree that would be interesting. Is that something you can assist with? My expectation is that libbitcoin beats Core but Hornet wins due to parallelism. 

> Generally speaking can the DB be somehow used for other purposes like an indexer or similar use cases, or is it constrained as a db engine for Hornet Node?

It could be extended to index transactions and addresses etc. if desired. It's just low level code so anything's possible. My initial priority though was just to discover what performance I could get for initial block download.

Thanks,
T#

-------------------------

tobysharp | 2026-01-22 04:52:34 UTC | #5

> Another note is that I see bitcoin.pdf being mentioned but I couldn’t find word “mempool” mentioned Hornet-wise yet.

Indeed, so far Hornet is focused on the consensus aspects of the protocol: creating a pure, declarative, executable specification of consensus rules; and demonstrating a novel engineering design for fast and memory-efficient validation and sync.

Hornet has a layered design where consensus is separated from mempool and policy.

Thanks for your interest,
T#

-------------------------

optout | 2026-01-22 06:33:21 UTC | #6

[quote="tobysharp, post:1, topic:2201"]
On my 32-core dev workstation, these are like-for-like results for re-validating the whole mainnet chain without script or signature validation:

* Bitcoin Core: **167 minutes**
* Hornet UTXO(1): **15 minutes**

[/quote]

Impressive results!
Could you also share the details on the memory/storage:
- the RAM/swap/storage of the HW (hdd/ssd/nvme…)
- is the Hornet UTXO DB in-memory, on-disk, disk-backed memory, cached disk, etc.
thx

-------------------------

tobysharp | 2026-01-23 05:44:53 UTC | #7

> the RAM/swap/storage of the HW (hdd/ssd/nvme…)

128 GB RAM, 1 TB NVMe.

> is the Hornet UTXO DB in-memory, on-disk, disk-backed memory, cached disk, etc. thx

The pk script, amount, and funding details of transaction outputs are flushed to an append-only disk file as the chain grows. When these are required for validation, they are fetched using high queue depth async reads (io_uring on linux).

Meanwhile, the index entry containing the txid, output index, height and some bit flags are held in memory (48 bytes each, one entry for each unspent output). I've designed this so that the index could instead be stored on disk, at the cost of some query performance, but for now the server-class hardware seemed the more interesting case than the consumer-class hardware.

-------------------------

stickies-v | 2026-01-28 17:01:22 UTC | #8

Thanks for your continued work on this, Toby! Looking forward to the day you’re releasing the code so I can have a better look. This looks promising.

Is my understanding correct that you have on-disk TXO database (since it’s append-only, I suspect it contains both spent and unspent TXOs?) and an in-memory UTXO index (which as per your latest comment could also be migrated to being on-disk)? Roughly how big are the database and the index, at the current chaintip (or whichever height you’re using)? I suppose pruning spent TXOs from the disk database is not something you plan on supporting, given the focus on server-class hardware and related architecture choices?

-------------------------

tobysharp | 2026-02-27 10:51:23 UTC | #9

Hi stickies-v. Thanks for reading the post so carefully and for your questions :) 

> Is my understanding correct that you have on-disk TXO database (since it’s append-only, I suspect it contains both spent and unspent TXOs?) and an in-memory UTXO index (which as per your latest comment could also be migrated to being on-disk)?

Yes, that's exactly correct.

> Roughly how big are the database and the index, at the current chaintip (or whichever height you’re using)?

That's a great question, I should have included this info. 

The index is 48 bytes per UTXO, and this number looks unlikely to change. Even if I trimmed the structure to 44 bytes of data, I think 48 bytes would likely still be needed for nice alignment. At 175M UTXOs (an approximate count) that's 7.8 GiB which is fine atm for my hardware. 

If the UTXO count were to substantially increase, then I would be looking to store the index on disk, and pay one I/O page read per query, and hide all the additional latency in IBD with parallelism. I think this would put me in a good spot where decent RAM is not critical for the performance. 

The on-disk TXO database currently appears to be 134 GiB. However, I haven't yet written the code to withhold unspendable outputs like OP_RETURN: currently these are included in that total. I also haven't made much effort to compress this storage yet, although it's also not clear that such efforts would be necessary or fruitful.

As you correctly point out, this ends up largely being spent transaction data that no longer needs to be fetched for validating future blocks. 

> I suppose pruning spent TXOs from the disk database is not something you plan on supporting, given the focus on server-class hardware and related architecture choices?

It's something I've considered. I think the sensible plan there would be to blast through the whole IBD to tip, and then create a new table on disk that only includes the unspent outputs to that point: essentially a compaction step. This could be optional, or it could be also be a process that's user-invoked when the user wants to defrag storage.

So the options exist to improve the storage efficiency for hardware with smaller resources. However, my assumption was that it's the workstation- / server-class hardware that particularly benefits from this parallel approach. Do you agree?

Thanks again,
T#

-------------------------

shrec | 2026-02-26 16:59:32 UTC | #10

Clang 21.1.0, i7-11700, single core, pinned:

![image|690x465](upload://1yYBsau9JLB0ifGIan7Ok5hJEqU.png)

-------------------------

shrec | 2026-02-26 17:02:07 UTC | #11

this benchmark is from my dev machine that is already loaded by dev tools.

-------------------------

system | 2026-02-27 06:05:49 UTC | #12



-------------------------

system | 2026-02-27 10:52:09 UTC | #13



-------------------------

