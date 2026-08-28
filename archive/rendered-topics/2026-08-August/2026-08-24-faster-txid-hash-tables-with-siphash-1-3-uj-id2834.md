# Faster txid hash tables with SipHash-1-3-UJ

sipa | 2026-08-27 15:07:33 UTC | #1

I'd like to briefly highlight a small improvement that is coming in Bitcoin Core 32.0 (to be [released](https://github.com/bitcoin/bitcoin/issues/35122) in october): a custom variant of [SipHash](https://en.wikipedia.org/wiki/SipHash): [SipHash-1-3-UJ](https://github.com/bitcoin/bitcoin/blob/a0ccd4ad171567c2b911160a76bde65ba379b4f6/src/crypto/siphash.h#L118L159).

For context, Bitcoin Core's validation and P2P logic extensively makes use of hash tables, for remembering what peers already gave us, for deduplicating things, for indexes, and for caching UTXOs and other things.

For several of these, these is a (mild) concern about denial of service (DoS): the data in the tables is ultimately provided by our peers, who may be in a position to give us data specifically crafted to trigger collisions in the hash tables. Individual collisions are not a problem, and to some extent even expected, but a large amount of entries that all collide with one another (a multi-collision) would end up in the same hash table bucket, causing severe performance degradation. For this reason, Bitcoin Core uses the salted hash function SipHash (with a secret salt randomly generated at startup) where relevant.

SipHash is designed as a (salted) cryptographic hash function (technically, a [PRF](https://en.wikipedia.org/wiki/Pseudorandom_function_family)), but with just a 64-bit output. This is simpler and faster than a traditional "full" cryptographic hash function, but obviously its collision resistance cannot be better than $2^{32}$. Apart from that, it is as unpredictable as a 64-bit function can be. This makes it a great choice for DoS-resistant hash tables, and it is the default in several programming language implementations for this reason (including Rust and Python).

In one particular instance, the UTXO set cache (a very performance-critical component) it is overkill, however. UTXOs are indexed by the txid of the transaction that created them, and the position in its outputs. The crucial point here is that txids are *already* cryptographic hashes (double SHA256), so it is worth asking if that fact cannot be exploited in the hash function applied on top for computing hash table buckets.

To explain where improvements are possible, consider what the current (Bitcoin Core up to 31.x) hash function used for UTXOs is. *h* is the txid here, and *i* is the output position.

![siphash24_diagram|690x387](upload://b5zdfAezxllMb4j30ZOJi4nm83E.png)
![sipround_diagram|690x192](upload://clKvgvhKjqkMcEsw6touPA83PKF.png)

In total this involves 14 SipRound calls (10 in 5 Compress steps, 4 in Finalize).

To improve upon this, we make three changes:
* **Switch to SipHash-1-3.** This is a fairly common variant for hash tables (and the default in Python and Rust), as SipHash-2-4 is designed for a stronger notion of security (indistinguishability from random) than what is actually needed for hash tables (ease of creating multi-collisions). This just drops the number of SipRounds per Compress from 2 to 1, and per Finalize from 4 to 3.
* **Make it unpadded and block-based.** SipHash is traditionally defined over inputs that are sequences of *bytes*, which require some padding to convert to a sequence of 64-bit inputs fed to the Compress calls. This padding guarantees that inputs of different length cannot easily be made to collide, but we do not actually care about that here, as all inputs are the same size. The result is that we treat our hash function now in terms of an input that consists of 64-bit blocks directly, rather than bytes. To prevent confusion with the old scheme, the final constant XOR'ed into v<sub>2</sub> is changed from 0xff to 0x6465646461706e75 ("unpadded").
* **Add support for "jumbo" blocks.** With the above in place, we now allow each input block to be *either* a normal 64-bit block, or a large 256-bit jumbo block, allowing the latter *only* when they are themselves the output of a cryptographic hash. This means that 256-bit hashes in the input (like our txid) can now be processed as a single SipRound, rather than 4 of them. This is justified by the fact that while attackers have control over the input indirectly, the cryptographic hash in between means they cannot simultaneously control many bits (control $n$ bits at a cost of $2^n$ grinding work).

This is SipHash-1-3-UJ.

So Bitcoin Core 32.0 will use:

![siphash13uj_diagram|690x472](upload://6BVyiTJlhG7ZMFAH3BCnETw4SeX.png)

All together, this means we reduce the number of SipRounds from 14 to 5, or with all constant-time overheads, from 17.0 ns to 10.6 ns per lookup on my Ryzen 5950X CPU.

This construction has not received significant scrutiny from cryptographers, though I did run it by [Jean-Philippe Aumasson](https://www.aumasson.jp/), one of the authors of SipHash, who did not see a way to attack it after looking at it for 20 minutes. I believe that is acceptable in this context, because we are working in a setting where the SipHash keys k<sub>0</sub>, k<sub>1</sub> are randomly generated and not known to attackers. Despite that, there does not appear to be ways to construct speed up multi-collisions (over brute force) even if the keys *are* known, so this construction is likely still significant overkill.

This was introduced in [this PR](https://github.com/bitcoin/bitcoin/pull/35215), and is just one (rather small) performance improvement in Bitcoin Core 32.0. The biggest one is probably [fetch block input prevouts in parallel during ConnectBlock](https://github.com/bitcoin/bitcoin/pull/35295). The SipHash-1-3-UJ function was later also used in [hash keys and pack positions to reduce disk usage](https://github.com/bitcoin/bitcoin/pull/35531), and may end up being used in more contexts in later releases.

Thanks to all who helped us get this in, including @l0rinc and @andrewtoth.

-------------------------

andrewtoth | 2026-08-24 23:16:38 UTC | #2

Excellent writeup!

I would also like to point out that SipHash-1-3-UJ was also used to speed up [fetch block input prevouts in parallel during ConnectBlock](https://github.com/bitcoin/bitcoin/pull/35295). Since the parallel fetching change was merged first, the PR introducing SipHash-1-3-UJ also modified the [`earlier_txids` set](https://github.com/bitcoin/bitcoin/pull/35215/changes#diff-f0ed73d62dae6ca28ebd3045e5fc0d5d02eaaacadb4c2a292985a3fbd7e1c77cR386) in `CoinsViewOverlay::StartFetching` to use the new hash function. We need to store all txids for each block we connect in a set, which is used to filter out prevouts that are created by an earlier transaction in the same block. These prevouts will be created in the `CoinsViewOverlay` cache directly, so they must not be fetched from the main cache or disk.

This set must hash the txid for every transaction in each block for insertion, and then hash the prevout hash of every input of each transaction to check existence. This is all serial overhead on the main thread before any of the parallel fetching speedup can be made, so this lighter hash function is a clear win here as well. This set is only inserting txids that are the result of SHA256ing the transaction data by our node directly, so they fit the use case for this new hash function exactly.

-------------------------

ajtowns | 2026-08-28 05:46:54 UTC | #3

Would you consider writing up the spec in a BIP in case it's useful for over-the-wire protocols? :slight_smile: (I believe this is a reasonable fit for template sharing, erlay and potentially a bumped version of compact block relay)

-------------------------

