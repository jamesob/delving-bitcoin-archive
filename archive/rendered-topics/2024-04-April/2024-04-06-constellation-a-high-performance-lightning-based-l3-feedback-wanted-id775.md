# Constellation - a high performance Lightning-based L3. Feedback wanted

azz | 2024-04-06 14:41:27 UTC | #1

-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA256


<div data-theme-toc="true"> </div>

Hi all,

I'm looking for funding for a project I'm working on so have decided to ask for technical feedback here. This will not be a definitive proposal as the design is still being iterated upon. Note all the terms to identify concepts are also a WIP, as they should be clear but should not be confused with concepts at lower layers. 

I'm working on a L3 protocol on top of Lightning called **Constellation**. The main goal is to mitigate UX issues seen at the two lower levels, such as non-instant transaction confirmation (L1), rising transaction fees (L1), channel liquidity issues (whether routing or inbound) (L2) and throughput bottlenecks (L1) by making use of various different protocols and technologies either finalised or in active development. All funds on L3 are 1:1 backed by funds held in L2 channels. 

## the complicated stuff

Constellation has a **federated security model that's enforced via ROAST**, a wrapper around the FROST protocol to create BIP 340 compatible Schnorr signatures. However, unlike other federated scaling layers, **Constellation is a network of multiple, interoperable federations** which settle differences via Lightning. This means that **any trust model is possible** with Constellation. Entities who have the capabilities to run their own nodes and funds to pay for on-chain Lightning channel operations can be fully self-sovereign and self-sufficient. Entities that would prefer not to run their own nodes, likely the majority of people, can opt to choose a federation with a trust model that suits them, nominally a guild consisting of many independent operators. 

These federations are referred to as **guilds**, and the members that they consist of are called **operators**. While guilds can increase their security linearly by adding more (honest) operators, the performance loss scales roughly only logarithmically, due to how L3's ledgers work.

Unlike another federated layer like Liquid, operators have individual ledgers and their own underlying Lightning node(s). Constellation's ledgers are very different from how other ledgers work. They are UTXO-based, like Bitcoin, as opposed to account based. However, a single operator actually operates 2^16 distinct partitions. A given transaction on L3 may be present on multiple partitions, each of which is able to determine a "leader" partition. We exploit the seemingly random nature of hashes, as each input and output will have a random "identifier". We can partition transactions using these hashes, to run 65536 interlinked, communicating ledgers per operator. 

As a basic example, we have a transaction A + B -> C + D, where A and B are inputs and C and D are outputs. When received, this transaction will be partitioned based on its inputs - determining the physical CPU core(s) that process the transaction. Upon receiving a transaction, it's in the unconfirmed state. Let's say that A and C both are in the same partition, P1 (via this hashing prefix mechanism) and B and D are on different partitions, P2 and P3 respectively. That means we have 3 partitions that this transaction should be present on. Next,  we work out what physical CPU core each of those partitions is on. P1, P2 and P3 may all actually be on the same CPU core. They may be on different CPU cores, even different machines (an operator is enforced to run multiple machines). A CPU core will operate at least one partition - this enables an operator to scale their potential throughput up and down by adding and removing CPU cores.

One of these partitions is determined as the "leader" of this transaction. The leader is responsible for deciding when a transaction is confirmed. Each partition will only have a subset of the data, either input or output. If P1 is chosen as the leader, it will validate the script of A. P2 and P3 will receive copies of the transaction and P2 will validate the spending script for B. All partitions will persist their copy of the transaction to disk, and P2 and P3 will message P1 when they've applied the transaction to their local ledger. Once P1 receives confirmation from the others, it will confirm the transaction in its own ledger and inform the other partitions to confirm it too. Once they all respond confirming the transaction, the transaction is confirmed within a single operator. 

Note that while I say it confirms the transaction, it actually batches these in "blocks". These blocks are limited in size, but have strict local deadlines of 50µs. If the size limit is hit before the deadline, the block is sent onwards - else it is sent upon hitting the deadline. Transactions are processed in batches to reduce overhead. Batches moving between partitions during the confirmation stage move in *packages*, and each partition advances its ledger using batches of transactions called blocks. Once there are transactions that have finished validation, they'll be put into blocks as fast as they're required. 

Transactions will often, even mostly, be contained within multiple blocks. When a leader of a partition creates a block, it tells its replicants. The replicants reply when they've validated and successfully persisted the block. When the leader hears back from the majority of operators in this replica (note: *not* majority of replicants), it confirms the block locally and tells the replicants to confirm it, advancing their copies of the partition. Unlike Bitcoin, two blocks at the same "height" may never be created. There can only possibly be one leader at a time, as it needs a majority of operators to confirm. If the block is invalid, the other operators may reject it and elect a new leader. 

