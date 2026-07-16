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

MartinS84 | 2026-07-16 11:48:03 UTC | #2

I dont understand the difference between 1 and 2. they both have in common that the public key is exposed. I think a first step should be to no longer allow transactions to these adresses. That shouldnt be a problem to anyone but it disincentivices using such adresses. 

> The network does not typically intervene to “rescue” funds in such cases

Because it cant, the network cant know wether a key is from an insecure seed generation. But we do have adress checksums to prevent user error for example.

> **Rate-limit spends (e.g., via the “Hourglass” proposal).**

Thats what i would prefer, although its a more complex solution than freezing or not freezing.

I would say theres 2 Situations, before Q-day, after Q-day. And Q-day is not neccessarily one day, because on one day one company might be able to crack those adresses, and at some point maybe your smartphone can. 

Anyway. Ideally, nothing should be frozen before Q-day, and everything should be frozen after Q-day. But we dont know when that is, so we need 1 solution that fits both Situations.

The hourglass proposal is OK-ish before Q-Day, because for the few people that might be in jail right now or for some other reason cant move their bitcoin, this leaves a path to slowly access their bitcoin. IMO thats not going to be a lot of People/Bitcoin anyway. Satoshi wont come back ;)

And the hourglass proposal is also OK-ish after Q-Day, because the stolen Bitcoin will basically behave like a blockreward regarding inflation. And because miners can decide which TX are mined, it might even be a blockreward, because anyone with a QC can pay the miners to include his "stolen" bitcoin over a competitors "stolen" bitcoin. So they could share the loot with a miner.

plus of course, you can make some bip39-proof to spend all vulnerable coins, this can be done as softfork aswell, because it tightens the rules. The rules are then: you need the signature + the bip39 proof.

So the only coins affected by the limitation/freeze are some really old coins. What are the chances that someone actually owns these, cant move them now, but can move them later? Is satoshi in jail for 25 years and has his keys hidden somewhere? So really no one is affected.

> **How can the network possibly reach a consensus on *when* to activate this switch?**

I dont know, the problem is, we have told the narrative, that bitcoin is going to last forerver "as is", and thats not true. Cryptography has a limited lifetime before it needs to be replaced. 

An actual human decision has to be made.

-------------------------

