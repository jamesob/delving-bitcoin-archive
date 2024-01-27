# Disclosure: Btcd consensus bugs due to usage of signed transaction version

dergoegge | 2024-01-22 14:56:46 UTC | #1

Btcd prior to version [v0.24.0](https://github.com/btcsuite/btcd/releases/tag/v0.24.0) does not correctly implement the consensus rules outlined in [BIP 68](https://github.com/bitcoin/bips/blob/deae64bfd31f6938253c05392aa355bf6d7e7605/bip-0068.mediawiki) and [BIP 112](https://github.com/bitcoin/bips/blob/deae64bfd31f6938253c05392aa355bf6d7e7605/bip-0112.mediawiki), making it susceptible to consensus failures. Btcd users are advised to upgrade to version v0.24.0 or above.

# Details

BIP 68 & 112 describe two soft-forks that introduced relative time locks to Bitcoin transactions (necessary for e.g. [Hash Time Locked Contracts](https://bitcoinops.org/en/topics/htlc/)). The rules outlined in the BIPs are only active for transactions with version 2 or higher.

While both Bitcoin Core and btcd store the transaction version as a **signed** 32-bit integer, in the context of BIP 68 & 112 it is supposed to be treated as **unsigned** (otherwise half the range of the version would not support BIP 68 & 112). Btcd however, used the signed transaction version in both the [BIP 68](https://github.com/btcsuite/btcd/blob/e4c88c3a3ecb1813529bf3dddc7a865bd418a6b8/blockchain/chain.go#L383C1-L392C3) and [BIP 112](https://github.com/btcsuite/btcd/blob/e4c88c3a3ecb1813529bf3dddc7a865bd418a6b8/txscript/opcode.go#L1172C1-L1178C3) logic without a prior cast to `uint32`. As consequence, transactions with negative versions are incorrectly treated as not enforcing the BIP 68 rules or incorrectly rejected for use of `OP_CHECKSEQUENCEVERIFY` (BIP 112).

# Impact

If triggered, these bugs can result in btcd nodes not accepting a block that Bitcoin Core nodes would accept (or vice versa), resulting in a chain split. Chain splits can lead to e.g.:

- Lightning Nodes using btcd as their chain backend are at risk of losing funds due to not receiving updates for the canonical chain.
- An attacker could trigger a split and then mine on the “btcd chain” (likely without competition from other miners) to trick btcd users into accepting payments that can’t occur on the canonical chain.
- Miners that use btcd are at risk of mining on top of an invalid chain, wasting their resources.

Transactions with negative versions are non-standard but as the recent past has shown, that would not represent a significant hurdle for an attacker.

# Credits

Thanks to the btcd project for awarding me with a bug bounty reward of 0.023 BTC and thanks to [Guido Vranken](https://twitter.com/GuidoVranken) for suggesting differential fuzzing of btcd’s and Bitcoin Core’s script interpreter to me.

# Timeline

- 22-05-2023 - Initial disclosure to Lightning Labs
- 21-06-2023 - Fixed merged into btcd (https://github.com/btcsuite/btcd/pull/1981)
- 31-12-2023 - btcd v0.24.0 released
- 22-01-2024 - Public disclosure

---

Support security-focused Bitcoin research and development by [donating to Brink](https://brink.dev/donate).

-------------------------

Chris_Stewart_5 | 2024-01-22 14:59:12 UTC | #2

Here is a PR to add a static test vector that tests this logic in bitcoin core:

https://github.com/bitcoin/bitcoin/pull/29291

Here is what the patch looks like to fix this in bitcoin-s:

https://github.com/bitcoin-s/bitcoin-s/pull/5346

-------------------------

Davidson | 2024-01-22 22:26:32 UTC | #3

Congratulations on this finding! Very interesting work.

[quote="dergoegge, post:1, topic:455"]
Thanks to the btcd project for awarding me with a bug bounty reward of 0.023 BTC and thanks to [Guido Vranken ](https://twitter.com/GuidoVranken) for suggesting differential fuzzing of btcd’s and Bitcoin Core’s script interpreter to me.
[/quote]

It is my understanding is that you got this bug by using this fuzzing. Is this OSS?

-------------------------

dergoegge | 2024-01-22 23:04:10 UTC | #4

[quote="Davidson, post:3, topic:455"]
It is my understanding is that you got this bug by using this fuzzing. Is this OSS?
[/quote]

Yes, this was found through differential fuzzing. The harness isn't public yet but I'm planning on publishing a write up on the discovery (including the code) soon.

-------------------------

0xB10C | 2024-01-27 00:16:18 UTC | #5

It seems like this was exploited on testnet (a day after being published here) in block 000000002f4830471b6b346578546615c031b99da5e7fabeac119b63f1843f82 with [5839f20446d7b9446e82c00117ee3699fa84154e970d57f09add60deef2eaa18](https://mempool.space/testnet/tx/5839f20446d7b9446e82c00117ee3699fa84154e970d57f09add60deef2eaa18). I tried to sync a btcd v0.23.4 node on testnet, which seems stuck on 2575398. A btcd v0.24.0 node is fine.


![image|690x272](upload://aHjsVhhMvbQqgBh5lTT9YgmXAHF.png)

According to https://forkmonitor.info/nodes/btc, their mainnet btcd v0.23.3 node is still fine. I also haven't seen a non-standard transaction exploiting this on mainnet in the past days.

-------------------------

