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

