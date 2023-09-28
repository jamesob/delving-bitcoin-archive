# Covenant tools softfork

jamesob | 2023-09-28 18:38:38 UTC | #1

I'd like to propose a softfork deployment that activates the consensus changes outlined in

- [BIP-118](https://github.com/bitcoin/bips/blob/7004ad1a825a0422b78bbf1a96bf748d5e380569/bip-0118.mediawiki) (`SIGHASH_ANYPREVOUT` for Taproot Scripts)
- [BIP-119](https://github.com/bitcoin/bips/blob/7004ad1a825a0422b78bbf1a96bf748d5e380569/bip-0119.mediawiki) (`CHECKTEMPLATEVERIFY`)
- [BIP-345](https://github.com/jamesob/bips/blob/4aae726be9610a675b362e66f539ce0d5f903a5f/bip-0345.mediawiki) (`OP_VAULT`)

These changes make possible a number of use-cases that are broadly beneficial to users of Bitcoin, including

- [vaults](https://bitcoinops.org/en/topics/vaults/) (reactive custodial security),
- [LN-Symmetry](https://bitcoinops.org/en/topics/eltoo/),
- efficient implementations of [DLCs](https://bitcoinops.org/en/topics/discreet-log-contracts/),
- [non-interactive channel openings](https://utxos.org/uses/non-interactive-channels/),
- [congestion control](https://utxos.org/uses/scaling/),
- decentralized mining pools (via [CTV compression in coinbase payouts](https://utxos.org/uses/miningpools/)),
- various [Lightning efficiency improvements](https://twitter.com/roasbeef/status/1692589689939579259),
- using [covenant based timeout-trees](https://bitcoinops.org/en/newsletters/2023/09/27/) to scale Lightning, and more generally enabling [channel factories](https://bitcoinops.org/en/topics/channel-factories/).

We also see that many speculative scaling solutions (e.g. [Ark](https://arkpill.me), [Spacechains](https://gist.github.com/RubenSomsen/c9f0a92493e06b0e29acced61ca9f49a#spacechains)) require locking coins to be spent to a particular set of outputs without any additional authorization (i.e. CTV, or APO's emulation of it).

---

The patches for BIP-118 and BIP-119 have long been stable and well scrutinized; they each haven't changed in some time.

BIP-345 (OP_VAULT) is the newest of the three by a good margin, and originally I was going to exclude it from this proposal. But after a number of discussions, it became clear that BIP-345 may be the most immediately usable of the three in terms of a major use-case: 
- LN-symmetry (the major use of ANYPREVOUT) will require a good amount of time to coordinate deployment, and 
- CTV, while an important primitive, has mostly niche direct uses (DLC efficiency, possible decentralized mining pools, congestion control, ...) outside of BIP-345-style vaults.

Vaults with BIP-345, on the other hand, would be delayed only by the speed of wallet implementers. [Example wallet implementations](https://github.com/jamesob/opvault-demo/) already exist, and the appetite for safer modes of custody is almost universally present among both industrial and individual users.

The implementation for these consensus changes is only about [7,000 lines](https://github.com/bitcoin/bitcoin/compare/master...jamesob:bitcoin:2023-09-covtools-softfork?), and that includes comprehensive testing. The limited nature of these changes relative to the last two softforks that we've had in Bitcoin give me some comfort in proposing the relatively young code for BIP-345 -- with focused effort, the limited line count and tight scope will make it a tractable deployment to review.

I will be opening this branch as a draft PR in the Core repo.

The activation mechanism is currently drafted as the same modified version of BIP-9 [described in BIP-341](https://en.bitcoin.it/wiki/BIP_0341#Deployment), but I am agnostic in this department. There is currently no signaling period specified - we will wait until further indications of consensus before choosing specific dates.

What do you think?

-------------------------

jamesob | 2023-09-28 18:43:36 UTC | #2

The related Bitcoin Core pull request is here: https://github.com/bitcoin/bitcoin/pull/28550

-------------------------

