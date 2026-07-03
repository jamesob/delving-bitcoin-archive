# Empirical ML-DSA-87 data from a live SHA-256 PoW chain (relevant to BIP-360 / Witness V3 sizing discussion)

geobees | 2026-07-03 05:29:02 UTC | #1

We've been running ML-DSA-87 (NIST FIPS 204, Security Level 5) as the sole signature scheme on KVANTA5 (KV5), a SHA-256 proof-of-work chain, live on mainnet since June 1, 2026. To get concrete data on the input-count and witness-size questions raised in this thread, we ran a deliberate stress test on mainnet: fan a single input out to 4,000 P2QR outputs, then consolidate all 4,000 back into a single output paid to: kvqr1kj0dn9plgxxtewg44vm8ze3f899he22p27qev2xfmh6yt0498qgqx8gexh. Posting the real numbers below.

**## Background**

* Consensus: SHA-256 PoW, ASIC-compatible
* Signature scheme: ML-DSA-87 (CRYSTALS-Dilithium, Level 5) from Genesis --- no hybrid, no fallback scheme
* Two output types: native P2QR (\`kvqr1...\`) and a wrapped P2SH form (\`3...\`) for legacy/pool compatibility.
* 231M hard cap, mainnet live since June 1, 2026
* First non-coinbase P2QR spend confirmed on-chain at block #283

**## Stress test: two stages, real mainnet blocks**

**\*\*Stage 1 (Block** #21797**) --- fanout, 1 input -> 4,001 outputs\*\***
TXID \`804b8e7bcea43dbaebe62078f5f28e2df26ceb77a828102c231d598f5634e255\`. Size 179,323 bytes, vsize 179,323, weight 717,292. One P2SH Compatibility Address input spending out to 4,000 new P2QR outputs of 0.01 KV5 each plus one larger change-style output, confirmed under live PoW. Mined 6/18/2026, 1:29:13 AM.

**\*\*Stage 2 (Block** #21798**) --- consolidation, 4,000 inputs -> 1 output\*\***
TXID \`9d24c8669da9bcccf05b3eee6cf80a210873ed4a8b5d408da29c64a13c02dbca\`. This is the more relevant data point for the input-cost question this thread flags as open. Size **\*\*29,072,055 bytes (\~29MB)\*\***, vsize 29,072,055, weight **\*\*116,288,220\*\***, mined one block later at 1:30:02 AM. All 4,000 P2QR outputs from Stage 1 were spent back into a single 39 KV5 output, confirmed under real proof-of-work consensus --- a deliberate stress test, not organic traffic, but run on live mainnet rather than a testnet or simulator.

Exact per-input size from these totals: 29,072,055 bytes / 4,000 inputs ≈ **\*\*7,268 bytes/input\*\*** average, consistent with ML-DSA-87's signature (4627 bytes) plus public key (2592 bytes) sizes plus scriptSig/varint overhead per input.

As an independent check, a separate, much smaller transaction on the same chain --- 2 P2QR inputs, 2 outputs, total size 14,632 bytes --- yields a near-identical figure of \~7,253 bytes/input once output and transaction overhead are subtracted. Two transactions at very different scales (2 inputs vs. 4,000) landing within \~15 bytes/input of each other is a good sanity check that this number is stable rather than an artifact of one stress test.

Decoding that smaller transaction's scriptSig at the opcode level confirms this precisely: each input's signature is pushed via \`OP_PUSHDATA2\` with an explicit length field of \`0x1312\` (little-endian) = exactly **\*\*4627 bytes\*\*** --- matching the ML-DSA-87 signature size bit-for-bit, not just approximately.

**\*\*Mainnet proof at scale:\*\*** Block #24521 (mined June 19, 2026) contained a single transaction with **\*\*715,001 P2QR outputs\*\*** --- one \~28 KV5 change output plus 715,000 0.0001 KV5 outputs --- confirmed in a 30.77 MB block under live proof-of-work. Queryable on the KV5 explorer. This validates that the chain handles large output counts not as a stress test, but as routine mainnet operation.

The **\*\*weight\*\*** figure may be the more striking number for this discussion: 116,288,220 weight units on a single transaction is roughly 29x the weight of a maximally full Bitcoin block (4,000,000 WU) --- a concrete illustration of what "the input side is the expensive side" actually costs at this scale.

**## Implementation note**

The ML-DSA-87 cryptography is based on the pq-crystals reference implementation ( https://www.pq-crystals.org/index.shtml ) --- the original CRYSTALS-Dilithium team's code. The P2QR transaction output scheme itself, and the Mining Compatibility Wrapper using P2SH addresses for pool compatibility, were designed specifically for this chain. We've coded the integration, not the underlying cryptography.

**## Why this might be useful to this thread**

The BIP-360 / Witness V3 discussion has been working from theoretical limits on PQ input/output counts per block. We ran this as a deliberate stress test specifically to put real numbers behind those limits on a live SHA-256 PoW chain --- not a simulator, not a testnet. All claims below are queryable on the KV5 explorer's live verification ledger.

If it's useful, we're glad to make raw block/transaction data available for independent verification, run further stress tests at different scales (e.g. inputs-per-transaction near the theoretical \~72/720 figures from the BIP draft), or share the script we used.

**\*\*Reference txids:\*\***

* Fanout tx (block #21797, 1->4,001): \`804b8e7bcea43dbaebe62078f5f28e2df26ceb77a828102c231d598f5634e255\`
* Consolidation tx (block #21798, 4,000->1): \`9d24c8669da9bcccf05b3eee6cf80a210873ed4a8b5d408da29c64a13c02dbca\`
* 715K output proof (block #24521, single TX, 715,001 outputs): query-able on kvanta5.live explorer: \`abb67b2c05c965080cc19b6d4dcf23d501888e1d87bf4cf4122a50bcee2e6cf8\`
* First P2QR spend (block #283): see chain explorer

Happy to answer questions on the implementation in the meantime.

GitHub: https://github.com/Kvanta-Organization/Kvanta5-Core
Explorer: https://kvanta5.live/

-------------------------

