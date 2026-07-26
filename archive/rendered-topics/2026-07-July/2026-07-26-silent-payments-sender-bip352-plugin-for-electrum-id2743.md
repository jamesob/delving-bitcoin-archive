# Silent Payments Sender - BIP352 plugin for Electrum

Zenul_Abidin | 2026-07-26 15:30:35 UTC | #1

Source code: https://github.com/ZenulAbidin/electrum-silent-payments-sender/

This is a plugin I made that adds support for BIP352 Silent Payments into Electrum desktop. Currently, it lets Electrum users send to Silent Payment addresses.

I have tested the plugin and made sure that it works with all supported types of addresses and transaction configurations in Electrum.

It integrates with Electrum's normal Send tab:

![](https://talkimg.com/images/2026/07/26/Uh9m7Z.png)

And it can be done all through software. Once you select inputs, this plugin devices Taproot output, then signs normally.

Features:

* Electrum 4.6.0+
* sender only; no scanning/receiving
* mainnet, testnet, signet, and regtest
* standard single-signature deterministic software wallets
* P2PKH, P2WPKH-P2SH, and P2WPKH inputs
* raw addresses and BIP21 `sp` payment instructions
* BIP352 1.1.1, including forward-compatible v1-v30 address parsing

Not supported yet / Todo:

* hardware, watch-only, imported-key, multisig, and 2FA wallets
* Taproot inputs, batching, payjoin, or swaps
* unsigned preview/export or later signing

It hasn't been audited obviously, so use carefully.

Installation instructions:

1. Go to [Releases](https://github.com/ZenulAbidin/electrum-silent-payments-sender/releases/), download silent_payments_sender-1.0.0.zip and verify the SHA256 hash.
2. Electrum -> Tools -> Plugins -> Add plugin.
3. Select silent_payments_sender-1.0.0.zip and enable it.
4. Use Electrum's normal Send tab.

I hope you guys find it useful. And don't forget to verify the file hash after you download.

-------------------------

