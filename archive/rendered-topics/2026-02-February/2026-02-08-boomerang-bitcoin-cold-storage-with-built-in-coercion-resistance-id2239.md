# Boomerang: Bitcoin Cold Storage with Built-In Coercion Resistance

bitryonix | 2026-02-09 09:00:28 UTC | #1

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

bitryonix | 2026-02-09 05:36:52 UTC | #2

**Duress Mechanism in the Boomerang Protocol: Protecting Users Under Coercion**

Hey everyone, diving deeper into the Boomerang protocol's security features, here's a detailed breakdown of the duress mechanism. This is a system designed to let users signal distress discreetly during withdrawals, without alerting an attacker. It leverages memorized "consent sets" based on countries, encrypted communications, and integration with Search and Rescue (SAR) entities to trigger help if needed. The goal is to make coercion risky for attackers while maintaining plausible deniability.
I'll cover the core concepts, setup, runtime usage (e.g., during withdrawals), security assumptions, and how it handles adversaries. This is based on the protocol's design docs and diagrams. Please feel free to ask for clarifications!

**What is the Duress Mechanism?**
The duress system allows users to signal coercion (e.g., being forced to sign a transaction) by entering a non-standard input during "duress checks." These checks happen at key points like committing to a withdrawal or randomly during the process. If duress is signaled, the system silently sends a payload to SARs (trusted rescuers) to decrypt and use "doxing data" (personal info for location/recovery) to initiate a rescue, without changing observable behavior to tip off the attacker.
Key actors:

* **User**: Creates doxing data and a password; memorizes a "consent set" of 5 countries.

* **SAR (Search and Rescue Entity)**: Holds encrypted doxing data; Once received a payload created after a positive duress signal, can decrypt and act on duress signals.

* **Adversary**: Can capture/observe the user but can't break crypto or hardware security.


**Doxing Data: The Rescue Payload**
Doxing data helps SAR locate and rescue the user:

* **Static Data**: Fixed info like name, ID, addresses, phone numbers, and trusted contacts (encrypted with doxing_key = SHA256(doxing_password)). Sent to SAR during setup.

* **Dynamic Data**: Real-time feeds (e.g., phone location) encrypted and updated continuously.

On duress, Boomlet (hardware wallet) sends the doxing_key (encrypted for SAR) in message payloads. SAR decrypts, hashes to identify the user, and starts rescue ops.

**Core Concepts**

* **Consent Set**: 5 unique countries (no order) chosen by the user during setup. Matching this set means "no duress"; any other combination signals duress.

* **Country Mapping**: Based on 195 countries (UN-like list) sorted alphabetically (1 = Afghanistan, ..., 195 = Zimbabwe).

* **Duress Checks**: Boomlet generates 5 random lists of 1-195 numbers; ST (Secure Terminal) maps to countries and displays as columns. User selects one per column.

* **Plausible Deniability**: Under torture, user can reveal any 5 countries as the "consent set"—attacker can't verify without risking a real duress signal.

Probability of random guess succeeding: \~3.73e-12 (1 in 267 billion), making blind attacks infeasible.

**Setup Process**
Done via isolated devices (Iso, Boomlet, ST) with QR codes for secure, air-gapped comms:

* Boomlet and ST exchange identity pubkeys via Iso/QR to establish shared ECDH key (AES encryption).

* Boomlet generates random list \[1-195\], encrypts, sends via Iso/QR to ST.

* ST decrypts, maps to countries (alphabetical), displays list.

* User selects 5 countries via joystick (memorizes names); ST records indices, encrypts, sends back via QR/Iso.

* Boomlet decrypts, extracts numbers as consent_set.

* Confirmation: Boomlet sends 5 new random lists; user re-selects to match consent_set. Mismatch? Retry or restart.

After setup, user only remembers 5 country names; system stores numbers securely.

**Runtime Process (e.g., During Withdrawal)**
Checks triggered by Boomlet (at commitment or PRNG-based random intervals):

* Boomlet generates 5 random \[1-195\] lists, encrypts, sends via Niso/QR to ST.

* ST decrypts, maps to country columns, displays on small monitor.

* User selects one country per column via joystick (hidden from observer).

* ST encrypts indices, sends back via QR/Niso to Boomlet.

* Boomlet decrypts, extracts numbers, compares to consent_set:

  * Match = No duress: Encrypts all-zeroes payload for SAR.

  * Mismatch = Duress: Encrypts doxing_key for SAR.

* Payload embedded in next withdrawal message to Watchtower (WT).

* WT forwards payload to SAR.

* SAR decrypts:

  * All zeroes? Ignore (no duress).

  * Else? Hash to doxing_identifier, search database, decrypt doxing data, start rescue.

  * Malformed? Increment counter; if threshold hit, halt protocol.

* SAR signs/acknowledges back to WT/Boomlet for verification.

Once duress is signaled, all future messages carry it. Persistent alert.

**Security Features and Attacker Assumptions**
Covers forced transactions or continued signing under compromise. Resistant to:

* Replay attacks (fresh random lists each time).

* Typos (set-based, order-insensitive).

* Observation (ST's small screen + hidden input).

Attacker can:

* Capture/observe user, read plain messages, act as user, destroy hardware.

But cannot:

* Break JavaCard security, crypto, or observe ST input without seeing screen + hands simultaneously.


Consequences: No observable changes on duress (plausible deniability); withdrawal proceeds normally but SAR is alerted.

-------------------------

