# Bitcoin Inquisition 25.2

ajtowns | 2024-04-27 02:51:43 UTC | #1

Bitcoin Inquisition 25.2 is now available from:

https://github.com/bitcoin-inquisition/bitcoin/releases/tag/v25.2-inq

This includes support for the following proposed consensus changes:
 * [BIP 119](https://github.com/bitcoin/bips/blob/584f4a732ba94199fc097fbb9b4660db868dd712/bip-0119.mediawiki) OP_CHECKTEMPLATEVERIFY ([PR#33](https://github.com/bitcoin-inquisition/bitcoin/pull/33))
 * [BIP 118](https://github.com/bitcoin/bips/blob/584f4a732ba94199fc097fbb9b4660db868dd712/bip-0118.mediawiki) SIGHASH_ANYPREVOUT  ([PR#34](https://github.com/bitcoin-inquisition/bitcoin/pull/34))
 * [BIN-2024-1](https://github.com/ajtowns/binana/blob/8264328e6c7fd9e9f30efb8273fb94700f001454/2024/BIN-2024-0001.md) aka [BIP 347](https://github.com/bitcoin/bips/pull/1525#issuecomment-2075518532) OP_CAT ([PR#39](https://github.com/bitcoin-inquisition/bitcoin/pull/39))

This is based on [Bitcoin Core 25.2](https://bitcoincore.org/en/releases/25.2/).

The first two of these have been active on the default signet since block 106704 (2022-09-06), and the third should become active in a few days at block 193536.

Note that transactions making use of any of these new features will not be relayed by nodes that don't support these features, so if you'd like your transaction to get to a miner, you'll need to peer with other inquisition nodes. An easy way to do this is to specify `addnode=inquisition.bitcoin-signet.net`. By default, Bitcoin Inquisition nodes will advertise themselves via the subversion string, which can be viewed with `bitcoin-cli -netinfo 3`.

See the [23.0 announcement](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-December/021275.html) for more details.

-------------------------

