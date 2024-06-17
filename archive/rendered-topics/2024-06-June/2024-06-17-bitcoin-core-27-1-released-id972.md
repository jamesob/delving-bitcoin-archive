# Bitcoin Core 27.1 Released

fanquake | 2024-06-17 15:15:45 UTC | #1

Bitcoin Core version v27.1 is now available from:

[https://bitcoincore.org/bin/bitcoin-core-27.1/](https://bitcoincore.org/bin/bitcoin-core-27.1/)

Or through BitTorrent:

[magnet:?xt=...](`magnet:?xt=urn:btih:39554dd8aae5688f462538a821d2227ea1b8cbb6&dn=bitcoin-core-27.1&tr=udp%3A%2F%[2Ftracker.openbittorrent.com](http://2Ftracker.openbittorrent.com)%3A80&tr=udp%3A%2F%[2Ftracker.opentrackr.org](http://2Ftracker.opentrackr.org)%3A1337%2Fannounce&tr=udp%3A%2F%[2Ftracker.coppersurfer.tk](http://2Ftracker.coppersurfer.tk)%3A6969%2Fannounce&tr=udp%3A%2F%[2Ftracker.leechers-paradise.org](http://2Ftracker.leechers-paradise.org)%3A6969%2Fannounce&tr=udp%3A%2F%2Fexplodie.org%3A6969%2Fannounce&tr=udp%3A%2F%[2Ftracker.torrent.eu.org](http://2Ftracker.torrent.eu.org)%3A451%2Fannounce&tr=udp%3A%2F%[2Ftracker.bitcoin.sprovoost.nl](http://2Ftracker.bitcoin.sprovoost.nl)%3A6969&ws=http%3A%2F%2Fbitcoincore.org%2Fbin%2F)

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
possible, but it might take some time if the data directory needs to
be migrated. Old
wallet versions of Bitcoin Core are generally supported.

Compatibility
==============

Bitcoin Core is supported and extensively tested on operating systems
using the Linux Kernel `3.17+`, macOS `11.0+`, and Windows `7` and newer. Bitcoin
Core should also work on most other Unix-like systems but is not as
frequently tested on them. It is not recommended to use Bitcoin Core on
unsupported systems.

Notable changes
===============

### Miniscript

- [#29853](https://github.com/bitcoin/bitcoin/pull/29776) sign: don't assume we are parsing a sane TapMiniscript

### RPC

- [#29869](https://github.com/bitcoin/bitcoin/pull/29869) rpc, bugfix: Enforce maximum value for setmocktime
- [#29870](https://github.com/bitcoin/bitcoin/pull/29870) rpc: Reword SighashFromStr error message
- [#30094](https://github.com/bitcoin/bitcoin/pull/30094) rpc: move UniValue in blockToJSON

### Index

- [#29776](https://github.com/bitcoin/bitcoin/pull/29776) Fix #29767, set m_synced = true after Commit()

### Gui

- [#gui812](https://github.com/bitcoin-core/gui/pull/812) Fix create unsigned transaction fee bump
- [#gui813](https://github.com/bitcoin-core/gui/pull/813) Don't permit port in proxy IP option

### Test

- [#29892](https://github.com/bitcoin/bitcoin/pull/29892) test: Fix failing univalue float test

### P2P

- [#30085](https://github.com/bitcoin/bitcoin/pull/30085) p2p: detect addnode cjdns peers in GetAddedNodeInfo()

### Build

- [#29747](https://github.com/bitcoin/bitcoin/pull/29747) depends: fix mingw-w64 Qt DEBUG=1 build
- [#29859](https://github.com/bitcoin/bitcoin/pull/29859) build: Fix false positive CHECK_ATOMIC test
- [#29985](https://github.com/bitcoin/bitcoin/pull/29985) depends: Fix build of Qt for 32-bit platforms with recent glibc
- [#30097](https://github.com/bitcoin/bitcoin/pull/30097) crypto: disable asan for sha256_sse4 with clang and -O0
- [#30151](https://github.com/bitcoin/bitcoin/pull/30151) depends: Fetch miniupnpc sources from an alternative website
- [#30216](https://github.com/bitcoin/bitcoin/pull/30216) build: Fix building fuzz binary on on SunOS / illumos
- [#30217](https://github.com/bitcoin/bitcoin/pull/30217) depends: Update Boost download link

### Doc

- [#29934](https://github.com/bitcoin/bitcoin/pull/29934) doc: add LLVM instruction for macOS < 13

### CI

- [#29856](https://github.com/bitcoin/bitcoin/pull/29856) ci: Bump s390x to ubuntu:24.04

### Misc

- [#29691](https://github.com/bitcoin/bitcoin/pull/29691) Change Luke Dashjr seed to [dashjr-list-of-p2p-nodes.us](http://dashjr-list-of-p2p-nodes.us)
- [#30149](https://github.com/bitcoin/bitcoin/pull/30149) contrib: Renew Windows code signing certificate

Credits
=======

Thanks to everyone who directly contributed to this release:

- Antoine Poinsot
- Ava Chow
- Cory Fields
- dergoegge
- fanquake
- furszy
- Hennadii Stepanov
- Jon Atack
- laanwj
- Luke Dashjr
- MarcoFalke
- nanlour
- Sjors Provoost
- willcl-ark

As well as to everyone that helped with translations on
[Transifex](https://www.transifex.com/bitcoin/bitcoin/).

-------------------------

