# Bitcoin Inquisition 29.4

ajtowns | 2026-07-22 23:16:16 UTC | #1

Bitcoin Inquisition 29.4 is now available from

https://github.com/bitcoin-inquisition/bitcoin/releases/tag/v29.4-inq

This is based on [Bitcoin Core 29.4](https://github.com/bitcoin/bitcoin/releases/tag/v29.4/).

This includes support for the following proposed consensus changes:
 * [BIP 118](https://github.com/bitcoin/bips/blob/584f4a732ba94199fc097fbb9b4660db868dd712/bip-0118.mediawiki) SIGHASH_ANYPREVOUT  ([PR#84](https://github.com/bitcoin-inquisition/bitcoin/pull/84))
 * [BIP 119](https://github.com/bitcoin/bips/blob/584f4a732ba94199fc097fbb9b4660db868dd712/bip-0119.mediawiki) OP_CHECKTEMPLATEVERIFY ([PR#85](https://github.com/bitcoin-inquisition/bitcoin/pull/85))
 * [BIN-24-1](https://github.com/ajtowns/binana/blob/8264328e6c7fd9e9f30efb8273fb94700f001454/2024/BIN-2024-0001.md) aka [BIP 347](https://github.com/bitcoin/bips/blob/740e826c19391a7a290933f514c15518e00780f0/bip-0347.mediawiki) OP_CAT ([PR#83](https://github.com/bitcoin-inquisition/bitcoin/pull/83))
 * [BIN-24-3](https://github.com/bitcoin-inquisition/binana/blob/3477736b354a8523615695971c6250cc4f98d452/2024/BIN-2024-0003.md) aka BIP 348 OP_CHECKSIGFROMSTACK ([PR#87](https://github.com/bitcoin-inquisition/bitcoin/pull/87))
 * [BIN-24-4](https://github.com/bitcoin-inquisition/binana/blob/3477736b354a8523615695971c6250cc4f98d452/2024/BIN-2024-0004.md) aka BIP 349 OP_INTERNALKEY ([PR#86](https://github.com/bitcoin-inquisition/bitcoin/pull/86))
 * [BIP 54](https://github.com/bitcoin/bips/blob/ed7af6ae7e80c90bcfc69b3936073505e2fc2503/bip-0054.md) Consensus Cleanup ([PR#90](https://github.com/bitcoin-inquisition/bitcoin/pull/90), [PR#99](https://github.com/bitcoin-inquisition/bitcoin/pull/99))
 * [BIP 446](https://github.com/bitcoin/bips/blob/8c369ac8e60629ac6c032ffe21bb5ec5b35213d7/bip-0446.md) TEMPLATEHASH ([PR#100](https://github.com/bitcoin-inquisition/bitcoin/pull/100))

The first two of these have been active on the default signet since block 106704 (2022-09-06), the third since block 193536 (2024-04-30), and the fourth and fifth have been active since block 272592 (2025-10-06), and the sixth has been active since block 291168 (2026-02-12).

The seventh has been triggered as of block 314315 and should be active as of block 314928 (around 2026-07-26).

Note that transactions making use of these new features will not be relayed by nodes that don't support these features, so if you'd like your transaction to get to a miner, you'll need to peer with other inquisition nodes. An easy way to do this is to specify `addnode=inquisition.bitcoin-signet.net`. By default, Bitcoin Inquisition nodes will advertise themselves via the subversion string, which can be viewed with `bitcoin-cli -netinfo 3`. See the [23.0 announcement](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-December/021275.html) for more details.

-------------------------

