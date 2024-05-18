# Bitcoin Inquisition 27.0

ajtowns | 2024-05-18 07:24:11 UTC | #1

Bitcoin Inqusition 27.0 is now available from

https://github.com/bitcoin-inquisition/bitcoin/releases/tag/v27.0-inq

This is based on [Bitcoin Core 27.0](https://bitcoincore.org/en/releases/27.0/).

This includes support for the following proposed consensus changes:
 * [BIP 119](https://github.com/bitcoin/bips/blob/584f4a732ba94199fc097fbb9b4660db868dd712/bip-0119.mediawiki) OP_CHECKTEMPLATEVERIFY ([PR#55](https://github.com/bitcoin-inquisition/bitcoin/pull/55))
 * [BIP 118](https://github.com/bitcoin/bips/blob/584f4a732ba94199fc097fbb9b4660db868dd712/bip-0118.mediawiki) SIGHASH_ANYPREVOUT  ([PR#56](https://github.com/bitcoin-inquisition/bitcoin/pull/56))
 * [BIN-2024-1](https://github.com/ajtowns/binana/blob/8264328e6c7fd9e9f30efb8273fb94700f001454/2024/BIN-2024-0001.md) aka [BIP 347](https://github.com/bitcoin/bips/blob/740e826c19391a7a290933f514c15518e00780f0/bip-0347.mediawiki) OP_CAT ([PR#57](https://github.com/bitcoin-inquisition/bitcoin/pull/57))

The first two of these have been active on the default signet since block 106704 (2022-09-06), and the third since block 193536 (2024-04-30).

Note that transactions making use of any of these new features will not be relayed by nodes that don't support these features, so if you'd like your transaction to get to a miner, you'll need to peer with other inquisition nodes. An easy way to do this is to specify `addnode=inquisition.bitcoin-signet.net`. By default, Bitcoin Inquisition nodes will advertise themselves via the subversion string, which can be viewed with `bitcoin-cli -netinfo 3`. See the [23.0 announcement](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-December/021275.html) for more details.

This no longer includes relay support for annexdatacarrier (see [PR#22](https://github.com/bitcoin-inquisition/bitcoin/pull/22)) or pseudo ephemeral anchors (see [PR#23](https://github.com/bitcoin-inquisition/bitcoin/pull/23)). The signet inquisition miner and inquisition relay node mentioned above will likely be updated in the next week or two, at which point transactions using these features will no longer confirm.

-------------------------

