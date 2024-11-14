# Bitcoin Core 27.2 Released

fanquake | 2024-11-04 10:50:46 UTC | #1

Bitcoin Core version 27.2 is now available from:

  <https://bitcoincore.org/bin/bitcoin-core-27.2/>

Or through BitTorrent:

[magnet:?xt=...](magnet:?xt=urn:btih:f21febdf8c54d2a9b09ed54f7eebb909537fb7b0&dn=bitcoin-core-27.2&tr=udp%3A%2F%2Ftracker.openbittorrent.com%3A80&tr=udp%3A%2F%2Ftracker.opentrackr.org%3A1337%2Fannounce&tr=udp%3A%2F%2Ftracker.coppersurfer.tk%3A6969%2Fannounce&tr=udp%3A%2F%2Ftracker.leechers-paradise.org%3A6969%2Fannounce&tr=udp%3A%2F%2Fexplodie.org%3A6969%2Fannounce&tr=udp%3A%2F%2Ftracker.torrent.eu.org%3A451%2Fannounce&tr=udp%3A%2F%2Ftracker.bitcoin.sprovoost.nl%3A6969&ws=http%3A%2F%2Fbitcoincore.org%2Fbin%2F)

This release includes various bug fixes and performance
improvements, as well as updated translations.

Please report bugs using the issue tracker at GitHub:

  <https://github.com/bitcoin/bitcoin/issues>

To receive security and update notifications, please subscribe to:

  <https://bitcoincore.org/en/list/announcements/join/>

How to Upgrade
==============

If you are running an older version, shut it down. Wait until it has completely
shut down (which might take a few minutes in some cases), then run the
installer (on Windows) or just copy over `/Applications/Bitcoin-Qt` (on macOS)
or `bitcoind`/`bitcoin-qt` (on Linux).

Upgrading directly from a version of Bitcoin Core that has reached its EOL is
possible, but it might take some time if the data directory needs to be migrated. Old
wallet versions of Bitcoin Core are generally supported.

Compatibility
==============

Bitcoin Core is supported and extensively tested on operating systems
using the Linux Kernel 3.17+, macOS 11.0+, and Windows 7 and newer. Bitcoin
Core should also work on most other Unix-like systems but is not as
frequently tested on them. It is not recommended to use Bitcoin Core on
unsupported systems.

Notable changes
===============

### P2P

- [#30394](https://github.com/bitcoin/bitcoin/pull/30394) net: fix race condition in self-connect detection

### Init

- [#30435](https://github.com/bitcoin/bitcoin/pull/30435) init: change shutdown order of load block thread and scheduler

### RPC

- [#30357](https://github.com/bitcoin/bitcoin/pull/30357) Fix cases of calls to FillPSBT errantly returning complete=true

### PSBT

- [#29855](https://github.com/bitcoin/bitcoin/pull/29855) psbt: Check non witness utxo outpoint early

### Test

- [#30552](https://github.com/bitcoin/bitcoin/pull/30552) test: fix constructor of msg_tx

### Doc

- [#30504](https://github.com/bitcoin/bitcoin/pull/30504) doc: use proper doxygen formatting for CTxMemPool::cs

### Build

- [#30283](https://github.com/bitcoin/bitcoin/pull/30283) upnp: fix build with miniupnpc 2.2.8
- [#30633](https://github.com/bitcoin/bitcoin/pull/30633) Fixes for GCC 15 compatibility

### CI

- [#30193](https://github.com/bitcoin/bitcoin/pull/30193) ci: move ASan job to GitHub Actions from Cirrus CI
- [#30299](https://github.com/bitcoin/bitcoin/pull/30299) ci: remove unused bcc variable from workflow

Credits
=======

Thanks to everyone who directly contributed to this release:

- Ava Chow
- Cory Fields
- Martin Zumsande
- Matt Whitlock
- Max Edwards
- Sebastian Falbesoner
- Vasil Dimov
- willcl-ark

As well as to everyone that helped with translations on
[Transifex](https://www.transifex.com/bitcoin/bitcoin/).

-------------------------
