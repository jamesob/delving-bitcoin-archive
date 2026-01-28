# BLISK: Boolean circuit Logic Integrated into the Single Key

olkurbatov | 2026-01-27 14:56:39 UTC | #1

Hello everyone,

We’ve been working on teaching EC points to “understand” authorization policies expressed with boolean logic. We’d love feedback and ideas!

TL; DR. Bitcoin mainly gives two ways to express “who can spend”:

* Threshold / multisignatures (FROST / MuSig2): great privacy and efficiency, but they fundamentally express only cardinality (*k-of-n*).

* Scripts: more expressive, but limited in practice, and the policy can become visible when used.

We built a PoC framework that compiles a monotone boolean policy (AND/OR logic) over users’ long-term keys into a single signature verification key (one EC point). On-chain part remains boring: verification of one Schnorr signature against one key. All the policy “drama” happens off-chain.

What this means for Bitcoin:

* You can express policies that threshold signatures can’t capture faithfully, e.g. **`(A ∨ B) ∧ (C ∨ D)`** (structure, not just counting). Policies we have used for benchmarks are below:

  ![Screenshot 2026-01-27 at 01.17.36|308x500](upload://fMI9CIhaAJiWearTalGJFZHV8rC.png)

* Any compiled policy (despite of the policy complexity) produces one verification key, and spending produces one signature verified against it.

* The policy is open only fo signers, all external parties see only the single public key.

* No exotic primitives, just a combination of well-known cryptography:

  * MuSig2 for AND gates

  * ECDH for OR gates

  * NIZK (e.g., Bulletproofs) to make circuit resolution verifiable and prevent OR-gate cheating

* It requires a setup phase, but:

  * Key rotation is non-interactive (no re-DKG required)

  * Users can keep using existing long-term keys (unlike threshold DKGs that produce fresh key material)

References:

Paper: [https://eprint.iacr.org/2026/088.pdf](https://eprint.iacr.org/2026/088.pdf)

Blog explainer: [https://hackmd.io/@olkurbatov/HJm5h0JH-l](https://hackmd.io/@olkurbatov/HJm5h0JH-l)

Not a production-ready framework: https://github.com/zero-art-rs/blisk

Thanks!

-------------------------

evd0kim | 2026-01-28 10:30:50 UTC | #2

Thanks for this post.
There is a policy in the example:

```
let policy_str = "(policy my_policy (and (or A B) (or C D)))";
```

However, only A and C are co-signers in Musig2 which spends onchain. I guess B or D could not authorize any spending alone. Does policy signer resolves Musig2 setup and generates valid partial signature instead?

-------------------------

olkurbatov | 2026-01-28 15:25:28 UTC | #3

The policy `(and (or A B) (or C D))` has exactly 4 coalitions that can produce a valid signature: $\{A,C\}, \{A,D\}, \{B,C\}, \{B,D\}$.

We can represent this policy in the form of the following PK tree:

![Screenshot 2026-01-28 at 17.07.28|690x259, 100%](upload://shU4e3PImM1jTVW5bd0YyvkvIHI.png)

The verification key `P` is **not** a MuSig2 aggregation of all four parties. Instead, it's a MuSig2 aggregation of `P_left` and `P_right`, where:

```

P_left = hash(ECDH(P_A, P_B)) * G

P_right = hash(ECDH(P_C, P_D)) * G

```

so `P = MuSig2_KeyAgg(P_left, P_right)`.

To produce a valid under `P` signature, we need two parties who can access `ECDH(P_A, P_B)` and `ECDH(P_C, P_D)`, respectively. Either A or B can compute the left ECDH (using their private key with the other's public key), and either C or D can compute the right one. So, the set $\{B,D\}$ **can** produce a valid signature, but $\{A,B\}$ **cannot** because they both belong to the same `P_left ` branch, leaving the right branch unsatisfied.

[quote="evd0kim, post:2, topic:2217"]

However, only A and C are co-signers in Musig2, which spends on-chain.

[/quote]

In the code example, Alice and Carol are the active signers, but this is just one of the four valid coalitions. Bob and Dave (or any other valid pair) can sign in exactly the same way.

-------------------------

