# Asynchronous Pipeline Consensus: Decoupling Mining from Full-Block Validation

kcu201104 | 2026-09-22 15:27:54 UTC | #1

I’d like to share a framework designed to scale L1 throughput under Nakamoto security by shifting propagation latency from a physical bottleneck to a configurable validation window.

**Core Concept: Asynchronous Pipeline Consensus (k=6,i=2)**

* **Decoupled Mining Initiation:** Miners start mining block n+1 immediately upon verifying the Compact Block (CB) of block n, bypassing the serialization delay of downloading the Full Block (FB).

* **Deferred Declaration:** The validity of the transaction data in FBn is evaluated and finalized at block n+k via a 1-bit header field: `n_k_declaration` (True/False).

* **i–Confirmation Broadcast Delay (i=2):** To optimize bandwidth, FBn is broadcast only after CBn achieves i confirmations (at height n+i). This limits bandwidth waste during transient fork races.

* **Throughput Scaling:** Simulating a 10MB block size and 600s interval yields \~66.7 TPS—a 10x increase over Bitcoin—using a Poisson mining process. This is achieved with zero reduction in mining frequency; the difficulty target remains unchanged.

**Block Structure: Extended Header and Compressed UTXO IDs**

* **Extended Header (113 bytes):** Standard 80 bytes + 1 byte (`n_k_declaration`) + 32 bytes (`Comp_UTXO_Root`: Merkle root of the compressed UTXO ID list).
* **6-byte Compressed UTXO IDs:** Instead of full transactions, the CB carries a list of 6-byte truncated hashes of the UTXOs referenced by transaction inputs.
* **Mempool Independence:** CB validation constructs a Merkle tree from these compressed IDs to match `Comp_UTXO_Root`. It does not consult the local mempool for transaction reconstruction, eliminating the round-trip recovery stalls (`getblocktxn`) inherent to conventional BIP 152 at the CB layer.

**Logical Isolation and Invalidation Recovery**

* **No Chaining Rule:** UTXOs generated in block n cannot be consumed in blocks n+1 through n+k.
* **Atomic Frozen Set:** Upon receiving CBn, the node atomically registers the input UTXO IDs of block n and releases those of block n−k in a single state transition. Transactions violating this filter are excluded from candidate block generation.
* **Recovery Logic:** If FBn−k is declared False, the isolated assets are restored to spendable status in the mempool. Blocks from n−k+ to n−1 remain unaffected due to this decoupling, preventing cascading invalidation across subsequent blocks.
* **Alternative (Appendix B – Dual-Coinbase):** Removes mandatory freezing by introducing Coinbase1 (Draft Reward) and Coinbase2 (Finalized Reward, determined at n+k+1 based on the proportion of transactions consuming UTXOs from the recent k-block window, with a threshold X%). This removes the economic constraint on parameter k, enabling extreme scaling (e.g., k=30).

**Security and Attack Vector Deterrence**

* **Selfish Mining Bounded:** Expected revenue remains strictly governed by classical α*α* and γ parameters (γnew≤γBitcoin). Withholding an FB yields no private lead and triggers certain reward forfeiture if the k-window expires.
* **Difficulty Target Scaling (Punitive DAA):** Any block carrying a False declaration without a corresponding data-availability justification triggers an immediate computational penalty via a penalty coefficient η=1.2, scaling the difficulty target to a harder value: Tpenalty=Tnormal/η (lower target = higher difficulty).
* **Work-Weighted DAA Patch:** The normalized adjustment rule triggers at each 2016-block epoch: Tnormal+=Tnormal⋅(t/t0)⋅\[2016/(Ntrue+Nfalse⋅η)\], where Ntrue​ and Nfalse​ denote the number of True-declared and False-declared blocks in the epoch. This penalizes defectors with immediate hash attenuation (to 1/η≈83.3% of their nominal hash rate) and diminished long-term yields.
* **Upper Bound Calibration (η<1.22):** Constraining η=1.2 prevents a malicious pool controlling up to 45% of the hash rate from executing a Data Withholding Attack to artificially suppress the remaining 55% honest nodes' effective dominance.

**Fork Convergence and Validation Rules**

* **Data-Availability-Gated Chain Selection:** When a node discovers a longer chain B than its current chain A, it applies the longest-chain rule with a data-availability filter. If the fork is at the threshold (m=1,az=0), where az​ denotes the relative progress of Group A beyond the fork point, the node compares the `n_k_declaration` in the competing CB with its local data status. If the declaration matches local availability, the CB is accepted; otherwise, it is rejected. This prevents ghost chain attacks where a longer chain is claimed without the underlying data.
* **Deterministic Convergence (m>1):** Once a clear length difference exists, the longest-chain rule is prioritized. Blocks declared False are accepted without requiring Full Block data. Blocks declared True must be fully downloaded and verified before the switch is finalized. Regardless of its length, a chain is never accepted without the underlying data for True-declared blocks.
* **Coupling Mechanism:** If the n-th Full Block encounters an (n+k)-th Compact Block with `n_k_declaration = True` at an intermediate node, they are coupled into a single packet and propagated together. Nodes mining the (n+k)-th block with a False declaration due to data latency will immediately switch to True upon receiving this coupled packet, accelerating convergence without additional data request cycles.
* **Resilience to Block Withholding:** Even if an attacker withholds Full Block data to deceive the network with a longer chain, the Mandatory Data Verification rule (Section 3.4.2) ensures that nodes without the data will not accept the attacking chain. The malicious chain is rejected by honest nodes and becomes isolated due to propagation delays and resource exhaustion—imposing a physical cost on adversaries that honest participants do not bear.

**Key Discussion Points**

1. **Declaration Incentives:** The protocol relies on miners honestly declaring `n_k_declaration` based on local data availability. A miner could declare False even when the Full Block is available, hoping to orphan a competing block. The difficulty penalty η=1.2 is intended to deter this, but the upper bound is tight. Is there a more robust mechanism—perhaps a bond that is forfeited upon a verified false declaration—that could replace or supplement the penalty?

2. **Parameter Selection:** The current configuration (k=6, i=2, η=1.2, 6-byte IDs, 10MB blocks, 600s interval) is conservative. Appendix A discusses alternative configurations (e.g., 1MB blocks with 100s interval for faster confirmation). What is the optimal parameter set for high-frequency use cases (e.g., DeFi) versus settlement use cases (e.g., large transfers)?

**Additional discussion points:**

* **UTXO Freezing Trade-off:** Mandatory freezing with a smaller k, or voluntary freezing with a larger k (Appendix B)?

* **FB Propagation:** How should the protocol handle poor mempool synchronization—by increasing k, by improving FB relay, or by some combination?

---

```c
**Proposal: Scaling L1 Throughput via Asynchronous Pipeline Consensus**
(Full paper: "Minimizing Mempool Dependency in PoW Mining...", IACR ePrint 2026/141)

**Paper PDF:** [https://eprint.iacr.org/2026/141](https://eprint.iacr.org/2026/141)


```

GyuChol. Kim

-------------------------

