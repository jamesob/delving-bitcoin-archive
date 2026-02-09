# Boomerang: Bitcoin Cold Storage with Built-In Duress Protection

bitryonix | 2026-02-08 22:27:40 UTC | #1

Hello everyone,

I’d like to share a new Bitcoin custody protocol designed to significantly improve physical security and coercion resistance for high-value holders.

Boomerang is a Bitcoin cold storage protocol with integrated duress protection. It lets you set up custody such that withdrawals become intentionally non-deterministic and include duress signaling, without requiring any changes to Bitcoin consensus. This creates uncertainty for attackers and gives holders a better chance to survive coercion attempts.  

**Key features at a glance**

* Protocol-level duress protection built into the custody process.  

* Non-deterministic withdrawal ceremony to reduce predictability.

* Fully compatible with Bitcoin consensus.  

* Proof-of-concept implementation in Rust.


**Protocol Description**
Boomerang is a Bitcoin cold-storage protocol designed for high-value holdings, providing strong protection against duress (e.g., coercion or threats) without altering Bitcoin's consensus rules. It achieves this through a non-deterministic withdrawal process enforced by secure hardware, creating unpredictability in signing with embedded, plausibly deniable duress signaling and optional "search-and-rescue" (SAR) escalation. Funds are locked in a Taproot output with two spending regimes: a probabilistic "Boomerang" path using MuSig2 keys (including a non-backupable key in a Java Card applet) requiring 5-of-5 multisig, and a deterministic "normal" path with timelocks for fallback.

**Setup**
The setup involves coordinated, secure steps among entities like the user, isolated environments (Iso), secure terminals (ST), watchtowers (WT), SAR providers, and hardware applets (Boomlet/Boomletwo):

* **SAR Registration:** User registers with SAR entities via an encrypted phone app, providing doxing data for potential rescue.

* **Key Generation:** In an air-gapped Iso, generate a normal mnemonic key and Boomlet applet creates MuSig2 shares plus a non-backupable boomlet key. Set duress consent via selecting 5 countries (exact match signals no duress).

* **Parameter Agreement:** Peers agree on timelocks (milestone blocks), and WTs via encrypted, signed TOR exchanges.

* **Mystery Generation:** Each Boomlet randomly selects a secret "mystery" threshold (steps in withdrawal) within a user-defined range.

* **Backup and Sync:** Create/verify a single backup applet (Boomletwo); synchronize keys, parameters, and fingerprints via WT for consistency.

This ensures no single party can predict or bypass the process, emphasizing tamper-evident hardware and opsec.

**Withdrawal**
Withdrawal is a collaborative, unpredictable multi-phase ceremony post-milestone_block_0, involving all 5 peers signing a PSBT:

* **Initiation:** One peer creates and shares the PSBT; all approve via ST and WT.

* **Duress Check:** Each peer selects countries; mismatches signal duress, triggering encrypted SAR payloads without altering behavior.

* **Digging Game:** A ping-pong loop of signed messages via WT, incrementing a counter until each Boomlet hits its secret mystery threshold (could take months due to randomness). Pseudo-random duress checks occur throughout.

* **Signing:** Generate/aggregate MuSig2 partial signatures offline in Iso; broadcast via WT.

* **Fallback:** If stuck (e.g., lost hardware), funds unlock deterministically after timelocks via normal keys.

The design prioritizes coercion resistance through unpredictability and deniability.

**Repositories**

* Design and specification of the protocol: [https://github.com/bitryonix/boomerang_design](https://github.com/bitryonix/boomerang_design)

* Proof-of-concept Rust implementation following the design: [https://github.com/bitryonix/boomerang](https://github.com/bitryonix/boomerang)


I’m looking forward to your critical reviews, feedbacks and collaborations, especially around security analysis, usability improvements, and real-world deployment guidance.

Thanks,
bitryonix

-------------------------

