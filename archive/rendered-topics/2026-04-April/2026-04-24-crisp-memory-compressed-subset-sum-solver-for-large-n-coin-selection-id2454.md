#  CRISP — memory-compressed subset-sum solver for large-n coin selection

prashanshan6 | 2026-04-24 13:25:20 UTC | #1

Hi all — sharing an independent research project on subset-sum solving that I think has a narrow but real application in Bitcoin coin selection. Looking for feedback on whether the regime it targets is actually interesting to anyone here.

**## Background**

Bitcoin Core’s current coin selection pipeline (BnB → CoinGrinder → SRD → Knapsack) handles the change-free case well for typical wallets. Murch and sipa’s BnB paper (2017) and the subsequent CoinGrinder addition cover the n ≤ 200-ish regime that represents essentially all real Core users. I am not proposing to replace any of this.

However, there are scenarios where BnB’s iteration cap forces a fallback to knapsack (with change), even when an exact-match subset exists:

\- **\*\*Long-running JoinMarket makers\*\*** with 500-1000+ post-mix UTXOs per mixdepth

\- **\*\*Mining pool payout batching\*\*** where the UTXO pool is the entire pool wallet (Foundry USA’s payout txs routinely involve batches drawn from 100-175 UTXOs)

\- **\*\*Custodians doing cold→hot sweeps\*\*** where the source wallet may hold 1000+ UTXOs

\- **\*\*Silent Payments servers\*\*** where the sender’s wallet state for a single send can include many UTXOs tagged to one recipient

