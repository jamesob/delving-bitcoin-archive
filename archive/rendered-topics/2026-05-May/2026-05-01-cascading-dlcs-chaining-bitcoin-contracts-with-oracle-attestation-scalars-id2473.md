# Cascading DLCs: Chaining Bitcoin Contracts with Oracle Attestation Scalars

Pedro_Martins_Novaes | 2026-05-01 14:41:01 UTC | #1

# Cascading Discreet Log Contracts: Oracle Signatures as Contract-Activation Secrets

Hello Delving Bitcoin,

I'm sharing a whitepaper proposing **Cascading Discreet Log Contracts**, or **cDLCs**.

The main claim is narrow:

> A DLC oracle signature scalar can be used not only to settle a parent DLC, but also as the hidden scalar of adaptor signatures that activate a child DLC funding transaction.

Let the oracle's attestation for outcome \(x\) be

\[
s_x = r_o + H(R_o \| V \| x)v \pmod n
\]

with public attestation point

\[
S_x = s_xG = R_o + H(R_o \| V \| x)V.
\]

For a bridge transaction \(B_e\) spending a parent outcome output into a child DLC funding output, each required signature is prepared as a Schnorr adaptor signature using

\[
T_e = S_x.
\]

With \(e = H(R^* \| P \| m_e)\), the adaptor verifies before oracle resolution:

\[
\hat{s}G = R^* - S_x + eP.
\]

After the oracle publishes \(s_x\), any holder of the required adaptor state computes

\[
s = \hat{s} + s_x \pmod n,
\]

and obtains a valid Schnorr signature:

\[
sG = R^* + eP.
\]

So the same scalar \(s_x\) does two things:

1. completes the parent DLC outcome signatures;
2. completes the bridge transaction that funds the child DLC.

Before \(s_x\) is known, completing \(B_e\) requires either forging the required Schnorr signatures or finding \(\log_G(S_x)\). After \(s_x\) is published, the bridge is completable by any party holding the complete prepared adaptor state.

This gives a native Bitcoin construction:

\[
\mathrm{Funding}_i
\rightarrow
\mathrm{CET}_{i,x}
\rightarrow
B_e
\rightarrow
\mathrm{Funding}_j.
\]

No new opcode, covenant, or consensus change is required. Bitcoin validates only ordinary signatures and timelocks; the cascading semantics are carried by the oracle scalar and pre-signed adaptor state.

The attached paper formalizes the construction, the security boundary, the required state-retention assumptions, and the machine-checked algebra in Ada/SPARK.

The specific review request is:

> Under the stated assumptions, is the claim correct that a DLC oracle attestation scalar can serve as the adaptor secret for a child-funding bridge transaction, thereby allowing one DLC outcome to activate another DLC without consensus changes?

 repo: https://github.com/dev865077/NITI

Whitepaper: https://github.com/dev865077/NITI/blob/ab9363f2d118f799a6d80e77519527ef07aff7d6/Cascading%20Discreet%20Log%20Contracts%20(cDLCs).pdf

-------------------------

