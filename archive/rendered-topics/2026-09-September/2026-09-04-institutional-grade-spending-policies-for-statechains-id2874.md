# Institutional Grade Spending Policies for Statechains

evd0kim | 2026-09-04 16:44:09 UTC | #1

## Summary

This post addresses a shortcoming of the MercuryLayer project: advanced spending policies. The existing open-source implementation uses a "single signer" approach with a dedicated private key per coin. That model offers no authorization structure, which corporate entities with established approval processes need.

The approach below combines **B**oolean circuit **L**ogic **I**ntegrated into the **S**ingle **K**ey (BLISK) with the blind-signed, MuSig2-based statechain transfer protocol. It turns out that this requires almost no change to the MercuryLayer specification, and keeps BLISK entirely on the client side. Mercury's protocol rests on exactly the principle that makes this possible: the client builds the aggregate key, the aggregate nonce and the challenge, and the server returns a blind share.

## Intro

A naive way to split the key in MercuryLayer is Shamir Secret Sharing (SSS), especially if a Trusted Execution Environment (TEE) coordinates the shares. That simplification holds for blind statechain transfers, because the TEE acts for the user and sits inside the user's trust perimeter. But more advanced approaches without a TEE should also exist.

One of them is Iceberg ([paper](https://eprint.iacr.org/2026/1757), [implementation](https://github.com/furszy/benchmark-iceberg)), which formalises *nested threshold multi-signatures*. It thresholdizes one participant inside an existing multi-signature execution while leaving the outer protocol, including its nonce exchange and message flow, unchanged. The paper targets Lightning channels, and it also analyses why FROST does not fit inside a two-party MuSig2 protocol. MercuryLayer's blind MuSig2 signing should be compatible with the same idea, although that is out of scope here.

This post takes a different route: [BLISK](https://hackmd.io/@olkurbatov/HJm5h0JH-l). BLISK is not a threshold scheme and does not compete with Iceberg on the same axis. It expresses *structure* rather than *cardinality*. Policies such as

> - the CEO **OR** the CFO must approve, **AND** at least one board member
> - two executives **AND** a compliance officer **AND** (legal **OR** counsel officer)
> - Carol can send the payment if (Alice **OR** Bob) approves

cannot be expressed by any $t$-of-$n$ threshold signature, because a threshold can only count. Bitcoin script can express them, but only by revealing them explicitly onchain and by paying for the extra witness data. BLISK maps the whole circuit to a single public key. Members also keep their long-term keys: no distributed key generation is required.

## Similarities

In the proposed scheme, BLISK becomes a virtual MuSig2 participant. The abstraction is thin enough that institutional policies and ordinary single signers can send and receive coins in the same anonymity set. If the blinding holds, the Statechain Entity (SE) cannot tell an institution from a wealthy individual.

For one top-level gate of the circuit (say gate 1, from Bob's viewpoint), BLISK calculates:

$$ s_{\mathrm{gate1}}^{Bob} = \underbrace{\sum_{l=1}^4 b^{\,l-1} \cdot rB_{\mathrm{gate1},l}}_{\text{effective nonce } (k_j)} + c \cdot \underbrace{a_{ABC}}_{\text{KeyAgg coeff}} \cdot \underbrace{H\big(H([sk_B]pk_A)\cdot pk_C\big)}_{\text{secret of this OR gate } (q_j)} $$

where

* $b$ is the MuSig2 nonce coefficient,
* $c$ is the signature challenge,
* $a_{ABC}$ is the aggregation coefficient of the top-level gate,
* the innermost term is the ECDH secret of a nested gate $(A \vee B) \vee C$, as Bob derives it.

Define the compact terms

$$ k_j = \sum_{l=1}^{4} b^{\,l-1} r_{j,l}, \qquad q_j = H\big(H([sk_B]pk_A)\cdot pk_C\big) $$

The BLISK expression is then a per-gate version of the client share $z_B$ in the MercuryLayer [protocol specification](https://github.com/mercury-layer/mercurylayer/blob/f130a3be38fb742916bcef7ebabf86741c3d08ab/docs/protocol.md). Many gate-level shares compress into one virtual Mercury client share:

$$ z_B = \sum_j z_j = \sum_j \left(k_j + c \cdot a_j q_j\right) = k_B + c \cdot x_B $$

with $k_B = \sum_j k_j$ and $x_B = \sum_j a_j \cdot q_j$.

## Does blinding work?

The two schemes differ only in the challenge term. Standalone BLISK uses the ordinary MuSig2 challenge over the *aggregate* nonce $R$:

$$ c = H(pk_{\mathrm{agg}}, R, m) $$

MercuryLayer uses a blinded challenge

$$ c_{\text{Mercury}} = e + \alpha $$

where $e = H(P, R, m)$ is the ordinary challenge and $\alpha$ is the client's blinding nonce. BIP327 derives [one session-wide challenge](https://github.com/bitcoin/bips/blob/09e21036a4001fe6c9ba65c1d3a39b737768132f/bip-0327.mediawiki) $e$ from the aggregate nonce, the aggregate key and the message; every partial signature contains that same $e$, multiplied by the signer's own key-aggregation coefficient and secret key.

So each gate-level share must use the blinded challenge instead:

$$ z_j = k_j + c_{\text{Mercury}} \cdot a_j \cdot q_j $$

When BLISK sums its shares, the blinding factor $\alpha$ then aligns with the SE's contribution, and verification succeeds without any circuit member learning the unblinded challenge $e$. From here on, $c$ means $c_{\text{Mercury}}$.

The SE answers with its own nonce $k_S$ and private share $x_S$:

$$ z_S = k_S + c \cdot x_S \quad\Longrightarrow\quad z_S \cdot G = R_S + c \cdot X_S $$

which the client can check directly. The client then forms $z = z_S + z_B$, and with $P = X_S + X_B$:

$$ z \cdot G = (R_S + c \cdot X_S) + (R_B + c \cdot X_B) = R_S + R_B + c \cdot P $$

Mercury builds the total commitment as $R = R_S + R_B + \alpha \cdot P$, so substituting $c = e + \alpha$ gives

$$ z \cdot G = (R_S + R_B + \alpha \cdot P) + e \cdot P = R + e \cdot P $$

This is the standard Schnorr verification equation.

## Key reassignment

For convenience, let's switch to [MercuryLayer's original notation](https://github.com/mercury-layer/mercurylayer/blob/f130a3be38fb742916bcef7ebabf86741c3d08ab/docs/protocol.md): $o_i$ is the owner's private key share and $O_i = o_i \cdot G$ its public form.

A transfer requires Owner 1 to obtain a blinding factor $x_{SE}$ from the SE and add it to $o_1$. With BLISK, no single party holds $o_1$ anymore:

$$ t_1 = x_{SE} + o_1 = x_{SE} + \sum_{i=1}^{n} a_i \cdot x_i $$

The useful observation is that this is still linear. A coordinator (not a TEE) can therefore assemble the blinded sum for any quorum that resolves the policy root, and the same coordinator can drive the backup-transaction signature.

To make the summation blind to the coordinator, each pair of contributing members $(i,j)$ derives a mask by key agreement:

$$ m_{ij} = H(x_i \cdot X_j \,\Vert\, \mathrm{sid}) = H(x_j \cdot X_i \,\Vert\, \mathrm{sid}) $$

Member $i$ adds $+m_{ij}$ when $i < j$ and $-m_{ij}$ when $i > j$, so its total mask is $m_i$ and

$$ \sum_i m_i = 0 $$

The members already hold the keys for this, so no extra round is needed. The session identifier $\mathrm{sid}$ keeps the masks fresh, which matters because the contributing set may differ between transfers.

The transfer starts from a single initiator. The initiator must be a quorum member and must **not** be the coordinator. It requests $x_{SE}$, which the SE issues once and publishes as $X_{SE} = x_{SE} \cdot G$. The initiator (member 1) computes

$$ u_{1} = x_{SE} + a_{1} \cdot x_{1} + m_{1} $$

and every other member independently computes

$$ u_i = a_i \cdot x_i + m_i $$

Each sends its value to the coordinator, which sums $t_1 = \sum_i u_i$ and checks

$$ t_1 \cdot G = O_1 + X_{SE} $$

The coordinator does not know $x_{SE}$, so it cannot recover $o_1$. The initiator knows $x_{SE}$ but never sees $t_1$. Splitting these two roles is what protects the quorum against a malicious insider. Note that the check detects a bad contribution but does not identify its author. On failure, initiator and coordinator have to restart with a fresh $x_{SE}$.

The receiving side is symmetric, and falls back to plain MercuryLayer if the coin goes to a single-key owner. The SE expects the subtraction

$$ t_2 = t_1 - o_2 = t_1 - \sum_{j} b_j \cdot y_j $$

The receiving quorum derives pairwise masks $m'_j$ in the same way. Its initiator holds $t_1$ and sends $w_{1} = t_1 - b_{1} \cdot y_{1} + m'_{1}$; every other member sends $w_j = -\,b_j \cdot y_j + m'_j$. The coordinator sums $t_2 = \sum_j w_j$ and checks

$$ t_2 \cdot G = (O_1 + X_{SE}) - O_2 $$

using only public values. Here too the initiator and the coordinator must be different parties: together they hold $t_1$ and $t_2$, and therefore $o_2$.

The coordinator then sends $t_2$ to the SE, and the protocol rejoins the normal MercuryLayer transfer:

$$ s_2 = s_1 + t_2 - x_{SE} = s_1 + x_{SE} + o_1 - o_2 - x_{SE} = s_1 + o_1 - o_2 $$

The aggregate public key $P$ is unchanged, because $s_2 + o_2 = s_1 + o_1$. The SE then deletes $s_1$ and publishes $S_2$ with the signature count $K$, and every receiving member checks $P = S_2 + O_2$ against the published share.

-------------------------

