# State of the transaction privacy work in Bitcoin

nkaretnikov | 2026-06-18 21:27:51 UTC | #1

I would like to better understand what's happening around transaction privacy in Bitcoin.

My perception is that it's all happening in sidechains, L2s, or requires wallet support.
Examples: Confidential Transactions (sidechain), Silent Payments (wallet), PayJoin (wallet), CoinJoin (wallet).
There is also related work, like Cross-Input Signature Aggregation, that would make CoinJoin cheaper, but isn't a privacy feature by itself.

I assume people moved on from L1 privacy work for several reasons:

* Lightning is popular, and offers some privacy that many consider sufficient
* Implementation complexity, plus a focus on transparency and a fixed supply (see past inflation/counterfeiting bugs in privacy-coin implementations)
* Concerns about regulatory pressure if transactions become untraceable.

Is this now mostly a wallet-layer concern, or are there protocol-level gaps worth filling (like CISA)?

Anything else with a realistic shot at adoption that I've missed?

-------------------------

nkaretnikov | 2026-06-20 19:58:17 UTC | #2

It seems there's ongoing work on CoinSwap.

Summary of the protocol:
https://opensats.org/blog/developing-advancements-in-onchain-privacy
https://opensats.org/topics/coinswap

An actively developed implementation:
https://github.com/citadel-tech/coinswap
https://github.com/citadel-tech/coinswap/milestone/1

Protocol spec:
https://github.com/citadel-tech/Coinswap-Protocol-Specification
https://bitcointalk.org/index.php?topic=321228.0

An older PoC from Chris Belcher:
https://github.com/bitcoin-teleport/teleport-transactions

-------------------------

nkaretnikov | 2026-06-20 20:23:14 UTC | #3

Physical devices that allow you to pass Bitcoin offline, like cash. Not a protocol, but worth mentioning.
https://opendime.com/
https://satscard.com/

-------------------------

light | 2026-06-20 20:24:25 UTC | #4

Shielded CSV: https://eprint.iacr.org/2025/068 

Needs a soft fork that supports building a trustless bridge (e.g. BIP-441) in order to be used trustlessly with BTC.

-------------------------

cmp_ancp | 2026-06-20 20:54:41 UTC | #5

There is an interesting project concerning coinjoin coordination in nostr, making it descentralized and avoiding regulatory pressures.

joinstr https://share.google/MhV0i0CeMvv1x0RnB

Also, you commented about LN superficially, but there are different concepts and protocols inside LN that aren't broad used or implemented today and can bring more privacy in the future. Taproot channels, PTLCs, maybe a gossip that doesn't doxx UTXOs, LN over Ark, blinded payments over BOLT12 (maybe the most important BOLT12 feature in respect to privacy and one of the least implemented among wallets), trampoline nodes, splices, LSP spec, neutrino and filter based sync tech, etc.

LN over Ark may be the UX killer, resolving the inbound liquidity problem and cheapenning multiple channel maintanance (something that increases privacy).

-------------------------

