# Protocol for zero-conf transactions

RobinLinus | 2026-05-03 04:12:15 UTC | #1

Here’s a writeup of a protocol for zero-conf transactions using a co-signing server trusted by the recipient.

It’s similar in spirit to the 2FA mechanism implemented in Blockstream Green, but uses a connector output so the recovery timelock only starts when unilateral exit is initiated. This means long-lived UTXOs don’t need periodic timelock refreshes/redeposits.

https://gist.github.com/RobinLinus/b9fcbb0e5de19ec6b9a80aa848a94253

-------------------------

