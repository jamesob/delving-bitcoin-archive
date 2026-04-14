# # CAAS: A Continuous Adversarial Audit Standard for secp256k1 Libraries

shrec | 2026-04-14 01:49:22 UTC | #1

**## Background**

I’ve been building \[UltrafastSecp256k1\]( https://github.com/shrec/UltrafastSecp256k1 ) — an independent secp256k1 implementation with GPU acceleration (CUDA/OpenCL/Metal), embedded platform support (ESP32, STM32), and full Bitcoin protocol coverage (ECDSA, Schnorr BIP-340, BIP-352 Silent Payments, BIP-324, MuSig2, FROST).

Craig Raw integrated the library into \[Frigate\]( https://github.com/sparrowwallet/frigate ) for BIP-352 Silent Payments scanning.

This post describes the audit methodology — **\*\*CAAS (Continuous Adversarial Audit System)\*\*** — and why I believe it’s structurally more sound than traditional snapshot audits for a library at this stage.

**—**

**## The Snapshot Problem**

Traditional audit model:

\> code at commit X → 2-week engagement → PDF → trust

The Bitcoin community’s core principle is “don’t trust, verify.” The irony is that cryptographic library audits almost universally violate this principle: the PDF is a trust artifact, not a verification artifact. You cannot run a PDF. You cannot reproduce it. It does not update when the code does.

Heartbleed is the canonical example. OpenSSL had been reviewed extensively. The bounds check was missing for two years. No continuous system checked the specific property that failed on every commit.

**—**

**## What CAAS Provides**

Seven principles, all with executable evidence:

**\*\*1. Every claim has a running test.\*\*** \`docs/AUDIT_TRACEABILITY.md\` maps every security claim to a specific test, CI workflow, and evidence artifact. Claims without tests are removed.

**\*\*2. Every discovered weakness becomes a permanent test.\*\*** The bug capsule system:

\`\`\`json

{

“id”: “BUG-2026-0001”,

“category”: “CT”,

“title”: “CT branch leak in ecdsa_sign_recoverable”,

“fix_commit”: “0a93ff4b”,

“expected”: { “result”: “no_timing_leak”, “timing_threshold”: 10.0 },

“exploit_poc”: true

}

\`\`\`

\`bug_capsule_gen.py\` converts this into a regression test + exploit PoC + CTest integration, automatically. The attack class cannot silently return.

**\*\*3. Exploit writing is part of audit.\*\*** 171 exploit PoC modules in \`audit/test_exploit\_\*.cpp\` cover known CVE and ePrint attack classes — including recent ones directly relevant to the Bitcoin ecosystem:

| Attack | Reference | Coverage |

|--------|-----------|----------|

| Nonce bias / lattice | ePrint 2023/841, 2024/296 | \`test_exploit_biased_nonce_chain_scan.cpp\` |

| ECDSA DFA fault | ePrint 2017/975 | \`test_exploit_batch_sign.cpp\` |

| Timing / EUCLEAK | ePrint 2024/1380 | CT pipeline |

| FROST weak binding | ePrint 2026/075 | \`test_exploit_batch_schnorr_forge.cpp\` |

| Batch verify bypass | ePrint 2026/663 | \`test_exploit_batch_verify_poison.cpp\` |

| Schnorr Fiat-Shamir | ePrint 2025/1846 | \`test_exploit_batch_schnorr.cpp\` |

| ROS / concurrent Schnorr | ePrint 2020/945 | \`test_exploit_batch_soundness.cpp\` |

**\*\*4. CT verification is three independent pipelines:\*\***

\`\`\`

ct-verif.yml    — LLVM IR symbolic analysis (compile-time)

valgrind-ct.yml — Valgrind Memcheck taint propagation (runtime)

ct-prover.yml   — dudect Welch t-test statistical timing (empirical)

\`\`\`

All three run on x86-64. \`ct-arm64.yml\` runs the full pipeline on native ARM64. CT claims are not “the code looks side-channel-free” — they are three independent systems saying OK on every commit.

**\*\*5. Formal verification for the hardest parts.\*\*** SafeGCD (Bernstein-Yang divstep for \`ct::scalar_inverse\`) is verified with:

\- Z3 SMT prover: 7 theorems, 17 proofs

\- Lean 4: 19 theorems including CT≡branching equivalence exhaustive over 2²⁴ 8-bit inputs

This is the same approach used for \`bitcoin-core/secp256k1\`'s constant-time scalar inverse.

**\*\*6. Research ingestion is systematic.\*\*** \`scripts/research_monitor.py\` fetches ePrint and NVD feeds daily. Every relevant signal is matched against the source graph, a PoC is constructed, and the result is stored in \`ai_memory.py\` (persistent cross-session SQLite). The gap between a published attack and evaluation against this codebase is bounded by one working day.

**\*\*7. Full reproducibility.\*\*** Everything runs in Docker:

\`\`\`bash

\# External auditor — zero local setup required

docker build -f Dockerfile.auditor -t ufsecp-auditor .

docker run --rm ufsecp-auditor

\# Bit-for-bit reproducible build verification

docker build -f Dockerfile.reproducible -t uf-repro-check .

docker run --rm uf-repro-check

\`\`\`

The auditor image contains libsecp256k1 (reference), @noble/secp256k1, coincurve, and python-ecdsa — all hash-pinned — for differential tes@nobleing.

**—**

**## BIP-352 Performance (Relevant to Frigate Integration)**

The GPU BIP-352 pipeline runs entirely on-GPU with zero CPU round-trips across 7 stages (k×P → serialize → tagged-SHA256 → k×G → point-add → serialize+prefix → prefix-match):

| Mode | ns/op | Throughput |

|------|-------|-----------|

| CUDA GLV w=4 | 179.2 ns | 5.58 M/s |

| CUDA LUT (64MB table) | 91.0 ns | **\*\*11.00 M/s\*\*** |

| CPU x86-64 reference | 24,285 ns | 41 K/s |

Frigate real-world result (133M tweaks, 2-year scan to block 914,000):

| Hardware | Time |

|----------|------|

| 2× RTX 5090 | 3.2 seconds |

| RTX 5080 | 7.7 seconds |

| CPU Intel Core Ultra 9 285K (24 cores) | 3 minutes 50 seconds |

Independent CPU comparison by @craigraw (\[bench_bip352\]( https://github.com/craigraw/bench_bip352 )): UltrafastSecp256k1 is 1.20× faster than libsecp256k1 on full BIP-352 pipeline, with individual operations up to 11.4× faster (tagged SHA-256 path).

**—**

**## What This System Does Not Claim**

Transparency is as important as coverage:

\- **\*\*No third-party cryptographic audit yet.\*\*** Openly documented in \`AUDIT_PHILOSOPHY.md\`. CAAS is designed to make one maximally efficient — but it hasn’t happened.

\- **\*\*GPU CT is code-discipline only.\*\*** Vendor JIT compilers may transform Metal/OpenCL/CUDA kernels at runtime. The 3-pipeline formal CT verification applies to CPU paths only. Production signing always routes through the CPU CT layer.

\- **\*\*FROST/MuSig2 are Tier 2 (Experimental).\*\*** Protocol-level formal proofs are future work. PoC tests cover known attack vectors; API may evolve.

\- **\*\*Novel attacks.\*\*** No prior PoC covers unknown unknowns by definition.

**—**

**## For Reviewers**

Three commands produce a full external audit package:

\`\`\`bash

\# 1. Build auditor environment

docker build -f Dockerfile.auditor -t ufsecp-auditor .

\# 2. Run full suite

docker run --rm ufsecp-auditor

\# 3. Run specific exploit category

docker run --rm ufsecp-auditor ctest --test-dir /build -L exploit -V

\`\`\`

Machine-readable evidence:

\`\`\`bash

python3 scripts/export_assurance.py -o assurance_report.json

python3 scripts/external_audit_prep.sh  # full package in one command

python3 tools/source_graph_kit/source_graph.py hotspots 20  # coverage gaps

\`\`\`

The source graph indexes 9,071 functions with 8,638 symbol-level audit scores, 4,766 test→function mappings, and 7 documented open audit gaps — visible to any reviewer before they start.

**—**

**## Discussion**

I’m particularly interested in feedback on:

1\. The CT layer coverage — are there secp256k1-specific timing attack vectors that the current three-pipeline approach doesn’t cover?

2\. The FROST/MuSig2 protocol security — what’s the right approach for formal verification at this tier, given that Lean 4 proofs for SafeGCD are already in CI?

3\. The comparison to \`bitcoin-core/secp256k1\`'s audit approach — what does their model get right that CAAS should incorporate?

Repository: \[ github.com/shrec/UltrafastSecp256k1 \]( https://github.com/shrec/UltrafastSecp256k1 )

CAAS specification: \[docs/AUDIT_STANDARD.md\]( https://github.com/shrec/UltrafastSecp256k1/blob/efe4aeda07a71c0bd939c156a6d5325f4f495142/docs/AUDIT_STANDARD.md )

-------------------------

