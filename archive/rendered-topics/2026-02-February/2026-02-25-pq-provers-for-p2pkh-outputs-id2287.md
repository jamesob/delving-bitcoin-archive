# PQ provers for P2PKH outputs

olkurbatov | 2026-02-25 11:20:42 UTC | #1

## Intro
As part of the past research project, we've been benchmarking PQ proof systems for proving ownership of a Bitcoin P2PKH address, without revealing the public key. Despite the fact that the benchmarks were made ~6 months ago (so some projects could make additional optimization and improvements for their frameworks, affecting the proof generation complexity), we wanted to share our results here since they shed light on the practical feasibility (and current limitations) of using current ZK frameworks for a post-quantum transition (I was told that this topic was raised on the mailing list some time ago).

## Motivation
Quantum computers threaten systems where knowledge of the public key suffices to recover the private key. For P2PKH outputs, the public key is hidden behind `ripemd-160(sha-256(pk))` until spending time, but once revealed in a transaction, it becomes vulnerable.

One proposed approach to a PQ transition is to allow users to prove they own a Bitcoin address and cryptographically link it to a post-quantum address, all without exposing their public key on-chain. This requires a zero-knowledge proof that:
* The prover knows a public key `pk` such that `ripemd-160(sha256(pk))` matches a declared hash (from which the P2PKH address is trivially derivable).
* The prover knows a valid ECDSA signature `sigma` over a pre-determined message hash `hm` under `pk`.

The proof system itself must be post-quantum secure.

## What we're proving
The circuit (program) is the same across all systems. The witness is `(pk, sigma)` and the statement is `(pk_hash, hm)`. Inside the proof:
1. ECDSA signature verification over secp256k1
2. SHA-256 hash of the public key (we could implement SHA-256 + RIPEMD-160 in circuits but we simplify the proving and leave `ripemd-160(.)` calculation public)

The public signal is `(sha256(pk), hm)`: the verifier never sees `pk` itself. From `sha256(pk)` the P2PKH address is recoverable publicly via hash160 + Base58Check. All implementations use the same static test vectors (64-byte uncompressed public key, 64-byte signature, 32-byte message hash).

> Such a circuit design allows us to separate the signing (wallet) device from the proving device:
> 1. We don't need to extract the secret key from the signing device, but only to ask signing the constant pre-defined message, so this approach is supposed to be secure
> 2. We don't need to implement the proving on the signing device (sometimes it's not even possible because of their low-power design)

Source code: https://github.com/BlockstreamResearch/pq-p2pkh

## Benchmark results

All native benchmarks were run on Apple M2 Max.

| Proving System | Proving Time | Verification Time | Memory usage| Proof Size | Notes |
|---|---|---|---|---|---|
| STWO-Cairo | 8 s | 50 ms | 6 GB | 5.6 MB | Cairo program, STWO prover |
| Ligero (Apple M2 Max) | 4 s | 2 s | 1.8 GB | 4.2 MB | Runs on Mac|
| Ligero (WebGPU) | 22 s | 4 s | 1.9 GB | 4.2 MB | Runs in Chromium browser |
| RISC Zero | 2 m | 100 ms | 1.2 GB | 2 MB | RISC-V zkVM |
| SP1 | 1 m | 2 s | 16 GB | 10 MB | Succinct's zkVM |
| Valida (Lita) | 4.5 m | 2 s | 12 GB | 6 MB | Plonky3 backend |

A few notes and observations:

* STWO-Cairo is the fastest native prover at ~8 seconds, but its current WASM build requires ~6 GB RAM, which makes it impractical for some environments today.

* Ligero is the only system that actually runs in a browser (via WebGPU), with a 22-second proving time. This is nice if client-sideness is important.

* RISC Zero was additionally successfully tested on mobile (iPhone 14 Pro): proof generation took ~6 minutes with ~1.2 GB peak RAM, which is surprisingly feasible for a phone.

* Verification times are generally fast (50 ms for the fastest), but the question of where verification happens matters enormously. The full on-chain verification (without wrappers) is not currently possible.

## Summary

These benchmarks suggest that **client-side STARK proving of P2PKH ownership is approaching feasibility**, but:

- Proof sizes. Our early (pre-optimization) benchmarks show the proof size ranged from 5.6 MB (STWO-Cairo) to 10 MB (SP1) for P2PKH-related proofs. These are fine for off-chain registries but far too large for anything on-chain.

- These proofs are useful for external registries (linking Bitcoin addresses to PQ addresses) or for future covenant designs, but not for direct on-chain PQ migration without protocol changes.

- Even 8 seconds on an M2 Max is a lot of computation, especially for low-powered signing devices (HWW).

## Appendix. PQ signature verification benchmarks
We got carried away and also benchmarked STARK proofs for post-quantum signature verification, i.e., proving, within a STARK, that an SLH-DSA signature is valid. This might be relevant to privacy-preserving PQ applications (e.g., anonymous credentials, accumulators, etc.).

| System | Algorithm | Proof Gen. | Proof Verif. | Proof Size |
|---|---|---|---|---|
| SP1 | Falcon-512 | 40 s | <1 s | 12.5 MB |
| RISC Zero | Falcon-512 | 2 min | <1 s | 1.4 MB |
| SP1 | ML-DSA-87 (Dilithium) | 4.5 min | 3 s | 96.3 MB |
| RISC Zero | ML-DSA-87 (Dilithium) | 21 min | <1 s | 9.4 MB |
| SP1 | SPHINCS+-SHAKE256-256f | 97 min | 75 s | ~2.4 GB |
| RISC Zero | SPHINCS+-SHAKE256-256f | 9 hours | 5 s | 226 MB |

-------------------------

