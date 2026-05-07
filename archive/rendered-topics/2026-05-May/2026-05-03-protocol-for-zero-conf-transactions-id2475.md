# Protocol for zero-conf transactions

RobinLinus | 2026-05-03 04:12:15 UTC | #1

Here’s a writeup of a protocol for zero-conf transactions using a co-signing server trusted by the recipient.

It’s similar in spirit to the 2FA mechanism implemented in Blockstream Green, but uses a connector output so the recovery timelock only starts when unilateral exit is initiated. This means long-lived UTXOs don’t need periodic timelock refreshes/redeposits.

https://gist.github.com/RobinLinus/b9fcbb0e5de19ec6b9a80aa848a94253

-------------------------

Laz1m0v | 2026-05-03 19:40:49 UTC | #2

[quote="RobinLinus, post:1, topic:2475"]
Here’s a writeup of a protocol for zero-conf transactions using a co-signing server trusted by the recipient.

It’s similar in spirit to the 2FA mechanism implemented in Blockstream Green, but uses a connector output so the recovery timelock only starts when unilateral exit is initiated. This means long-lived UTXOs don’t need periodic timelock refreshes/redeposits.
[/quote]

@robin_linus *'A co-signing server trusted by the recipient.'*

Congratulations, you just reinvented PayPal using a 2-of-2 multisig.

You are asking users to beg a server for 'pre-signed change exits', praying the server doesn't equivocate, and calling it a protocol. It's not a protocol. It's an interpretive, permissioned spreadsheet.

While you are doing interactive off-chain gymnastics to simulate safety, we are enforcing absolute L1 bivalence. With TUSM, there is no 'trusted co-signer' enforcing policy. The math enforces the policy. The Taproot address *is* the state. The covenant *is* the consensus.

If your asset needs a human-maintained server to prevent a double-spend, you don't have an asset. You have an illusion. `.simf` fixes this. Simplicity always win

-------------------------

evd0kim | 2026-05-03 19:58:41 UTC | #3

Thank you bookmarked.

I was thinking about connectors while working on draft ideas https://delvingbitcoin.org/t/zk-statechains-without-states/2166 and a prototype implementation https://delvingbitcoin.org/t/ciphera-a-bitcoin-zk-appchain-implementation/2285. While I choose to work on [Blind Vaults version](https://delvingbitcoin.org/t/building-a-vault-using-blinded-co-signers/2141)  - I also reused parts of the demo code - i was keeping in mind connectors might be very effective in revoking states, instead of schemes used in LN, but didn't have capacity to do this research. This post helps very much to fill gaps.

-------------------------

polespinasa | 2026-05-07 09:00:42 UTC | #4

Haven't read the full write-up, but just for the tittle it remained me of https://link.springer.com/article/10.1007/s10207-018-0422-4, maybe you would like to take a look at it if you didn't know about it.

-------------------------