In these cases, BnB hits \`ErrorMaxWeightExceeded\` or its iteration cap and falls back to change-creating algorithms. The Core response — “consolidate your wallet” — is a reasonable product decision for end-user wallets but less appropriate for services that structurally accumulate large UTXO sets.

**## What CRISP does**

CRISP maintains a compressed representation of the subset-sum frontier and updates it one item at a time via a streaming merge on the encoded form. The compression exploits an empirical observation: for non-super-increasing item sets, the consecutive gaps between reachable subset sums concentrate into a small, low-entropy alphabet (in tested instances, \~70 unique gaps with top-3 accounting for \~73% of occurrences and gap entropy \~3.1 bits/symbol). This makes varint-based run-length encoding extremely effective.

Per-step frontiers are written to disk as they’re produced; the reconstruction walker (a backward DFS) reads them through an LRU byte cache to reconstruct actual subsets without ever holding all frontiers in memory.

The walker is biased toward **\*\*minimum-cardinality solutions\*\*** by trying TAKE-before-SKIP at every branching node. In practice this means CRISP recovers the smallest-k subsets that sum to T, which matters for coin selection: fewer inputs = smaller transaction = lower fee. In the benchmarks below, the planted target used k=500 items, but CRISP consistently recovers solutions at k≈282–299 — i.e., it finds smaller valid subsets than the one that was planted, because smaller ones exist in the item set and the walker finds them first.

**## Benchmarks (laptop, WSL2)**

n=1000, k=500 (planted), 10 solutions reconstructed per row. \`k found\` is the cardinality range of the actual subsets recovered — note that it’s consistently well below the planted k=500 because the walker’s TAKE-first bias recovers minimum-k solutions:

| R | build time | recon time | build peak RSS | recon cache | k found |

|—|—|—|—|—|—|

| 10^5 | 3.3s | 2.7s | 2.0 MB | 1 GB | 282..286 |

| 10^6 | 3.8s | 2.9s | 2.0 MB | 1 GB | 293..296 |

| 10^7 | 6.9s | 3.9s | 2.5 MB | 1 GB | 294..298 |

| 10^8 | 42.7s | 18.1s | 9.7 MB | 1 GB | 294..299 |

| 10^9 | 7.5 min | 15.6 min | 67.7 MB | 3 GB | 294..298 |

| 10^10 | 66.1 min | 67.9 min | 568 MB | 3 GB | 294..299 |

The k range is remarkably stable across five orders of magnitude of R, suggesting minimum-k is a structural property of the item set rather than a function of item magnitude.

For comparison, naive Bellman bitset DP would require \~282 GB of RAM at R=10^10 and \~28 GB at R=10^9 — infeasible on a laptop. CRISP handles both with build memory under 600 MB.

The algorithm is **\*\*not a compute speedup\*\*** over Bellman — it’s a memory compression. A machine with enough RAM to run naive Bellman could match build time. The contribution is making the large-R regime runnable at all on commodity hardware.

**## Honest comparison to BnB**

| Aspect | BnB | CRISP |

|—|—|—|

| Small n (< 50) | Milliseconds, optimal | Overkill, seconds |

| Medium n (50-200) | Fast when found, iteration-caps when not | Seconds, always finds if exists |

| Large n (500+) | Often iteration-caps → knapsack fallback | Handles up to n=1000+ on laptop |

| Memory | O(n) stack depth | 100 GB disk at n=1000, R=10^10 |

| Complexity | Worst-case 2^n, heavily pruned | Worst-case O(nT), empirically much better |

| Guaranteed to find exact match if exists | No (time-bounded) | Yes |

| Minimum-cardinality bias | Yes (BnB design) | Yes (TAKE-first walker) |

| Deterministic output | Yes | Yes, plus randomized diverse-solution mode |

For typical Core users, BnB is strictly better. CRISP’s niche is the regime where BnB’s iteration cap matters.

**## Where I’d actually use it**

I don’t think CRISP belongs in Bitcoin Core. The user population that hits BnB’s cap is small, and “consolidate your UTXOs” is a valid answer there.

I **\*\*do\*\*** think it could matter for:

\- **\*\*JoinMarket’s \`MERGE_ALGORITHMS\`\*\*** as an opt-in 5th algorithm for change-free sends on large-n wallets (writing up a separate issue there)

\- **\*\*BDK\*\*** as a backend option for wallets that want to offer exact-match selection

\- **\*\*Pool operator tooling\*\*** (non-consensus, non-wallet; think payout script on a Foundry-shaped operation)

**## Limitations I know about**

\- Build time at R=10^10 is 66 min — not interactive. Good for scheduled batch jobs, not for a taker waiting on a CoinJoin coinflip round.

\- Disk usage at R=10^10 is \~100 GB. Checkpoint-style saves (every k steps) would cut this linearly but aren’t implemented.

\- Reconstruction cache budget dominates wall time at large R. 3 GB cache at R=10^10 shows 0% hit rate (every access a disk miss). A workstation with 32 GB allocated to cache would change this.

\- **\*\*Not peer-reviewed.\*\*** The diff-alphabet compression claim is empirical across 17 adversarial distributions, but the asymptotic bound under this structure is an open question.

\- Duplicate-heavy item sets can cause the walker to explore many index-distinct-but-value-equal subsets; a circuit breaker handles pathological cases.

**## Open questions for this forum**

1\. Is there anyone here hitting BnB’s iteration cap in practice on large-n wallets? If so, what’s your current workaround?

2\. Has anyone studied the gap-alphabet structure on subset-sum frontiers formally? I’ve only seen empirical work; would appreciate pointers to prior theory.

3\. For a potential BDK backend, would exposing “find exact subset” as an optional API be useful beyond coin selection (e.g., for wallet-to-wallet rebalancing tooling)?

4\. Side-channel concerns: the reconstruction walker’s branching is deterministic TAKE-first by default. For privacy wallets, should I be worried about timing analysis leaking information about the chosen subset, and if so what’s the right mitigation?

**## Links**

\- Repo: https://github.com/prashanshan6/crisp

\- README has the full benchmark table, flag reference, and algorithm description

\- MIT licensed; \~3,000 LOC C; tested against 17 adversarial distributions (arithmetic progressions, geometric sequences, prime-only, Fibonacci-capped, super-increasing resets, all-equal, bimodal clusters, near-max clustered, etc.)

Happy to answer technical questions or run additional benchmarks if anyone has specific scenarios they’d want to see.

-------------------------

