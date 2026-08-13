# Disclosure: LND doesn't wait for enough confirmations when closing channels

t-bast | 2026-08-13 15:06:43 UTC | #1

We're disclosing a vulnerability in lnd, which was fixed version 20.0 (shipped in February 2026). Before this version, lnd would forget channels that were collaboratively closed immediately after their first on-chain confirmation, without waiting for more blocks to protect against reorgs. If a 1-block reorg happened, an attacker could then publish any revoked state for that channel and lnd would not publish penalty-transactions, resulting in a loss of funds up to the entire channel amount.

This can be reproduced on regtest with the following steps:

*  open a large channel with an lnd node (e.g. 5 BTC)
* send payments through that node to move liquidity on the lnd side
* initiate a collaborative channel close
* generate 1 block, wait for the lnd node to detect the close
* invalidate that block with \`bitcoin-cli invalidateblock <blockhash>\`
* publish a revoked commitment transaction (e.g. the first commitment transaction where all the funds were on the attacker's side)
* mine blocks: penalty transactions are never published and lnd lost funds

Timeline:

* I discovered the issue while testing \`option_simple_close\` between \`eclair\` and \`lnd\` in february 2025 (this wasn't found using AI, which wasn't cool at that time: crazy, right?)
* I identified the bug in lnd, which was here: https://github.com/lightningnetwork/lnd/blob/09a4d7e224a64499017c392abe2ccd7f5cc48d03/peer/brontide.go#L3622
* I notified laolu, who acknowledged the issue and fixed it in https://github.com/lightningnetwork/lnd/pull/10331
* the fix was included in lnd v20.0

Nobody was impacted as far as we know. The fix is simple: lightning nodes should always wait for many confirmations before considering a transaction confirmed and moving on. The BOLTs recommend 6 confirmations, and implementations should let node operators use higher values, but should not let node operators use values lower than 6.

-------------------------

