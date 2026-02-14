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

bitryonix | 2026-02-14 12:54:11 UTC | #3

Here I have tried to present a high-level overview of the protocol. Your comments are highly appreciated. Thank you all.

### What is Boomerang?

Imagine you have a lot of bitcoins - maybe for a company or personal savings - and you’re worried not just about hackers, but about real-world threats like someone kidnapping you or threatening you to hand over the money. Regular “cold storage” (like keeping your keys offline on a hardware wallet) is great against online thieves, but it doesn’t help if an attacker forces you to sign a transaction right away. That’s where Boomerang comes in: it’s a system designed to make stealing bitcoins through force way harder and riskier for bad guys.

### Why Does Boomerang Exist?

* **The Problem**: Big bitcoin holders (like businesses with millions in bitcoin) often have a few key people who control the ultimate access. Hackers can’t easily get in, but a **“wrench attack”** - basically, physically forcing those people to transfer the money - works because normal systems let you send funds quickly and reliably.
* **The Goal**: Boomerang flips the script. It turns the withdrawal process into something unpredictable in timing during a certain period, with built-in ways to securely signal duress. Attackers can’t count on getting the money in a set timeframe (or without risks), so they’re less likely to try. Importantly, you can always withdraw eventually if everything’s legit. It’s just not instant or predictable at first, and becomes straightforward after a set time.

It’s like hiding your treasure in a vault that takes a random, unknown, but within a wide range (selected by you privately), amount of time to open, and you can secretly hit a panic button without the attacker detecting if you have done such thing during the process. After a milestone (like 2 years), the vault can be opened normally and deterministically.

### How Does It Work?

Boomerang isn’t for everyday spending. It’s for long-term storage where you rarely need to touch the funds. Here’s the basics:

1. **Setup Basics**:

   * You split control among a small group (e.g., 5 trusted people or “peers”).
   * Each person uses special hardware devices (like secure cards or apps) to hold parts of the keys.
   * You also connect to neutral “watchtowers” (services that help coordinate) and “search and rescue services” (that can alert help if things go wrong).
   * You provide encrypted personal info (like your location) to the rescue service, locked with a secret password only you know.
   * Each peer privately sets their own range for how many “steps” (rounds of checks) their part of the process might take; say, a minimum and maximum number, like 6 months to 1 year worth of steps (tied to Bitcoin’s blockchain timing). The group doesn’t share these ranges; each person’s device (Boomlet) randomly picks a number within their own private range when needed.
   * The group agrees on a future “milestone block 1” (a specific point on the Bitcoin blockchain, e.g., in 2 years) after which withdrawals become fast and predictable; no more randomness.
   * Prior to the milestone, the withdrawal is non-deterministic and the duress protection is in place. After that milestone, it is just a plain normal spending condition in a taproot address.

2. **Withdrawing Money**:

   * To move bitcoins, everyone in the group has to approve through a back-and-forth process.
   * In the boomerang period (before the milestone mentioned earlier, like the first 2 years), it involves multiple rounds of checks (the “digging game”). The exact number of steps for each peer’s part is randomly picked by their Boomlet within their own private min-max range. No one (not even the group) knows the others’ ranges or picks upfront, creating overall uncertainty in total time - so neither do attackers. This makes it hard for bad guys to plan around the timing, giving potential victims time (e.g., months) to be rescued if coerced.
   * During approvals, you can secretly signal “I’m under duress” (e.g., by choosing certain options in a quiz-like interface on a device). This looks normal to attackers but triggers the rescue service.
   * If anything seems off (like missed duress checks or messed up responses), the process just… stops. No money moves. But if all’s good, it completes - you just don’t know exactly how long it’ll take ahead of time.
   * After the milestone block (e.g., 2 years in), you enter a “normal era” where everything is quick and deterministic, like regular cold storage.

3. **Duress Protection**:

   * The “secret signal” is easy: You memorize a set of 5 countries during setup as the consent set. Later, the system shows lists of countries, and your choices either confirm “all good” by entering the consent set or scream “help!” by entering any other combination of countries without obvious signs or stopping the ceremony. You signal the duress but the attacker observe no change compared to the non-duress situation.
   * If duress is signaled, the rescue service gets your info and can start a “search and rescue” - like contacting authorities or trusted contacts.

The whole thing uses encryption and anonymous networks (like Tor) to keep communications hidden. It’s not foolproof against everything, but it makes coercion a bad bet for attackers: too uncertain in duration and too risky.

### Who Is It For?

* **Big Holders**: Companies or wealthy individuals with bitcoin they don’t need to access often (e.g., long-term reserves).
* **High-Risk Situations**: Places where physical threats are real, like in unstable regions.
* **Not For**: Casual users or quick trades; it’s too slow and complicated for that.

In short, Boomerang is like a bitcoin safe with a randomized time-delay lock and a hidden alarm during the boomerang phase, switching to normal access after a milestone. It prioritizes ultimate protection over convenience, making it tougher for anyone to force you out of your bitcoin. If you’re curious about setting it up or the costs, it involves hardware, fees for services, and coordinating with others.

-------------------------

