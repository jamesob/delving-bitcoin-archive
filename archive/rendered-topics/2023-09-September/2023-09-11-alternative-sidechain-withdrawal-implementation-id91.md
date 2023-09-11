# Alternative sidechain withdrawal implementation

ursuscamp | 2023-09-11 11:59:50 UTC | #1

## Preface

Since the furor over drivechains began recently, I have been thinking on ideas to maybe improve the BIP-300 sidechain withdrawal implementation in various ways.

I landed on the concept of a lightweight daemon (a **Sentinel**) which can run next to Bitcoin Core. Its job is to tell miners (and potentially other full nodes) when a withdrawal is valid so that it may be mined, using Nostr as a secondary communication layer to coordinate with sidechain nodes and users.

The first iteration of this was a distributed trust-based model where Bitcoin full nodes would trust their own preferred quoroms of sidechain nodes (companies, community members, etc) to tell them which withdrawal transactions are invalid. Other than the fact that it is less than ideal from a trust perspective, there are understandable fears about sidechain disagreements leading to Bitcoin forks.

So here is an alternative, improved approach.

## The Idea

There are three parties: The sidechain `S`, miner `M` and sidechain user `U`.

There are some interesting sidechain that `M` wants to support, so he begins running the Sentinel daemon.

In order to function as a Sentinel chain, the sidechain `S` must provide a loadable WASM module of configuration and consensus code for the sidechain. WASM is ideal for this because it is truly cross-platform and properly sandboxed. The WASM module is loaded by `M` into the Sentinel daemon. It configures the Sentinel daemon about where and how to find the sidechain blockheaders. Don't worry, our Sentinels will NOT be validating and storing the sidechain's blockchain. They will be SPV light clients for the sidechain. They will validate the blockheaders correctly descend from the sidechain genesis block, according to the consensus rules of sidechain (using PoW or other rules).

If you're thinking, "Wait, SPV requires trust or people can just generate fake blockheaders with thefts!" Don't worry, we'll get there.

So sidechain user `U` wishes to make a withdrawal from sidechain to Bitcoin. How would this work?

1. `U` makes a withdrawal transaction on `S` and it is included in an `S` block. The header of the block must include a merkle root for the transaction tree, and also a merkle root for the global state (UTXO set, account set, etc).
2. A marker transaction is placed on the Bitcoin blockchain with the `S` blockhash. This proves a sidechain block (and thus the withdrawal) existed at this point in history. This can be placed by any means, such as MM, BMM or whatever mechanism we choose.
3. There is some arbitrary delay. Let's say 100 blocks. After the marker transacation, the blockhashes of the _Bitcoin_ blocks are aggregated through XOR or cryptographic hashing and used as randomized input for the next steps.
4. After the mandated delay, `U` makes the withdrawal transaction from a sidechain UTXO on the Bitcoin chain.
5. At the same time, `U` publishes a withdrawal proof on Nostr. The randomized data from step `3` is used as input to a function that determines which random parts of the block must be revealed in the proof. The proof must contain:
  A. Proof that their withdrawal is in the transaction. 
  B. Deterministic set of transactions generated from the random data of the blockhashes.
  C. Enough of the global state to validation A and B.
6. If step `5` returns `true`, then the transaction is valid and miners can mine it. If it returns `false` the withdrawal is a forgery. Miners should not attempt to mine a withdrawal transaction until a valid proof is available, or they risk their block being invalidated by nodes.

## Why This Works

Under a normal SPV proof, the node you receive the block header from must be trusted because they can create a blockhash where the transaction merkle root is filled with nothing but dummy data except for the exact path that the user's transction is in.

In the described system above, someone publishes the block with the withdrawal to Bitcoin first. The Bitcoin blockhases in the mandatory delay after the above withdrawal introduce randomness. The randomness is used to determine which parts of the withdrawal block must be revealed by the withdrawer to prove they are withdrawing from a real block. In order to fake such a block, they would have to somehow predict all of the Bitcoin blockhashes *after* they made their withdraw in order to create a fake block that passed the proof. In this way, we can be sure that even though only a small amount of the transactions in the sidechain block was validated, that it is a real block.

## Question of Data Availability

Some may have concerns about availability of historical data. I am not sure this is a concern. There are two types of data that would be necessary: Sidechain block headers and historical withdrawal proofs.

1. **Sidechain block headers**: Upon supporting `S` the Sentinel daemon will need to grab the block headers, so those much be available. Sidechain nodes will be motivated to make sure those are available on public Nostr servers since sidechain withdrawals will freeze unless they're available. Sidechains may even provide their own dedicated Nostr server.
2. **Historical withdrawal proofs**: These are possibly not necessary. There's no need for the Sentinel daemon to validate historical withdrawals by scanning the Bitcoin blockchain with historical proofs. If there was ever an invalid withdrawal (i.e. theft) then the sidechain will collapse immediately on its own as the peg falters and trust breaks. Thus, if a sidechain is still operating at the time a Sentinel daemon begins the validation, it's safe to assume the withdrawals prior to that date are valid.

## Sentinel chains vs. Drivechains

Pros:

* No consensus changes required in Bitcoin Core. Possibly no changes at all. Thus, no bug risk for node runners that don't care about sidechains.
* Long delays for withdrawals are not necessary.
* Large withdrawal packages are also not necessary.
* If a sidechain gains wide community support, non-miners nodes could also run the software, transitioning the withdrawal process from trusted to trustless.

Cons:

* More difficult to launch sidechains because it requires off chain coordination among miners or nodes prior to launch.
* Potential for forks with node participation.
* Drivechains are currently a more complete proposal.

## Inspirations

* [Drivechains/BIP-300](https://www.drivechain.info)
* [Softchains](https://gist.github.com/RubenSomsen/7ecf7f13dc2496aa7eed8815a02f13d1)

-------------------------

