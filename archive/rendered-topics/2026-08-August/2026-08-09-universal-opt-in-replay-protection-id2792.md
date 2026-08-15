# Universal opt-in replay protection?

moonsettler | 2026-08-09 22:43:19 UTC | #1

I would like to propose for consideration adding opt-in replay protection in case a future fork war<sup>1</sup> requires it for resolution!

A hostile soft or hard fork could make stakeholders fighting against it difficult by not providing replay protection. This was an interesting lesson learned from recent events where people tried to ancestrally split their coins, but it has proven to be fairly difficult.

One way to do it for example is standardizing a 34 byte Annex payload as follows:

`<0xFAF0><32-byte-prior-block-hash>`

Transaction validation would work as before, except when it comes to block validation or block template construction, in which case the current (valid header chain with most work) must contain the hash.

Since the Taproot signatures commit to the annex, the signatures could therefore commit to a specific chain in case of a chain split condition. Obviously empty Annex remains valid and such signatures are expected to be valid on all chains.

This allows for bribing miners to work on a specific block or splitting coins to dump on a specific chain. Even if only one of the chains honors this rule, the coins can still be safely split.

Thoughts?

*<sup>1</sup> One example that was brought up is a situation where the miners might try to hard fork more inflation to bitcoin. Majority hash leaving the legacy chain and no replay protection would hinder the UTXO holders ability to express their economic preference safely and effectively.*

-------------------------

ajtowns | 2026-08-09 23:15:57 UTC | #2

I think a block height and a hash suffix would probably be superior; if there's already been a split, you can specify which side of the fork with perhaps 5 or 6 bytes that way instead of 32 just by picking a height where the two block hashes end in different digits.

Some prior references:

https://gnusha.org/pi/bitcoindev/20190508044928.z52oaxevwcppkvna@erisian.com.au/

https://github.com/ariard/bips/blob/9dc3f74b384f143b7f1bdad30dc0fe2529c8e63f/bip-annex.mediawiki

-------------------------

moonsettler | 2026-08-10 00:32:24 UTC | #3

Block height makes things easier, and I assume people would like to know if coins they receive have such condition, so it's probably a good idea for nodes to annotate the UTXOs with the closest immediate or ancestral block commitment.

While such commitment does not change the security meaning of "confirmations", people would probably still like to know how deep a reorg invalidates a transaction they received.

There could be a limit determined for both new commitment depth (roughly assumevalid height?) and how long nodes track such dependencies (discard after committed block is 100 deep buried?). To discourage misuse by performative commitments to past forks and significant events...

-------------------------

garlonicon | 2026-08-10 07:15:12 UTC | #4

> This was an interesting lesson learned from recent events where people tried to ancestrally split their coins, but it has proven to be fairly difficult.

Some attempts were successful, for example: https://bitcointalk.org/index.php?topic=5590674.msg67025339#msg67025339

I guess if anyone will split any coins successfully, then other people can just reuse it, to split theirs. Using P2PK is not that difficult, you just needed to be fast, because BIP-110 mined only two blocks. Also, if you use CoinJoin, then one splitted coin can split all of them, for the rest of users.

> Thoughts?

I think Satoshi intentionally designed transactions in a way, where they could end up on both chains, so many users would be unaffected by any forks. Nowadays, people want to fork their coins instead, which is kinda sad, and shows, that the community is now more splitted, than it was in the past. Ideally, we would have just 21 million coins, and people could peg them in and out to any subnetworks they want, but instead, we have just a lot of altcoins in practice.

> the miners might try to hard fork more inflation to bitcoin

They always can. I can launch an altcoin tomorrow, which will give you any coins, for any successful attempt of double-spending BTCs. Developers probably shouldn't try to protect users from all possible risks, because there are just too many ways of designing a harmful altcoin, and it would be just like fighting with the spam: yet another cat-and-mouse game.

> I think a block height and a hash suffix would probably be superior

I agree. Also, splitting coins through newly minted coinbase transactions is even better, and it will happen anyway. Most users should just transact normally, and pick their side, when they will be ready; in this way they will have their coins on both sides, and trade accordingly later.

By default, Bitcoin Core uses the latest block height to timelock new transactions. This alone is sufficient to split coins into BIP-110 and non-BIP-110 ones.

-------------------------

moonsettler | 2026-08-10 13:16:00 UTC | #5

[quote="garlonicon, post:4, topic:2792"]
Also, splitting coins through newly minted coinbase transactions is even better, and it will happen anyway.
[/quote]

