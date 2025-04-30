# SwiftSync -- Speeding up IBD with pre-generated hints (PoC)

theStack | 2025-04-09 10:41:39 UTC | #1

(EDIT: The working title for this thread was initially "IBD Booster", but the official name for the proposal is now "SwiftSync", see https://delvingbitcoin.org/t/swiftsync-speeding-up-ibd-with-pre-generated-hints-poc/1562/2?u=thestack and https://gist.github.com/RubenSomsen/a61a37d14182ccd78760e477c78133cd. Sorry for creating this confusion in naming, I should have checked with Ruben before publishing.)

A few weeks ago @RubenSomsen proposed to speed up the IBD phase in Bitcoin Core by 
reducing the chainstate operations with the aid of pre-generated hints. Right now, the main optimization
is to skip script verification up to the `-assumevalid` block (enabled by default, [updated on each release](https://github.com/bitcoin/bitcoin/blob/1344d3bd0f3b0ef3c4f339624da9ed96778bc138/src/kernel/chainparams.cpp#L121)), but all other block validation checks
are still active. In particular, UTXO set lookups and removals can be quite expensive and cause regular coins cache and expensive leveldb disk I/O operations. 

The idea is to only ever add coins to the UTXO set if we know that they will still be
unspent at a certain block height N. All the other coins we don't have to add or remove during IBD in the first place,
since we already know that they end up being spent on the way. The historical hints data consists of one boolean flag
for each transaction output ever created (up to including block N), signalling the answer to the question:
"Is this transaction output ending up in the UTXO set at block N?".

If the answer is yes, we obviously add it (as we would also normally do), but if the answer is no,
we ignore it. Consequently, we don't have to do any UTXO set lookups and removals during the IBD phase
anymore; the UTXO set only grows, and the order in which blocks are validated doesn't matter anymore,
i.e. the block validation could be done in parallel. The obvious way to store the hints data is bit-encoded, leading to a rough
size of `(number_of_outputs_created / 8)` bytes, being in the hundreds of mega-bytes area for mainnet (e.g. ~348 MiB for block 850900).

As a rough overview, the following checks in `ConnectBlock` are done in each mode:
|validation step | regular operation | assumevalid | IBD Booster|
|--- | --- | --- | ---|
|Consensus::CheckTxInputs  | :white_check_mark: | :white_check_mark: | :x: |
|BIP68 lock checks | :white_check_mark: | :white_check_mark: | :x: |
|SigOp limit checks | :white_check_mark:| :white_check_mark: | :x: |
|input script checks | :white_check_mark:| :x: | :x: |
|update UTXO set | :white_check_mark: | :white_check_mark: | grow-only, with MuHash check (see below)|
|block reward check | :white_check_mark: | :white_check_mark: | :x: |

To give some assurance that the given hints data is correct, the state of the spent coins is tracked
in a MuHash instance, where elements can be add and deleted in any order. If we encounter a transaction
output where our hints data says "false" (meaning, it doesn't end up in the final UTXO set at block N
and will be spent on the way there), we add its outpoint (txid+vout serialization) to the MuHash.
For any spend in a transaction, we then remove its outpoint (again txid+vout serialization)
from the MuHash. After block N, we can verify that indeed all the coins we didn't add to the UTXO have been spent (implying that the given hints data was correct) by verifying that the MuHash represents
an empty set at this point (internally that can be checked by comparing that the numerator and denominator have the same value).

I've implemented a proof-of-concept of this proposal and called it "IBD Booster", consisting of two parts:
* A python script ibd-booster-hints-gen, which builds on [py-bitcoinkernel](https://github.com/stickies-v/py-bitcoinkernel) and takes a datadir and a utxo set in SQLite format (as created by the [utxo_to_sqlite.py](https://github.com/bitcoin/bitcoin/blob/1344d3bd0f3b0ef3c4f339624da9ed96778bc138/contrib/utxo-tools/utxo_to_sqlite.py) script) as input, and outputs the bit-encoded hints file: https://github.com/theStack/ibd-booster-hints-gen
* A Bitcoin Core branch which supports reading in the hints file and use it for the optimization as described above with a `-ibdboosterfile` parameter: https://github.com/theStack/bitcoin/tree/ibd_booster_v0

The main logic is implemented in a function called `UpdateCoinsIBDBooster`, which is called as a drop-in replacement of `UpdateCoins` if
the IBD Booster is active for the current block, see https://github.com/theStack/bitcoin/blob/ce4d1aa5ac4e2a08172bcbc7e80f9b5675b20983/src/validation.cpp#L2106-L2119 vs. https://github.com/theStack/bitcoin/blob/ce4d1aa5ac4e2a08172bcbc7e80f9b5675b20983/src/validation.cpp#L2121-L2152

A guide for trying out the proof-of-concept implementation is following:


<h2>Generate the hints data</h2>

1. Run a non-pruned node, sync at least to the desired booster target height (886000 in this example)

2. Dump UTXO set at target height, convert it to SQLite3 database

```
$ ./build/bin/bitcoin-cli -rpcclienttimeout=0 -named dumptxoutset $PWD/utxo-886000.dat rollback=886000
$ ./contrib/utxo-tools/utxo_to_sqlite.py ./utxo-886000.dat ./utxo-886000.sqlite
```

3. Stop Bitcoin Core

```
$ ./build/bin/bitcoin-cli stop
```

4. Fetch the IBD Booster branch (contains the booster hints generation script in `./contrib`)
```
$ git clone -b ibd_booster_v0 https://github.com/theStack/bitcoin
$ cd bitcoin
```

4. Create IBD Booster file containing hints bitmap
(first start might take a while as the py-bitcoinkernel dependency has to be built in the background first [1])

```
$ pushd ./contrib/ibd-booster-hints-gen
$ uv run booster-gen.py -v ~/.bitcoin ../../utxo-886000.sqlite ~/booster-886000.bin
$ popd
```

<h2> Setup a fresh node using IBD Booster </h2>

Instructions:

1. Build and run the IBD-Booster branch (checked out above already)
```
$ <build as usual>
$ ./build/bin/bitcoind -ibdboosterfile=~/booster-886000.bin
```
2. Wait and enjoy. If everything goes well, you should see the following debug message after the booster target block:
```
*** IBD Booster: MuHash check at block height 886000 succeeded. ***
```

<h2> First benchmark results </h2>
On a rented large-storage, low-performance VPS I observed a ~2,24x speed up (40h 54min vs. 18h 14min) for running on mainnet with `-reindex-chainstate` (up to block 850900 [2]):

```
$ time ./build/bin/bitcoind -datadir=./datadir -reindex-chainstate -server=0 -stopatheight=850901

real    2454m37.039s
user    9660m20.123s
sys     380m30.070s

$ time ./build/bin/bitcoind -datadir=./datadir -reindex-chainstate -server=0 -stopatheight=850901 -ibdboosterfile=./booster850900.bin

real    1094m31.100s
user    1132m53.000s
sys     46m45.212s
```

The generated hints file for block 850900 can be downloaded here: https://github.com/theStack/bitcoin/raw/refs/heads/booster_data_850900/booster850900.bin.xz (note that this file has to be unpacked first using the `unxz` command)

One major drawback of this proposal is that we can't create undo data (the rev*.dat files) anymore, as we obviously don't have the prevout
information available in the UTXO set during the IBD booster phase. So the node we end up is unfortunately limited in functionality, as some indexes (e.g.
coinstatsindex and blockfilterindex) and RPC calls (e.g. getblock for higher verbosity that shows prevout data) rely on these files for
creating/delivering the full data.
My hope is that this proof-of-concept still serves as a nice starting point for further research of simliar ideas, or maybe is useful
for projects like benchcoin. 

Potential improvements:
- investigate further speedups (e.g we don't really need the full coin cache functionality up to the target block)
- don't load all hints data into RAM at once
- refine file format to include metadata and magic bytes for easy detection
- research about potential compression of the hints data (probably not worth the complexity imho, but an interesting topic for sure)
- implement parallel block validation
- investigate including the coin heights in the hints data to enable more checks (leads to significantly larger hints data though)
- hints generation tool speedups?
- ...

Thanks go to @RubenSomsen, @davidgumberg, @fjahr and @l0rinc (sorry if I forgot anyone) for discussing this idea more intensely a few weeks ago, and to @stickies-v for providing the great [py-bitcoinkernel](https://github.com/stickies-v/py-bitcoinkernel) project (I was considering to use [rust-bitcoinkernel](https://github.com/TheCharlatan/rust-bitcoinkernel) instead for performance reasons, but my knowledge of Rust is unfortunately still too bad).
Suggestions and feedback in any form, or more diverse benchmark results (or suggestions on how to benchmark this best) would be highly appreciated.

Cheers,
Sebastian

[1] Note that I only tried to run the script with [uv](https://docs.astral.sh/uv/) so far, since it just worked (tm) flawlessly from the start without any headaches. Using other Python package management tools might also work to build and run this script, but I haven't tested them.

[2] Block 850900 is way in the past of the current assumevalid block, but I had to stop there as I ran out of space on the benchmark machine.

-------------------------

RubenSomsen | 2025-04-02 19:38:57 UTC | #2

Nice work! Really great to see such a large speedup already on the initial benchmark, especially considering this is without parallel block validation. Btw I now call this "SwiftSync" and I've been working on a full write-up. I'll send you the unpublished draft, but I already published an initial overview [here](https://gist.github.com/RubenSomsen/82ccd0f913057e05353b437457f68a11?permalink_comment_id=5510301#gistcomment-5510301) (along with the coredev session notes). I also gave a presentation on it at the recent Amsterdam bitdevs. Slides are [here](https://docs.google.com/presentation/d/1sqZIW3tBubaZRpTHJzgkiW1Vln-aDvrWxJIYI7JIMS8/edit), though I'm not sure how clear they'll be without context.

Without going too much into detail (I'll leave that for the write-up), here are two important points:

First, assumevalid is not actually a prerequisite (though it does make for an easier first PoC). Perhaps that was my fault for emphasizing it too much - you weren't the only one who got that impression. When processing the output you can add all the UTXO set data into the hash instead of just the outpoint. On the input side you'd then have to re-download the UTXOs that are being spent in each block (essentially the undo data, though we should encode it a bit more efficiently... maybe 10% more data) in order to remove it again. This allows you to do full validation without assumevalid (a few more steps are actually needed, but you'll see the details in the draft).

Second, MuHash is not strictly required. We can use a regular hash function like sha256 and do modular arithmetic, provided we add a salt to prevent birthday attacks. So it'd be `hash(utxo_data_A||salt) + hash(utxo_data_B||salt) - hash(utxo_data_C||salt) - hash(utxo_data_D||salt) == 0` (thus proving `(A==C && B==D) || (A==D && B==C)`). This should be faster than MuHash.

And one point of lesser importance, but it's simple to implement so why not: you can encode the bit hints as the number of 0's before each 1. So 00010000011001 would be encoded as 3,5,0,2. There is more you can do, but this should already save a ton of space.

Very exciting!

-------------------------

vostrnad | 2025-04-02 22:46:27 UTC | #3

I've looked at the current hints data format to think about how to best compress it, and I've spotted a flaw. It seems you're storing the output count of each block as a 2-byte sequence, but that only goes up to 65,535. This is more than the current mainnet maximum of 26,908 outputs in a block, but since the smallest possible output only takes up 9 bytes, the theoretical maximum is well over 65,535 (demonstrated e.g. by [this 100 kB transaction with 10,002 outputs](https://mempool.space/tx/3c2284977f93a1395f1b0e06fd3a843c3e8ce2277a0d770781f6655d1aadfe13)).

This could be fixed by using a variable-length encoding for the output count, but do you even need it at all? The order of outputs as they are created is defined just as well without splitting them into blocks. (Also you'd avoid having to pad between blocks for a modest 0.4375 bytes saved per block on average, on top of the savings from removing the output counts.)

Fine work otherwise.

-------------------------

0xB10C | 2025-04-03 10:58:20 UTC | #4

[quote="RubenSomsen, post:2, topic:1562"]
And one point of lesser importance, but it’s simple to implement so why not: you can encode the bit hints as the number of 0’s before each 1. So 00010000011001 would be encoded as 3,5,0,2. There is more you can do, but this should already save a ton of space.
[/quote]

I remember thinking about this when first hearing about IBD booster / SwiftSync and just had a quick and dirty look at using delta / differential encoding together with CompactSize and VarInt's in this [notebook](https://gist.github.com/0xB10C/01900837f92e48da4800e57152ef2a1d). This seems to half the size of the uncompressed data. Though, as the hints will likely be distributed compressed and the `xz` compressed data is about the same size for all combinations, I'm not sure if it's worth to differentially encode it. 

| filename                          | size (bytes)      |
|-----------------------------------|------------|
| booster850900.bin                 | 364964716  |
| differential_compact_size.bin     | 188209875  |
| differential_varint.bin           | 190098381  |
|            |   |
|            |   |
|            |   |
| booster850900.bin.xz              | 87882052   |
| differential_compact_size.bin.xz  | 81697540   |
| differential_varint.bin.xz        | 81321492   |

Note that most of the median deltas are below 253 and encodable with a 1 byte CompactSize: 
![image|570x432](upload://xdCGxjytrHxfH80WKhEO2vTZVq5.png)

-------------------------

jamesob | 2025-04-03 12:55:03 UTC | #5

Very cool work.

I don't mean to be discouraging, but isn't this more or less what we spent a bunch of time on assumeutxo to accomplish? 

IBD booster requires (as far as I can tell) essentially trusted metadata -- the hints -- that were produced using some other, already synced node. It also requires modifications to very sensitive parts of the validation code in order to make use of the hints.

Assumeutxo snapshots, on the other hand, aren't trusted in the same way that hints seem to be given that the snapshots are committed to with hashes in the source code.

IBD booster provides a ~2.24x speedup -- which again is very cool -- but assumeutxo provides a higher order of speedup (sync in an hour or two) given that we're truncating the IBD process to the history between the snapshot base and the current network tip.

---

This is very interesting research, and I don't mean to discourage. But I'm just wondering if it isn't worth instead investing more effort to "deliver" assumeutxo in terms of making it usable with the GUI etc. rather than inventing and implementing yet another way to produce exogenous metadata, distribute it safely, and use it to speed up IBD.

-------------------------

RubenSomsen | 2025-04-03 14:39:09 UTC | #6

>Assumeutxo snapshots, on the other hand, aren’t trusted in the same way that hints seem to be given that the snapshots are committed to with hashes in the source code.

The exact same can apply to the hints - they could equally be committed in the source code. 

Also note that unlike assumvalid/assumeutxo the hints cannot affect consensus since validation will fail if the hints aren't correct rather than ending up on the wrong chain. This makes it much more conceivable to get them from a trusted third party and use them to validate all the way up to the tip (the worst they can do is waste your time).

>isn’t this more or less what we spent a bunch of time on assumeutxo to accomplish

It's actually completely orthogonal.

Today your options are to start with assumeutxo or not, and then to either use assumevalid or not for (background) validation. SwiftSync speeds up the latter, regardless of whether you utilize assumeutxo or not.

In fact, assumeutxo and SwiftSync synergize quite nicely in some ways. SwiftSync makes background validation near-stateless, meaning you wouldn't have to manage two full chainstates when you're simultaneously validating at the tip. And if you use SwiftSync (specifically the non-assumevalid version) with assumeutxo then you don't even need the hints anymore. You can just add every output (spent or unspent) to the hash aggregate and subtract the UTXO set from it (i.e. all outputs - inputs - UTXO set == 0).

-------------------------

theStack | 2025-04-09 10:29:17 UTC | #7

Thanks for the update and the resources, that all sounds very promising! I'll focus on the assumevalid version with my reply for now, since I still haven't fully processed the non-assumevalid one yet.

[quote="RubenSomsen, post:2, topic:1562"]
Second, MuHash is not strictly required. We can use a regular hash function like sha256 and do modular arithmetic, provided we add a salt to prevent birthday attacks. So it’d be `hash(utxo_data_A||salt) + hash(utxo_data_B||salt) - hash(utxo_data_C||salt) - hash(utxo_data_D||salt) == 0` (thus proving `(A==C && B==D) || (A==D && B==C)`). This should be faster than MuHash.
[/quote]

Nice idea! As already noted in private, I'm wondering though if only adding a salt is really enough to provide the same security guarantees as MuHash would do? Reducing the modulo field size from 3072 bits to 256 bits and using simple addition instead multiplication as basic operation seem to be quite drastic cuts. This is more "gut-feeling" and far away from a mathematical sound argument, but it feels to me that this was too good to be true if there aren't any significant downsides to this approach. (That said, I hope I'm wrong, and even if there is a reduction in security, it's maybe still good enough for the SwiftSync use-case, since the goals of MuHash and the aggregate hash here are different.)

Since it was relatively easy to do, I adapted the implementation above to take use of that suggestion (by abusing secp256k1 scalars for the aggregate hash type): https://github.com/theStack/bitcoin/tree/ibd_booster_v1_addsub_sha256 (commit https://github.com/theStack/bitcoin/commit/494692ebc57e159c36b0a2042abca59f539ca0c2)

The results are pleasant, with this I'm now already observing a ~5x IBD speed-up compared to the assumevalid run (again, up to block 850900, and using `-reindex-chainstate`), and this is still without any parallel block validation involved:

| | assumevalid only | SwiftSync (MuHash for aggregate hash) | SwiftSync (salted sha256 add/sub for aggregate hash) |
|--- | --- | --- | ---|
|time | 2454m37.039s | 1094m31.100s | 464m37.234s |
|speed-up | - | ~2.24x | ~5.28x |

Doing parallel block validation as next proof-of-concept step would be very nice, but that needs of course much more invasive changes in the codebase.

-------------------------

instagibbs | 2025-04-08 13:43:29 UTC | #8

A key trade-off is that a salt must be kept private for security of the node if hashes are being used.

-------------------------

harding | 2025-04-08 18:54:43 UTC | #9

> A key trade-off is that a salt must be kept private for security of the node if hashes are being used

Obviously the salt should not be publicly exposed, but are you sure it _needs_ to be secret?  If the hints file commits to the chaintip at the end of SwiftSync and the user has a copy of (or commitment to) the hints file when they choose their salt, how can an attacker use knowledge of the salt to create a TXO that negates a previously accumulated TXO?  The chaintip commits to all of the past txids, so they can't be changed (or we have bigger problems!).  The node also won't be using SwiftSync after IBD, so the attacker can't create new TXOs based on learning the salt.  I think that means that even an attacker who knows the salt can't do anything with it.

The exception would be if you started SwiftSync with one hints file and then switched to another (e.g., you upgraded node versions in the middle of IBD).  However, that must be forbidden anyway since a TXO that was previously hinted to be a UTXO might now be a STXO.

-------------------------

instagibbs | 2025-04-08 18:56:35 UTC | #10

[quote="harding, post:9, topic:1562"]
The node also won’t be using SwiftSync after IBD, so the attacker can’t create new TXOs based on learning the salt.
[/quote]

You might be right, but imo that's a lot of "probably"s we should think about a bit more robustly.

-------------------------

RubenSomsen | 2025-04-09 10:30:49 UTC | #11

I just published the [full SwiftSync write-up](https://gist.github.com/RubenSomsen/a61a37d14182ccd78760e477c78133cd).

It contains more details about the hash aggregate, and answers the following questions (and more):

- How exactly assumevalid is utilized and what the implications are
- How we can still check for inflation when we skip the amounts with
assumevalid
- How we can validate transaction order while doing everything in parallel
- How we can perform the BIP30 duplicate output check without the UTXO set
- How this all relates to assumeutxo

To my knowledge, every validation check involving the UTXO set is covered,
but I'd be curious to hear if anything was overlooked or if you spot any
other issues.

-------------------------

sjors | 2025-04-17 12:30:18 UTC | #12

[quote="theStack, post:7, topic:1562"]
I’ll focus on the assumevalid version with my reply for now, since I still haven’t fully processed the non-assumevalid one yet.
[/quote]

I'm much more excited about the SwiftSync variant that makes _less_ assumptions than `-assumevalid`. Would love to try a demo of that... https://gist.github.com/RubenSomsen/a61a37d14182ccd78760e477c78133cd#without-assumevalid

Maybe implementing this as a `-reindex` is easier for a PoC, since you already have the undo data on disk?

Regarding `MuHash` performance, have you tried with this? https://github.com/bitcoin/bitcoin/pull/19181. Though I guess this one https://github.com/bitcoin/bitcoin/pull/21590 added much more performance boost, and is already in your branch. One comment in that PR suggests there's more potential gain: "Using libgmp for inverses is 1.5x-2x faster still"

-------------------------

l0rinc | 2025-04-30 11:24:37 UTC | #13



# SwiftSync Benchmark Analysis and Future Ideas

I love this idea (needed a few iterations with Ruben to understand how it changes the guarantees); it's a much-needed alternative for something between full IBD and AssumeUTXO.
@theStack created an excellent prototype that makes it easy to check it out and experiment further.

I usually test my IBDs using hyperfine for precision, either an i9 processor with an NVMe SSD or an i7 with an HDD.

I have separated the addition of ibdboosterfile/swiftsyncfile argument to be the very first commit so that I can run both before/after versions with the exact same arguments (in the first instance the value is ignored; it's basically master branch with an unused arg).
I have measured a `-reindex-chainstate` instead of a full IBD to reduce unrelated networking fluctuations.

I have first tried it with the provided *850900* block hints with MuHash for `-dbcache=4500` (since we're not testing flushing behavior, so let's use plenty of memory).
As mentioned before, it seems like the MuHash version is surprisingly slow, it's actually 2x **slower** than master:

[details="MuHash benchmark details with 4.5 GB cache"]

```bash
COMMITS="76a731ed49e4234e62a79f492657d0138ded6437 9cdda4f301aa4aa1113024ee52c87b92449ad02b"; \
STOP_HEIGHT=850901; DBCACHE=4500; \
CC=gcc; CXX=g++; \
BASE_DIR="/mnt/my_storage"; DATA_DIR="$BASE_DIR/BitcoinData"; LOG_DIR="$BASE_DIR/logs"; \
(echo ""; for c in $COMMITS; do git fetch origin $c -q && git log -1 --pretty=format:'%h %s' $c || exit 1; done; echo "") && \
hyperfine \
  --sort 'command' \
  --runs 1 \
  --export-json "$BASE_DIR/rdx-${COMMITS// /-}-$STOP_HEIGHT-$DBCACHE-$CC.json" \
  --parameter-list COMMIT ${COMMITS// /,} \
  --prepare "killall bitcoind; rm -f $DATA_DIR/debug.log; git checkout {COMMIT}; git clean -fxd; git reset --hard; \
    cmake -B build -DCMAKE_BUILD_TYPE=Release -DENABLE_WALLET=OFF && cmake --build build -j$(nproc) --target bitcoind && \
    ./build/bin/bitcoind -datadir=$DATA_DIR -stopatheight=$STOP_HEIGHT -dbcache=5000 -printtoconsole=0; sleep 100" \
  --cleanup "cp $DATA_DIR/debug.log $LOG_DIR/debug-{COMMIT}-$(date +%s).log" \
  "COMPILER=$CC ./build/bin/bitcoind -datadir=$DATA_DIR -reindex-chainstate -blocksonly -connect=0 -printtoconsole=0 -stopatheight=$STOP_HEIGHT -dbcache=$DBCACHE -ibdboosterfile=../booster850900.bin"
```

```
> branch - 76a731ed49 IBD Booster: bugfix: skip txs that are overwritten later (duplicate txids)
> master - 9cdda4f301 Add ibdboosterfile arg - to simplify benchmark params
```

```
Benchmark 1: COMPILER=gcc ./build/bin/bitcoind -datadir=/mnt/my_storage/BitcoinData -stopatheight=850901 -dbcache=4500 -reindex-chainstate -blocksonly -connect=0 -printtoconsole=0 -ibdboosterfile=../booster850900.bin (COMMIT = 76a731ed49e4234e62a79f492657d0138ded6437)
  Time (abs ≡):        30699.125 s               [User: 30626.081 s, System: 534.603 s]

Benchmark 2: COMPILER=gcc ./build/bin/bitcoind -datadir=/mnt/my_storage/BitcoinData -stopatheight=850901 -dbcache=4500 -reindex-chainstate -blocksonly -connect=0 -printtoconsole=0 -ibdboosterfile=../booster850900.bin (COMMIT = 9cdda4f301aa4aa1113024ee52c87b92449ad02b)
  Time (abs ≡):        15758.185 s               [User: 17931.351 s, System: 832.149 s]
```

```
Relative speed comparison
        1.95          COMPILER=gcc ./build/bin/bitcoind -datadir=/mnt/my_storage/BitcoinData -stopatheight=850901 -dbcache=4500 -reindex-chainstate -blocksonly -connect=0 -printtoconsole=0 -ibdboosterfile=../booster850900.bin (COMMIT = 76a731ed49e4234e62a79f492657d0138ded6437)
        1.00          COMPILER=gcc ./build/bin/bitcoind -datadir=/mnt/my_storage/BitcoinData -stopatheight=850901 -dbcache=4500 -reindex-chainstate -blocksonly -connect=0 -printtoconsole=0 -ibdboosterfile=../booster850900.bin (COMMIT = 9cdda4f301aa4aa1113024ee52c87b92449ad02b)
```

[/details]

Luckily @theStack provided an alternative which just uses SHA-256 to keep track of spentness via `HashWriter`.
I have tested this version as well by first generating the hints until block *888888* for good luck🍀 and gave it more cowbell (45 GB in-memory cache to have a single flush at the end to reduce noise).
This is indeed 56% faster for me, resulting in a 2h:47m reindex-chainstate - sweet, I could get used to that.

[details="SHA-256 benchmark details with 45 GB cache"]

```bash
COMMITS="ab8dec1c87aef36e390bcc7d64d8604cc9170b93 e7194c13507a89544a3cf5f31c4eab88c19ba72b"; \
STOP_HEIGHT=888888; DBCACHE=45000; \
CC=gcc; CXX=g++; \
BASE_DIR="/mnt/my_storage"; DATA_DIR="$BASE_DIR/BitcoinData"; LOG_DIR="$BASE_DIR/logs"; \
(echo ""; for c in $COMMITS; do git fetch origin $c -q && git log -1 --pretty=format:'%h %s' $c || exit 1; done; echo "") && \
hyperfine \
  --sort 'command' \
  --runs 1 \
  --export-json "$BASE_DIR/rdx-${COMMITS// /-}-$STOP_HEIGHT-$DBCACHE-$CC.json" \
  --parameter-list COMMIT ${COMMITS// /,} \
  --prepare "killall bitcoind; rm -f $DATA_DIR/debug.log; git checkout {COMMIT}; git clean -fxd; git reset --hard; \
    cmake -B build -DCMAKE_BUILD_TYPE=Release -DENABLE_WALLET=OFF && cmake --build build -j$(nproc) --target bitcoind && \
    ./build/bin/bitcoind -datadir=$DATA_DIR -stopatheight=$STOP_HEIGHT -dbcache=5000 -printtoconsole=0; sleep 100" \
  --cleanup "cp $DATA_DIR/debug.log $LOG_DIR/debug-{COMMIT}-$(date +%s).log" \
  "COMPILER=$CC ./build/bin/bitcoind -datadir=$DATA_DIR -reindex-chainstate -blocksonly -connect=0 -printtoconsole=0 -stopatheight=$STOP_HEIGHT -dbcache=$DBCACHE -swiftsyncfile=../ibd-booster-888888.bin"
```

```
> master - ab8dec1c87 init: add -swiftsyncfile option
> branch - e7194c1350 init: use the -swiftsyncfile option
```

```
Benchmark 1: COMPILER=gcc ./build/bin/bitcoind -datadir=/mnt/my_storage/BitcoinData -stopatheight=888888 -dbcache=45000 -reindex-chainstate -blocksonly -connect=0 -printtoconsole=0 -swiftsyncfile=../ibd-booster-888888.bin (COMMIT = ab8dec1c87aef36e390bcc7d64d8604cc9170b93)
  Time (abs ≡):        15653.472 s               [User: 19512.089 s, System: 663.930 s]

Benchmark 2: COMPILER=gcc ./build/bin/bitcoind -datadir=/mnt/my_storage/BitcoinData -stopatheight=888888 -dbcache=45000 -reindex-chainstate -blocksonly -connect=0 -printtoconsole=0 -swiftsyncfile=../ibd-booster-888888.bin (COMMIT = e7194c13507a89544a3cf5f31c4eab88c19ba72b)
  Time (abs ≡):        10040.169 s               [User: 11106.200 s, System: 533.065 s]
```

```
Relative speed comparison
        1.56          COMPILER=gcc ./build/bin/bitcoind -datadir=/mnt/my_storage/BitcoinData -stopatheight=888888 -dbcache=45000 -reindex-chainstate -blocksonly -connect=0 -printtoconsole=0 -swiftsyncfile=../ibd-booster-888888.bin (COMMIT = ab8dec1c87aef36e390bcc7d64d8604cc9170b93)
        1.00          COMPILER=gcc ./build/bin/bitcoind -datadir=/mnt/my_storage/BitcoinData -stopatheight=888888 -dbcache=45000 -reindex-chainstate -blocksonly -connect=0 -printtoconsole=0 -swiftsyncfile=../ibd-booster-888888.bin (COMMIT = e7194c13507a89544a3cf5f31c4eab88c19ba72b)
```

[/details]

The final flush from memory to disk was included in the benchmark, it took roughly 12½ minutes (see https://github.com/bitcoin/bitcoin/pull/32043 for other pull requests meant to speed that up).

[details="Log details"]

```bash
2025-04-30T06:44:15Z *** SwiftSync: aggregate hash check at terminal block height 888888 succeeded. ***
2025-04-30T06:44:15Z UpdateTip: new best=00000000000000000001347938c263a968987bf444eb9596ab0597f721e4e9e8 height=888888 version=0x24bc4000 log2_work=95.509247 tx=1169009164 date='2025-03-22T09:15:51Z' progress=0.986891 cache=25442.8MiB(173676464txo)
...
2025-04-30T06:44:15Z [warning] Flushing large (24 GiB) UTXO set to disk, it may take several minutes
2025-04-30T06:56:52Z Shutdown: done
```

[/details]

## Future Ideas and Questions
I have a few ideas and questions about the implementation (some might be a bit undercooked for now):

1. **Further Optimizations**
   * Do we really need a cryptographic hash here, or can we use a "pretty-good" hash instead? I'd like to better understand the requirements for salting, especially if we're storing the hint hashes as well.
   * Could we use a native 64/128-bit aggregator instead of a 256-bit one? Especially if we can split one big aggregator to one-per-thread, which we'd combine at the very end to check for validity (fork-join)... Given a 256-bit hash for the hints and AssumeUTXO heights, can we make this "less secure"?

2. **Memory/Disk Handling**
   * Since we don't need to read or remove from the UTXO set anymore, we could use a concurrent queue/ring-buffer instead of the dbcache and write continuously to disk via a background thread (avoiding the expensive final flush). Or maybe just directly write to LevelDB with bigger in-memory cache.
   * We could investigate what it would take to add undos on a background thread during swift-syncing (e.g. while the main thread connects the next block).
   * Would this enable ultra-low-memory IBD (e.g., 100MiB, assuming gradual hint loading)?

3. **Validation and Security**
   * We should definitely validate that total amounts at the current tip don't exceed expected mined values (just noticed you mention this in the latest version of the writeup as well). While assumevalid skips script validation (doesn't really concern me), I find skipping amount validation more concerning (potential inflation bugs).
   * We need more negative tests for various invalid scenarios (invalid hints, different tricky double-spends, etc.).
   * Validate current state against hard-coded AssumeUTXO heights to fail early if inconsistencies are found. We could also add the utxo-hint-hashes there as well to be able to request the hints from any node safely.
   * Do we need to validate whether any of the outpoints accidentally hash to the same value?

4. **Performance**
   * How would this combine with other pending IBD optimizations (batched block reading/writing, obfuscation speedup, faster UTXO batch writing, etc.)?
   * What are the next bottlenecks after removing the obvious ones (since this removed the most obvious ones, which eclipsed all other ones)?
   * How many blocks have outputs that were either all spent or all unspent until a given height?  Could we optimize the hints for these cases? Can we optimize the storage compressibility (e.g. sorting by similar block spends)?

5. **Usage Extensions**
   * Could SwiftSync work for very low-performance devices (e.g. smartwatch) even after IBD (reorgs)?
   * Should we keep updating the hints after IBD to speed up RPC calls (maybe `gettxout`, `gettxoutsetinfo`, `scantxoutset`, `listunspent`, etc)?
   * Initially introduce this for `reindex(-chainstate)` only, since we've already validated the chain.


I'll try to parallelize this in the following weeks (threadpool + coroutines?), which should be straightforward (though measuring takes time), and apply other orthogonal optimizations (see https://github.com/bitcoin/bitcoin/pull/32043) to get a clearer picture of what we can achieve here.

-------------------------

