# Silent Payments for Multiparty Wallets

olkurbatov | 2026-10-15 15:41:44 UTC | #1

**Abstract**. BIP-352 does not recommend FROST- / MuSig2-based inputs "due to lack of a security proof". This post describes a protocol that lets a multiparty sender perform a secure Silent Payment: the wallet computes an ECDH share as a weighted sum of the signers' contributions, and $\mathsf{AggDLEQ}$ aggregates their responses into a single BIP-374 proof.

**Notation**. $G$ is the generator of secp256k1's group, and $[a]G$ is scalar multiplication.

## A shared interface for multiparty keys
We can represent a wallet key as a linear combination of per-signer secrets:

$$
x=\sum_{i\in T}c_i\cdot k_i,\quad\text{where }(c_i,k_i)=
\begin{cases}
(\lambda_i,\ s_i)_{\textsf{FROST}}\\
(\alpha_i,\ x_i)_{\textsf{MuSig2}}\\
(\prod_{\mathrm{path}(i)}\alpha,\ x_i)_{\textsf{Nested MuSig2}}\\
(\alpha_i,\ \mathsf{sk}_{\lor,i})_{\textsf{BLISK}}
\end{cases}
$$

> * **[FROST](https://github.com/siv2r/bip-frost-signing)**: $x=\sum_{i\in T} \lambda_i s_i$ with Lagrange coefficients $\lambda_i$ 
> * **[MuSig2](https://github.com/bitcoin/bips/blob/927b6de9915c9262615a6399de51b200f81e5aa4/bip-0327.mediawiki)**: $x=\sum_i\alpha_i x_i$ with $\alpha_i=\mathcal{H}_{\mathrm{agg}}(\vec X,X_i)$ over the key list $\vec X$.
> * Analogy for [Nested MuSig2](https://eprint.iacr.org/2026/223) and [BLISK](https://delvingbitcoin.org/t/blisk-boolean-circuit-logic-integrated-into-the-single-key/2217) constructions should be clear.

The same with tweaks. BIP-32 and the taproot tweak add scalars to the key, and the even-Y rule requires multiplying it by a sign. So the input key is $A=[g]X+[\tau]G$ with $g=\pm1$ and $\tau$ (BIP-327's and BIP-445's parity and tweak accumulators). A corresponding secret is:
$$
a=g\cdot x+\tau=\sum_{i\in T}(g \cdot c_i)\cdot k_i+\tau 
$$

## Weighted ECDH
For each scan key, its input posts the ECDH share $C=[a]B_{\mathrm{scan}}$ with a $\mathsf{DLEQ}$ proof. The shares of all inputs are summed and multiplied by $\mathsf{input\_hash}$ into the shared secret $S$, and output $k$ pays $P_k=B_{\mathrm{spend}}+[t_k]G$ with $t_k=\mathcal{H}(S||k)$.

The linearity from the previous section gives us $C$ as:
$$
C=[a]B_{\mathrm{scan}}=\sum_{i\in T}[c_i]D_i+[\tau]B_{\mathrm{scan}},\quad D_i=[k_i]B_{\mathrm{scan}} 
$$

So, each signer can contribute $D_i$, and a coordinator combines the contributions. BIP-352 sketches this as $a_1\cdot B_{\mathrm{scan}}+\dots+a_t\cdot B_{\mathrm{scan}}$. But a deployable version needs the coefficients $c_i$, without which the sum is not $[a]B_{\mathrm{scan}}$, plus tweak and parity handling.

It's almost there, but there are two remaining things we need to take care of:
1.  The contributions must be verifiable: one malformed $D_i$ gives a wrong $C$, so the payment goes to an output the receiver can't detect.
2. The proofs must also be aggregated: BIP-375 carries one 64-byte proof per input and scan key.

## AggDLEQ
A $\mathsf{DLEQ}$ (Chaum-Pedersen) proof shows that $A=[a]G$ and $C=[a]B$ share a discrete logarithm. For a statement $(A,B,C)$ and an optional message $m$, the prover samples $r\xleftarrow{\$}\mathbb{Z}_q$, sets $R_1=[r]G$, $R_2=[r]B$, derives $e=\mathcal{H}_{chal}(A||B||C||G||R_1||R_2||m)$, and outputs $\pi=(e,s)$ with $s=r+e \cdot a$. The verifier recomputes $R_1=[s]G-[e]A$ and $R_2=[s]B-[e]C$ and accepts iff re-deriving challenge returns $e$. BIP-374 instantiates such a system on secp256k1.

$\mathsf{AggDLEQ}$ allows to combine all $D_i$ contributions into one Chaum-Pedersen proof for $(A,B,C)$, with $B=B_{\mathrm{scan}}$, in two rounds.

### Protocol

#### Context sharing
The coordinator sends to each signer the context $\mathsf{ctx}$: the signer set $T$, the coefficients $c_i$ and the tweak $\tau$, the key $A$, the base $B$, the transaction (template and inputs) and a session id. Each signer validates the context.

#### Round 1
Signer $i$ generates two nonces $r_{i,1},r_{i,2}$ and sends
$$
D_i=[k_i]B, \quad U_{i,1}=[r_{i,1}]G, \quad U_{i,2}=[r_{i,2}]G, \quad V_{i,1}=[r_{i,1}]B, \quad V_{i,2}=[r_{i,2}]B .
$$
From the list $L$ of round-1 messages, the coordinator and each signer compute:
$$
C=\sum_{i\in T}[c_i]D_i+[\tau]B,\quad \mathsf{sid}=\mathcal{H}_{\mathrm{sid}}(\mathsf{ctx}\|A\|B\|C\|L),\quad \rho_i=\mathcal{H}_{\mathrm{bind}}(\mathsf{sid}\|i),
$$
$$
R_1=\sum_{i\in T}[c_i](U_{i,1}+[\rho_i]U_{i,2}),\quad R_2=\sum_{i\in T}[c_i](V_{i,1}+[\rho_i]V_{i,2})
$$
and a challenge
$$
\qquad e=\mathcal{H}_{\mathrm{chal}}(A\|B\|C\|G\|R_1\|R_2)
$$

#### Round 2
Each signer $i$ answers:
$$
z_i=r_{i,1}+\rho_i \cdot r_{i,2}+e\cdot k_i .
$$

With $K_i=[k_i]G$ the public key of signer $i$, the coordinator checks every response on both bases:
$$
[z_i]G\overset{?}{=}U_{i,1}+[\rho_i]U_{i,2}+[e]K_i,\quad [z_i]B\overset{?}{=}V_{i,1}+[\rho_i]V_{i,2}+[e]D_i,
$$
and if everything is fine, aggregates responces:
$$
s=\sum_{i\in T}c_i \cdot z_i+e \cdot \tau, \quad \pi = (e,s) 
$$

> With $r=\sum_i c_i(r_{i,1}+\rho_i r_{i,2})$ we get $s=r+e\cdot a$, $R_1=[r]G$ and $R_2=[r]B$, so $[s]G=R_1+[e]A$ and $[s]B=R_2+[e]C$. These are exactly the points the BIP-374 verifier reconstructs, so $\pi$ is an ordinary BIP-374 proof.

## Aggregated Silent Payment
1. The coordinator sends $\mathsf{ctx}$ to the signers.
2. Each signer returns its AggDLEQ round-1 message together with its signing nonces (round 1 of BIP-327 or BIP-445).
3. The coordinator computes $C$ and the outputs $P_k$. The transaction and the signing message are now fixed.
4. The coordinator sends $L$ and the signing context.
5. Each signer returns $z_i$, and the coordinator combines $(C,\pi)$.
6. The coordinator sends $(C,\pi)$ and every co-input's proof. Each signer verifies them and recomputes the outputs itself.
7. Each signer returns its partial signature. The coordinator aggregates the signatures and writes $(C,\pi)$ into the BIP-375 fields.

A signer releases its partial signature only after it has checked every proof and recomputed every output, and that turns proof soundness into "the receiver finds this payment", whatever the coordinator does.

## Compatibility
- BIP-352. The wallet's input is an ordinary eligible input with key $A$.
- BIP-374, BIP-375. The unchanged verifier accepts AggDLEQ proofs byte for byte.
- BIP-327, BIP-445. Signing is untouched and runs on the same shares. AggDLEQ takes the same coefficients and tweak as the signing session.
- New messages appear only inside the wallet between signers. The AggDLEQ rounds are communicated only over the wallet's coordinator channel; PSBT doesn't need any additional fields.

## Security
The paper with security proofs is under internal review; we will share it here when it's available on eprint. Main claims:
- $(C,\pi)$ produced by AggDLEQ has a single prover's distribution, so it hides the wallet's structure;
- outputs are pseudorandom for a chain observer under GapCDH even when input keys repeat;
- the construction is sound: the receiver can detect the payment regardless of what the coordinator does (except with negligible probability);
- joint unforgeability: while AggDLEQ and MuSig2/FROST signing run concurrently on the same shares, corrupted signers and the coordinator can not forge a proof or a signature. This is the most interesting part: the assumption is DH-AOMDL (AOMDL + static DH helper), and its hardness is open. We budgeted it with the best attack to our knowledge, but we have no sketch of a generic bound or a reduction to plain AOMDL. Additionally, this isn't specific to multiparty wallets: a plain BIP-375 signer computes $[a]B_{\mathrm{scan}}$ for whatever scan key the PSBT names, so its key is a static DH oracle too.

Best, Oleks

-------------------------