Let's say we have a basic 2-of-3 guild (2 operators need to confirm, 3 operators total) made of O1, O2 and O3. Each operator maintains 2^16 partitions. Partitions are replicated twice within every operator, ensured to be on separate machines. Therefore, O1 has 2^17 partitions for itself, 2^17 for replicating O2, and 2^17 for replicating O3. This is the same for every operator. This ensures incredible redundancy against attacks and failures, but makes the time-to-confirm for a transaction longer. Theoretically. Practically, the time-to-confirm still appears near-instant, faster than Lightning even. For a transaction to confirm, it needs all involved partitions to confirm. Each partition is its own state machine, replicated using this modified Raft consensus model. A given partition, for example O1-12345 has *2*3=6 copies* and *1 leader*. By default, this leader is operated by O1. If the leader dies, the replica inside O1 should take over. If neither is available, or if the operator is designated as malicious, O2 and O3 will start elect a leader between themselves. As majority holders, they can over-rule O1.

While complex, this model allows for extremely high transactional throughput. 2^16 leaders per operator mean each operator is limited to the transactional throughput of 65,536 CPU cores. Starlight, the official (Rust-based 🦀🦀) "node"/server implementation of Constellation will offload networking, signature verification, hashing and other heavier operations to different physical CPU cores to those that operate the partitions. This means Starlight will be able to achieve payment throughputs never really seen before, easily beating legacy fiat and bLocKcHaiNs while maintaining reasonable latency (< 3 sec, perhaps lower/higher, but fast) for payments. 

## the less complicated stuff

### deposits/withdrawals
Both are handled via eCash:
- - - minted via depositing funds over L2 or L1 to somewhere (LN channel or P2TR output) controlled by the ROAST threshold signature. 
- - - The notes are then redeemed into the L3 ledgers to a L3 address controlled by the depositor. 
  - This breaks the link between a user's depositing funds and the coins created on L3. 
- - - These funds are then spliced into the L2 channels that back each operator. 
- - - The L3 ledger will always have a total UTXO set valuing <= amount held on chain by the threshold multisig. eCash total amount + L3 ledger total amount = amount held in L2 channel local balances. Similar onchain mechanism to how phoenix does their magic stuff
- - - withdrawals are also handled by eCash, in mostly the same way but in reverse

"Withdrawals" can also be used for complete backwards compatibility, meaning a L3-based wallet can pay a L3 address, a L2 invoice or an L1 address from a single QR scan. UX is good. 

### cross operator/cross guild transactions?

*sorry i've run out of capital letters*

complicated but not really

each input and output will be identified by the operator that looks after it. all L3 funds are held in L2 local balances of channels with other operators. constellation is just a wrapper on top of lightning. except when you're dealing with billions of $s of bitcoin, liquidity issues don't exist anymore. your $100k soyboy EV payment won't have to completely ruin your channel balance. any big liquidity balance issues will be autonomously resolved by splicing in and out, because if a channel manages billions, the few thousand dollars of onchain fees to pay for the splice are pocket change.

when a transaction is made between operators (it doesn't matter if they're in the same guild or not), the funds are settled over LN. the funds will instantly show up as unavailable for each receiver, and these transactions can be queued. when the sending operator settles their differences (batching used, probably every second), the receiver works out how many of those queued transactions have now been paid off and can then make those funds available. 

### adding/removing machines to operators and operators to guilds?

constellation supports all four operations autonomously. these are all possible but i won't go into the details of how this works in this post

### how will devs use this?

#### wallets

devs can build L3 wallets in a very similar way to L1 wallets. we'd probably nab Bitcoin's UTXO and scripting language so stuff that works on L1 works without a huge amount of changes on L3. this would make updating hardware wallets "pretty easy" too.

for UX, maybe we can share a raw taproot public key derived via a HD wallet with each contact. they'd be able to create new addresses for the receiver by tweaking it, and they wouldn't be able to see any other coins owned by the receiver apart from those that they sent to them. offline sending? done.

#### electrs? mempool.space? Help?

Constellation will be both a node protocol and a light client protocol to enable standardised communication with operators. This will allow electrum-like API stuff directly to the operators. It'll inherit a lot of the performance and security designs of the actual operator architecture. While federated, the ledgers will still be publicly accessible and verifiable just like liquid... ish.

### but it's not trustless

no it's not, but Constellation makes reasonable tradeoffs that give humanity an incredibly advanced and capable financial interconnect while letting you run your own guild and maintaining full sovereignty. if you can afford the onchain fees. 

## summary

thank you for reading, i'm working on a blog post that will talk in more detail but i might just start writing the code instead. please let me know any technical feedback or thoughts or questions you may have. i appreciate all of y'all.
-----BEGIN PGP SIGNATURE-----

iHUEARYIAB0WIQTZqB4RHv8az8qZrF37Dus/ZKINRAUCZhFfFAAKCRD7Dus/ZKIN
RARKAQCdrP0E6m2S5Z/LYdgnJ7GR6YwELeBz0014IR5uoIVX8AD+NUX9Dop5z4pW
wkcwYfTHR3o2hUWwpWQhQhgSOowiQQ0=
=kEA5
-----END PGP SIGNATURE-----

-------------------------

