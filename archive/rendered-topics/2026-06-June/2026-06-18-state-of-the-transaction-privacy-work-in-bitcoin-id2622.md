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

