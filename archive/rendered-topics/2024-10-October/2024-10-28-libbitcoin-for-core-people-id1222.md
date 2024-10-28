# Libbitcoin for Core people

AntoineP | 2024-10-28 19:09:55 UTC | #1

Recently Eric Voskuil [shared](https://x.com/evoskuil/status/1847684550966599894) a benchmark of doing IBD with Libbitcoin versus Bitcoin Core, showing that [Libbitcoin could perform IBD 15x faster than Core](https://x.com/evoskuil/status/1848015101233672628) with `-assumevalid`.

Libbitcoin's approach is very different from Bitcoin Core. This writeup intends to summarize how Libbitcoin functions for someone familiar with Bitcoin Core's approach. This discusses the behaviour of upcoming, not yet released, version 4 of Libbitcoin.

Libbitcoin is event-based. It will kick-off multiple asynchronous tasks (it uses [Boost ASIO](https://www.boost.org/doc/libs/1_86_0/doc/html/boost_asio/overview/core/async.html)). As you'll see later it's important as it enables it to take full advantage of its approach of splitting the various validation steps into checks which require strict, partial or no ordering.

Libbitcoin's database is *conceptually* relational but the storage is abstracted out so different backends can be implemented. Conceptually the Libbitcoin node's database has a table for headers, transactions, transaction inputs and transaction outputs. It maintains an index from confirmation to header, header to transactions, transaction to header (only transactions confirmed in the best chain) and transaction output to its spending transaction.

To sync, Libbitcoin will spin up a hundred (by default) connections (more on how it handles connections later) and will redundantly request headers from all its peers. Once it gets a chain of headers which is "current", as defined by being less than 1h older than the system time (by default), it will start querying blocks from its peers. Blocks queries are spread across all peers.

Libbitcoin breaks down block validation into steps which need partial ordering and those that require strict ordering. Some checks (for instance block size, or transaction `nLockTime`) don't require any ordering. Checking scripts only require that you have seen the transactions that are being spent (partial ordering). Validating that the inputs actually exist requires that you went through all the blocks in order and checked the corresponding outputs were created and never spent (strict ordering). Libbitcoin will perform these checks, as well as the actual block downloading, concurrently.

When a block is downloaded, checks which don't require ordering are performed and then the transactions are stored. Of course this is fine to do DoS-wise because it only considers transactions which are part of the chain with the most PoW. Concurrently the validation (see below for terminology) thread will perform for all new transactions the check which only require partial ordering. That is, all checks (including script) but whether inputs exist (and also relative timelocks, interestingly). The confirmability (see terminology below) thread will, still in parallel, check for a range of validated blocks whether all inputs spent by these transactions exist and are yet unspent. If they are it will for each transaction insert an entry in the transaction -> header index, which conceptually "marks the block as confirmed". So the steps are 1) download 2) validation 3) confirmability but those happen concurrently for different ranges of blocks.

Note that for blocks below the milestone (see below for terminology), Libbitcoin will skip transaction validation (besides checking they were properly committed, i.e. no malleation of either transactions or witnesses). This is in my opinion a similar threat model to Bitcoin Core's `-assumevalid`.

Reorg'ing is straightforward. To unconfirm a block the node just wipes the blocks' transactions from the transaction -> header index.

This means that pruning can technically be implemented by removing input and output data for historical spent outputs. In this case the node wouldn't be able to reorg past the pruning point without requesting peers for missing transactions. However the store is at the moment append-only and supporting this is not a priority for the project as Eric does not consider pruning an important feature.

The transaction table is also used to persist unconfirmed transactions. They can occur by being announced by a peer or through a reorg. This is not implemented yet. When announced by a peer, the node would require a transaction has at a minimum feerate. For conflicts the node would require the transaction pays at least the minimum feerate for itself plus for the whole graph it conflicts with (i.e. all past conflicts and their descendants). In effect this requires the absolute fee paid to increase, pretty much like Core does.

Now peers/connections management. Libbitcoin will by default connect to a hundred outbound peers. They intend to make this number dynamic in the future (so it throttles back after IBD). When establishing a connection to a peer they try to establish 5 concurrently and only keep the one which finishes handshake first. For syncing headers they will redundantly request headers across all their connections. They do not monitor them for speed. Block requests are spread out across peers. The speed of responses to the block requests is monitored. Libbitcoin will compute a standard deviation across the set of download rate for each connection. Slow peers, as defined by a configurable negative deviation from the SD (default: 1.5x the SD), are dropped. Stalled peers are also dropped. DoS protection for requests made to the Libbitcoin node isn't implemented yet, but Eric expects it'll mainly consist of rate limiting requests.

Another important point when considering benchmarks between Core and Libbitcoin: it currently uses a 3 years old libsecp and does not have native SHA256 acceleration.

## Terminology

Libbitcoin uses similar words for similar but different things as Bitcoin Core does. This intends to clarify the similarities and differences.

- **Milestone**: equivalent to `-assumevalid` (but skips all checks, not only the script checks)
- **Checkpoint**: equivalent to Core's checkpoints
- **Validation**: all verification (transactions scripts, amounts, timelocks, ..) except whether prevouts exist and are unspent
- **Confirmability**: verification that transactions' prevouts exist and are unspent
- **Candidate**: most work chain which is not yet validated + confirmed
- **Transaction pool**: equivalent to Core's mempool but without the "mem" (it's stored on disk)

## Credits

Thanks to Eric Voskuil for patiently answering my questions and walking me through the inner working of Libbitcoin in more details. Obviously, all remaining mistakes are mine.

-------------------------

