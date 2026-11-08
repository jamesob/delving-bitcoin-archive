# Should wallets default to Taproot or SegWit receive addresses while some services still reject bc1p?

Anzus_GemWallet | 2026-11-08 13:33:13 UTC | #1

I work in BD and support at Gem Wallet rather than as a developer, so this comes from the support side.

A recurring case: a user copies their receive address, tries to withdraw from an exchange or another service, and gets "invalid address". The address is a valid Taproot (bc1p…) address; the sending service just doesn't accept it. From the user's point of view, the wallet gave them a broken address.

For wallet teams here:

1. Do you default to Taproot or SegWit (bc1q) for receiving, and what decided it?
2. If you offer both, where do you put the choice so people find it when they need it, without confusing everyone else?
3. Is there a reliable way to know which services accept bc1p, or is it still trial and error?

-------------------------

