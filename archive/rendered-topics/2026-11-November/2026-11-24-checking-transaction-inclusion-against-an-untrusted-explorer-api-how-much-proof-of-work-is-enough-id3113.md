# Checking transaction inclusion against an untrusted explorer API: how much proof of work is enough?

devdavidejesus | 2026-11-24 02:04:40 UTC | #1

Disclosure: I maintain btc-toolkit, a small stdlib-only Python CLI. This question comes from designing its `verify` command. I'm looking for design feedback, not users.

**Setup.** The tool has no node and no header chain. It queries an Esplora/Mempool API for four things: transaction status, the raw transaction, the Merkle proof, and the block header. It trusts none of them. It checks that:

1. The raw transaction hashes to the claimed txid (a transaction whose non-witness serialization is exactly 64 bytes is rejected).
2. The Merkle branch folds to a root (any node pair that could also be parsed as a 64-byte transaction is rejected, addressing CVE-2017-12842).
3. The 80-byte header commits to that root and hashes to the claimed block hash.
4. The header hash is below the target encoded by its `bits`, and that target is within the network's PoW limit.

What remains is the cost of forging a single header that meets its own target. The verdict ignores everything the API claims about height or timestamp, since a forger can choose those. Today, a mainnet header with difficulty ≥ 50 T is reported as "verified." That represents roughly 2^77.5 hashes, or about 38% of the expected work for a block at recent difficulty (~133 T in October 2026). Anything below that is reported as "consistent, weak work," rather than "verified."

The docs spell out what this does not prove: the block's height or that it belongs to the most-work chain.

**Questions**

1. **Fixed threshold.** A constant ages in both directions. As difficulty rises, it represents a smaller fraction of a real block's work; meanwhile, it labels genuine older blocks as weak. Is there a better anchor that still relies on nothing the API claims? One option would be a hardcoded (height, minimum difficulty) table updated with each release, at the cost of trusting the release.
2. **Descendant headers.** Fetching the next k headers and checking that each commits to the previous header's hash multiplies the forger's cost by roughly k, at little bandwidth cost. Is cumulative work over a short linked suffix the right metric? Is there a principled choice of k for verifying a payment claimed to be confirmed? The API could still serve a privately mined chain, so this raises the cost of forgery but proves nothing about consensus.
3. **Anything missing?** Are there other cases this verifier should reject, beyond the 64-byte ambiguity? The "Pragmatic definition of consensus for light clients" thread argues that SPV clients should halt on anomalies such as difficulty drops and time lags. With only a single header, there's little to compare against. Would checking the timestamp against the verifier's own clock be worthwhile, or would it be more misleading than useful without the previous 11 headers?

Code for reference: `btc_toolkit/verify.py`

https://github.com/devdavidejesus/btc-toolkit

Thanks for any pointers, including "this is already solved, see X".

-------------------------

