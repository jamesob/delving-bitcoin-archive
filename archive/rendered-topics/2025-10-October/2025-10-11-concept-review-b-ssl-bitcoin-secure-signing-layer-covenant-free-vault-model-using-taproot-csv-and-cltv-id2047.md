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

