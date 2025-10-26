# Concept Review: B-SSL (Bitcoin Secure Signing Layer) — Covenant-Free Vault Model Using Taproot, CSV, and CLTV

ilghan | 2025-10-11 11:42:34 UTC | #1

Hi everyone,

I’m sharing a new vault construction concept called **B-SSL — Bitcoin Secure Signing Layer**, a covenant-free Taproot vault design for self-custody that eliminates the risk of permanent key loss while maintaining full on-chain enforceability.

It uses existing Bitcoin primitives (Taproot, `CHECKSEQUENCEVERIFY`, `CHECKLOCKTIMEVERIFY`) and defines three spending paths:

• **(A or A₁) + C** after **2h–15d (CSV)** — configurable user-initiated path
• **A + B** after **1y (CLTV)** — fallback recovery
• **(B or B₁) + C** after **3y (CLTV)** — custodian recovery / inheritance safeguard

A₁ and B₁ are mirrored backups of A and B.
Optionally, an off-chain “Convenience Service” (CS) can enforce delays and emit secret notifications to protect against physical attacks or coercion.

🧾 **Whitepaper:** [GitHub – B-SSL Whitepaper Repository](https://github.com/ilghan/bssl-whitepaper/tree/main)

**Goal:**
This work is shared for peer review and discussion before implementation.
Feedback on Miniscript structure, policy expressiveness, and operational safety is very welcome.

Thanks

Francesco Madonna
[francesco@bitvault.sv](mailto:francesco@bitvault.sv)

-------------------------

marathon-gary | 2025-10-11 12:50:39 UTC | #2

This seems similar to https://github.com/Blockstream/miniscript-templates/blob/56916f60782bd9525b8d5b5b2959acb2efee038b/mint-005.md

Have you reviewed this miniscript?

-------------------------

ilghan | 2025-10-11 18:15:40 UTC | #3

their system doesn’t solve the problem of the loss of the keys in non-custodial .

If my B-SSL system holds true, that’s a clear breakthrough for Bitcoin’s future !

![image|690x386](upload://ovvioHdOoonyrW1cuBWrWQUrxHw.png)

Its premises are outstanding:

* **mint-005** = *joint custody coordination* → ideal for institutions sharing funds.

* **B-SSL** = *sovereign vault + human fail-safes* → ideal for individuals and companies wanting to **never lose control nor keys**.

> 🔒 **B-SSL uniquely achieves total key-loss resilience while preserving 100 % user sovereignty.**

Let me know if any of you here are able to crack it

-------------------------

marathon-gary | 2025-10-17 16:56:09 UTC | #4

> **mint-005** = *joint custody coordination* → ideal for institutions sharing funds.


Your AI generated response misses nuance that the referenced miniscript could be done with oneself with distributed keys.

-------------------------

ilghan | 2025-10-17 18:40:47 UTC | #5

I don’t think if you distribute the keys to yourself you can ever solve the problem of the loss of said keys , for the simple fact that one person who controls everything can very well lose everything .

Breakthrough like this rarely happen , a trustless key recovery system is the missing piece for self-custody to become for everyone, given the major blocker is undoubtedly the fear of losing the keys.

B-SSL marks the end between non-custodial and custodial because thanks to it everything will be non-custodial. The only task custodians will have is to hold the keys of the users while getting paid for it , except they can never ever steal the funds.

It’s a gift of freedom for every user who would otherwise never self-custody because of the fear of losing the keys (like, 80% of the world population?!) and I’m so blessed to have conceived it after 1y of hard work .

Ad maiora

-------------------------

scoresby | 2025-10-17 19:01:58 UTC | #6

Could you describe a little more how the primary spending path works? 

If a user wants to spend coins, they sign with key A and they need C to sign as well, correct? How are you implementing the CSV timelock on this spend path? 

Let’s pretend that the user set this timelock to 1 day. The timelock will expire if their coins are in the wallet for more than one day, at which point, A and C can just sign with no delay. So what is the purpose this timelock serves? 

I always imagined a vault was where a user sends coins to their wallet and can leave their coins in that wallet for an unspecified length of time and then enforce a delay when spending out of that wallet (usually by spending to an intermediate address), but I don’t think the construction you propose creates this effect, unless I’m misunderstanding something.

-------------------------

ilghan | 2025-10-18 06:45:22 UTC | #7

The purpose of signer C is to enforce time delay. While CSV starts to tick after transaction was received, the Convenience Service (which holds signer C) enforces time delay off-chain. Think of it as a secure environment that you can only interact for cosigning purposes. This environment will start counting time delay from the moment it receives spending transaction and will co-sign only when time delay has passed.

-------------------------

scoresby | 2025-10-18 16:28:06 UTC | #8

Thank you for your reply. 

> This environment will start counting time delay from the moment it receives spending transaction and will co-sign only when time delay has passed.

So if a user sets their CSV timelock to 15 days, that timer begins as soon as they send coins into the wallet and at this point key C cannot sign transactions because of the CSV timelock. But if more than 15 days pass, the CSV timelock expires and key C *can* sign. However, it sounds like key C will still not sign because key C is controlled by the CS which only begins counting down its time delay when it sees a transaction spending funds *out* of the vault.

If this is the case, why bother with the CSV timelock at all? I don’t believe I understand the purpose of the CSV timelock. Most users presumably would like to keep their coins in the vault for more than 15 days.

-------------------------

ilghan | 2025-10-19 11:11:38 UTC | #9

I invite you to focus on the end goal which is how to avoid two custodians to collude and then to assess the system in its entirety, if you analyze it part by part by not looking at the bigger picture then it could be difficult to make sense of it.  If you want to join the Bitcoin Optech audio recap discussion by next Tue, maybe it could help  :

[https://x.com/bitcoinoptech/status/1979146810200301740](https://x.com/bitcoinoptech/status/1979146810200301740)

*Francesco Madonna* 

*Founder,CEO @ [https://www.bitvault.sv/](https://www.bitvault.sv/)*

-------------------------

Bicaru20 | 2025-10-25 22:40:46 UTC | #10

**Hi,**

If I understand correctly, the main goal is to prevent key loss from being permanent and to provide a way for the user to recover the funds. However, I’m struggling to see how this achieves that because:

* If the user loses **Key A** (the primary key), the only way to recover the funds is through **Path 3** after a long delay (e.g., 3 years). This effectively means trusting custodians who hold shares like B1 and C.

* On the other hand, if the user doesn’t lose **Key A**, why would they use **Path 1** or **Path 2**, since they still have full control of the private key?

I think I am missing something :sweat_smile:

-------------------------

ilghan | 2025-10-26 11:32:35 UTC | #11

Hi,

The main **goal** **of** **B-SSL** is to **make total key loss non-fatal** while keeping the user *fully sovereign* and *on-chain verifiable*.
It achieves this with **timed Taproot spending paths** — not trusted intermediaries — all enforced by Bitcoin consensus rules.

---

**Vault Architecture**

* **Path 1 — `(A or A₁) + C after 2 h – 15 d (CSV)`**
  → everyday spending path with a short, configurable delay (2 hours – 15 days).
  Adds protection against hacks, coercion, or hasty moves.

* **Path 2 — `A + B after 1 y (CLTV)`**
  → **annual self-rotation path.**
  The user pre-signs a one-year transaction when the vault is created.
  If they’re active, they cancel and re-sign it yearly.
  This rotation (which can also be automated before recurring to the manual broadcasting) , keeps the vault “alive,” continually resetting all long-term delays —
  so effectively, the vault *renews itself every ≈2 years* and funds roll into a fresh vault.

* **Path 3 — `(B or B₁) + C after 3 y (CLTV)`**
  → **disaster-recovery / inheritance path.**
  Triggered only if the user has disappeared entirely.
  Custodians cannot act before 3 years — the delay is enforced on-chain.

If the user wants to bypass the legal framework for disaster recovery (although the incentive of the custodians is actually to comply), there are some adjustments which can be made:

  **Adjustment:**
  The *pre-signed transaction* for Path 3 doesn’t release funds.
  It sends them into a **new B-SSL vault**, effectively *rebuilding the vault structure on-chain*.
  This way, even in a disaster, custodians can only **re-lock the coins** into the next vault, or freezing the funds by refusing to broadcast the PSBT, but never redirect or steal them.

---

**Chained Vaults — Continuous Safety**

Users can create a **chain of vaults** (V₁ → V₂ → V₃ → V₄…), each with its own key set (A,A₁,B,B₁,C). - in batches of two vaults each time (this not to complicate the setup too much), so that once one vault expires , it creates a new one.
At any time, coins flow into the *next vault*, keeping security continuous and reducing the chance of irreversible loss to near zero.

If the user disappears after year 2 and **Path 2** cannot be broadcast because A and B are missing:

**→ There are only two outcomes:**

1. **Legal route:** If the user doesn't create PSBT for path 3--> the two custodians (holding B₁ and C) can release funds only after verification of ownership or inheritance, under legal supervision.
2. **Cryptographic route:** the user does create a PSBT for path 3--> **Path 3** activates after 3 years — a **deadman-switch recovery path** that rebuilds the vault automatically.

In case the user goes for the cryptographic route (2) , **malicious custodians can’t steal funds**:

* Before 3 years → impossible (consensus lock), visibility of the unsuccessful attempts (thanks to the 3y timelock)
* After 3 years → any collusion only results in **frozen funds**, not theft, since the PSBT destination is fixed and they can’t change it.

---

**Security Outcomes**

* As long as the user holds A/A₁ + B, they’re fully sovereign.
* Losing all keys isn’t fatal — funds can recover through Path 3.
**Custodians gain nothing from bad behavior: they can only freeze funds, not take them.**
* The vault chain ensures perpetual safety and recoverability — all on-chain.


---

 **Summary**

B-SSL makes key loss *recoverable*, spending *delay-protected*, and recovery *trust-minimized*.
After 2 years, the user either regains control legally or through the **3-year deadman-switch rebuild path**,
ensuring that **no custodian can ever steal** — only protect, cooperate, or freeze (but freezing isn't an option for them, they can only harm their reputation thereby ending their business, as nobody will trust them anymore. However, funds are still locked, they cannot steal, so they have no inceantive to behave badly and, in any case, they can be sued and forced to move)

I think both the outcomes (1) and (2) are gold if compared to the fact that today if you lose the quorum of the keys the funds are gone, and as a result of B-SSL, **with this system,  it makes no sense for the user to hand over the ownership of the funds to a custodian, moreover to pay him for that service, when the user can only hand over to them the keys, but not the ability to steal , lose, or re-hypothecate funds.**

-------------------------

ilghan | 2025-10-26 11:57:22 UTC | #12

![image|690x274](upload://3RJxVpv8TF1emTReouB9q3AfAjT.png)

This means **B-SSL preserves absolute sovereignty during life** and **controlled recoverability after death or loss**. Worst case scenario is freezing which is a neutral state and only harms the custodian's reputation, while the user has unlimited time to take legal actions.

-------------------------

