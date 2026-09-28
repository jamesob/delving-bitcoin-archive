# Bitcoin-rs: an independent Bitcoin node written in Rust

gosunuts | 2026-09-28 09:16:21 UTC | #1

I’d like to share early work on [bitcoin-rs](https://github.com/gosuda/bitcoin-rs), an independent Bitcoin full node written in Rust.

Like [btcd](https://github.com/btcsuite/btcd), it's an independent implementation, built to explore architectural and performance changes that are difficult to pursue within Bitcoin Core while remaining externally verifiable.

Some of the main differences are:

1. Integrated UTXO and script-index model — the node owns the canonical UTXO set used for validation. It also provides a read-only, Esplora-compatible script index for common wallet and explorer queries, reducing reliance on separate Electrum indexers.
2. Modular architecture with explicit failure boundaries — consensus, chainstate, storage, networking, mempool, indexing, and orchestration are kept deliberately small and separated, so failures and implementation complexity stay contained instead of propagating across the node.
3. Whole-node performance — synchronization, storage, memory usage, concurrency, I/O, and indexing are optimized together, with the goal of achieving competitive whole-node performance against Bitcoin Core.
4. Native and kernel-backed script validation — script validation can run either through bitcoin-rs’s native Rust implementation or through libbitcoinkernel. The kernel path can also be used as an oracle to cross-check the native implementation.

We are continuing to expand verification and benchmarking against Bitcoin Core.

Feedback and independent verification of the architecture and, especially, the verification approach would be very welcome.

Repository:

https://github.com/gosuda/bitcoin-rs

Live mainnet explorer:

https://bitcoin-rs-explorer.gosunuts.xyz

-------------------------

