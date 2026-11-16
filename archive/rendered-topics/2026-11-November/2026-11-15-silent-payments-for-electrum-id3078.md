# Silent Payments for Electrum

Zenul_Abidin | 2026-11-15 21:15:33 UTC | #1

A few months ago, I created a sending plugin for Silent Payments and published it on the Electrum plugins registry. Now, I’m back with a scanner for Silent Payments. It’s called spscand, and it works directly with the plugin.

* Code (MIT): ZenulAbidin/spscand on Github
* Latest version: v0.1.1 (GPG-signed)

It finds payments to your sp1 address (BIP352 1.1.2), tracks their spends and reorgs, and reports them through a local JSON API and a webhook.

It never asks a server about an address, output or transaction. You can get blocks from your own node, a blindbit server, or BIP352 filters. Tor works as well.

Feedback and critique are appreciated!

-------------------------