I disagree here, takes a 100 blocks for those to be spendable. It's also magnitudes harder to coordinate the splits, if it requires downloading specific software and people trusting it with their keys, than if each individual can just act on their own with their usual setup. In an attack serious enough for holders to act, the hashrate is split in favor of the attacking chain, time locks would be useless (they also do not guarantee replay protection technically, highly unsafe).

-------------------------

garlonicon | 2026-08-10 08:11:12 UTC | #6

> takes a 100 blocks for those to be spendable

Which is good. Most users shouldn't rush to split their coins, if they don't know, what they are doing, and which side is going to win. Bitcoin was intentionally designed, to include transactions in each fork, because that's what protects most users from picking the wrong side.

> It’s also magnitudes harder to coordinate to splits if it requires downloading specific software and trusting it with their keys

You don't need to download Bitcoin Knots, to split coins into BIP-110 and non-BIP-110 ones. You can just sign everything from Bitcoin Core, and broadcast two different versions of your transaction to two different networks. They have different relay rules anyway. And if you send your coins to yourself, then you can lose only fees, and only if some miner would actively try to unsplit your coins, by mining BIP-110 invalid transaction on a BIP-110 chain.

> In an attack serious enough for holders to act, the hashrate is split in favor of the attacking chain, time locks would be useless

Splits are unlikely to be 50/50. And if it is 10/90 or 1/99, then you can always use a timelock, relative to the faster chain, wherever it would be. If you split your coin on one chain, and it will be confirmed, then you can safely broadcast it on another chain. When both chains will have conflicting transactions, then they will never unite again, unless someone will try to do a chain reorg.

Edit: One more thing: people tend to think about bright sides of their inventions, for example: if you make a transaction, valid only on a given chain, then you can split your coins properly, and sell altcoins for BTCs. However, exactly the same code can be used in a harmful way: to prepare an incentive, to confirm some double-spend, or to mine a malicious chain instead. And for that reason alone, I think it shouldn't be introduced, if it could be abused too easily.

-------------------------

moonsettler | 2026-08-10 13:17:38 UTC | #7

Alright, you made your "points", let's hear from people who understood the assignment!

-------------------------

ajtowns | 2026-08-11 04:23:44 UTC | #8

[quote="moonsettler, post:3, topic:2792"]
I assume people would like to know if coins they receive have such condition, so it’s probably a good idea for nodes to annotate the UTXOs with the closest immediate or ancestral block commitment.
[/quote]

I was figuring it'd require something similar to the coinbase maturity constraint; so if you have a tx that requires "block 961632 must end in 0xba", then that tx must also have an explicit nLockTime preventing it from being mined until ten blocks later, ie block 961642, say. See also the related discussion regarding [input triggered transaction expiry](https://delvingbitcoin.org/t/input-triggered-transaction-expiry/2667).

[quote="moonsettler, post:3, topic:2792"]
There could be a limit determined for both new commitment depth (roughly assumevalid height?) and how long nodes track such dependencies (discard after committed block is 100 deep buried?). To discourage misuse by performative commitments to past forks and significant events…
[/quote]

I think performative commitments should be fine; it's just a matter of looking up a header's hash by height and doing a substr comparison. I believe the "nlocktime must be commitment height + X" rule would already force the tx out of the mempool if a reorg got within X blocks of invalidating the tx.

-------------------------

gmaxwell | 2026-08-15 05:10:11 UTC | #9

Protecting reorg safety has been a historical goal, and a pretty good one:  generally we should avoid transactions just being invalidated due to internal network churn-- casual random reorgs cannot cause permanent funds loss in Bitcoin absent doublespend fraud by the participants.  One potential option would be to require the transaction be hight-locked and the referenced block be some offset back from there, so that that the degree of reorg unsafeness is bounded. The natural number would be 100 blocks, which is the current horizon for unsafety for generated coins.

Perhaps the current popular example might be used to argue against 100 blocks since one side can't seem to manage that many.  But I think this isn't a good argument: the hostile side can do anything they want including relaxing this rule.  And in the current examples the bipcoiners first intentionally launched without replay protection and now are implementing it but intentionally opt-in and one-sided, so I think it is clear that they would just relax this rule (e.g. make it pass for any value) because they expressly intend to disrupt the commerce of anyone not using their alternative.

I don't think the fact that forkers are hostile in this current instance and could generally be assumed to be is necessarily a reason to not do something like this-- as it can still be useful if it only applies on Bitcoin.  But I think perhaps it is a reason why preserving the current amount of reorg safeness is probably not an actual loss.

-------------------------

