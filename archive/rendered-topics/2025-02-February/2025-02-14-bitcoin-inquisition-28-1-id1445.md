# Bitcoin Inquisition 28.1

ajtowns | 2025-02-14 07:38:36 UTC | #1

Bitcoin Inqusition 28.1 is now available from

https://github.com/bitcoin-inquisition/bitcoin/releases/tag/v28.1-inq

This is based on [Bitcoin Core 28.1](https://bitcoincore.org/en/releases/28.1/).

Along with the support for TRUC and anchor relay and full replace by fee behaviour from Bitcoin Core 28.0, this release also includes backported support for ephemeral dust (relay of txs with low or zero valued outputs that are immediately spent; [PR#69](https://github.com/bitcoin-inquisition/bitcoin/pull/69)). This allows for the creation of pre-signed transactions where the fees are fully paid for by a child transaction that is signed at a later date, see txs [7c1b4d](https://mempool.space/signet/tx/7c1b4d9c5c7bb411bc9c8742c7dda1d06c68655ef47585c8b58a8a314fcdd6ef#vout=1) and [5107bb](https://mempool.space/signet/tx/5107bb639ce389234e1da21a0a60d7c05ca1cf0d9a904e98c83ee0e50a597c8d#vin=0), for example.

This includes support for the following proposed consensus changes:
 * [BIP 119](https://github.com/bitcoin/bips/blob/584f4a732ba94199fc097fbb9b4660db868dd712/bip-0119.mediawiki) OP_CHECKTEMPLATEVERIFY ([PR#66](https://github.com/bitcoin-inquisition/bitcoin/pull/66))
 * [BIP 118](https://github.com/bitcoin/bips/blob/584f4a732ba94199fc097fbb9b4660db868dd712/bip-0118.mediawiki) SIGHASH_ANYPREVOUT  ([PR#67](https://github.com/bitcoin-inquisition/bitcoin/pull/67))
 * [BIN-2024-1](https://github.com/ajtowns/binana/blob/8264328e6c7fd9e9f30efb8273fb94700f001454/2024/BIN-2024-0001.md) aka [BIP 347](https://github.com/bitcoin/bips/blob/740e826c19391a7a290933f514c15518e00780f0/bip-0347.mediawiki) OP_CAT ([PR#68](https://github.com/bitcoin-inquisition/bitcoin/pull/68))

The first two of these have been active on the default signet since block 106704 (2022-09-06), and the third since block 193536 (2024-04-30).

Note that transactions making use of any of these new features will not be relayed by nodes that don't support these features, so if you'd like your transaction to get to a miner, you'll need to peer with other inquisition nodes. An easy way to do this is to specify `addnode=inquisition.bitcoin-signet.net`. By default, Bitcoin Inquisition nodes will advertise themselves via the subversion string, which can be viewed with `bitcoin-cli -netinfo 3`. See the [23.0 announcement](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-December/021275.html) for more details.

-------------------------

