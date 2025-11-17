# Thoughts on a network policy for quantum-vulnerable UTXOs (long-exposure quantum attacks)

show1225 | 2025-11-17 16:30:53 UTC | #1

I’d like to start a discussion on a potential policy framework for handling quantum-vulnerable UTXOs after “Q-day” (the day a practical quantum attack becomes public).

It seems we have two primary categories of vulnerability to consider, each potentially requiring a different response.

---

### **Category 1: Address type vulnerability (P2PK, P2TR)**

**1: Freeze transactions from these address types.**

* **Rationale:** Assuming a sufficient warning period where post-quantum (PQ) address types are introduced and adopted, any transaction spending from these legacy types *after* Q-day should be considered compromised.

**2: Rate-limit spends (e.g., via the “Hourglass” proposal).**

* **Rationale:** While full confiscation is undesirable, allowing a sudden market dump of compromised coins could severely disrupt network stability and miner incentives. A consensus-enforced rate limit could mitigate this shock.

* **Note:** This could also create a window for “white hat” quantum operators to recover funds and return them to owners who provide some form of off-chain (physical or cryptographic) proof of ownership.

---

### **Category 2: Address re-use vulnerability**

**1: Leave them as-is (allow them to be spent).**

* **Rationale:** This could be viewed as a form of user error, analogous to insecure seed generation. The network does not typically intervene to “rescue” funds in such cases, and responsibility lies with the user.

* **Note:** In my opinion, most funds in these addresses are likely accessible (as they have been used once). As Q-day approaches, we can expect most address owners to proactively move these funds to unspent / PQ addresses, minimizing the eventual damage.

**2: Enforce a “cold sleep” (freeze) on these UTXOs.**

* **Rationale:** This approach would prevent a massive, immediate theft of all funds in this category on Q-day. The freeze could be maintained until a robust, consensus-approved recovery method is developed (e.g., a ZK-proof of seed phrase ownership) to allow legitimate owners to reclaim their coins. This balances preventing a catastrophic theft event with maximizing the chance of recovery for owners who were unable to move their funds in time.

#### **Activation Timing**

* Activating a freeze or rate-limit **too early** could penalize legitimate, non-quantum-compromised users who haven’t migrated yet.

* Activating **too late** would mean we fail to prevent the initial, massive theft event.

**How can the network possibly reach a consensus on *when* to activate this switch?**

---

I’m interested in hearing the community’s opinions, corrections, or alternative ideas on this framework. Are there other scenarios or policy trade-offs to consider?

-------------------------

