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

