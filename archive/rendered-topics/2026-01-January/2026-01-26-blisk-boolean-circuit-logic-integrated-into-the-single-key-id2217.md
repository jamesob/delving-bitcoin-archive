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

nkohen | 2026-01-30 20:39:16 UTC | #4

Cool construction!

I've been thinking about it for a bit and I think I'm a little confused as to why the ECDH and attached ZKPs are necessary? Would it not be sufficient to designate one party from each disjunction subtree in the CNF to generate a secret for that subtree at random and to share it with all of the other members of the subtree (and then each member signs the public key and all parties, in the subree or not, verify these signatures and ensure that all members used the same key)? If I'm understanding correctly, during the ECDH-based setup you do have to store the aggregate secret for a disjunction subtree since you cannot non-interactively recompute it, and at that point it isn't stateless so why not just store a randomly generated/unstructured secret? Is there a benefit to ECDH-based key agreement over private broadcast?

Regardless of how the key agreement happens, so long as all parties agree on aggregate disjunction keys during setup, no adversary will be able to stop an honest qualified set from signing, and no adversary will be able to sign if they have only corrupted a non-qualified set (since they will be missing one fully-honest secret from a disjunction they are not a part of). This should still lead to a direct reduction to the security of MuSig.

I believe the idea of writing a policy in CNF where each disjunction subtree corresponds to a group of people who hold some replicated secret is called Replicated Secret Sharing (RSS). My understanding is that normally a RSS DKG is very simple and consists of just sharing some secrets and verifying everyone has the same secret using authentication.

If I'm not mistaken, it is also possible to do key rotation within RSS without needing ZKPs, though I don't know much in detail about this.

-------------------------

olkurbatov | 2026-02-01 20:57:29 UTC | #5

Thanks for the feedback!

Your intuition is right, we can realize CNF-by-clauses using replicated secrets: pick a per-clause secret, privately distribute it to all members of that OR-subtree, and then ensure everyone agrees on the resulting clause public key (via signatures, for example). If all parties agree on the clause keys, we still get the “qualified set can sign / non-qualified set can’t”.

But, I think the BLISK approach has some additional advantages:

1. **It avoids storing new secret material**. With RSS/private broadcast, every member stores extra clause secrets (one per clause they belong to), and you need secure authenticated channels (or a private broadcast primitive) plus a verifiable key rotation for those secrets (which might be difficult). BLISK’s OR-gate approach aims to keep the only long-term secrets as the parties’ existing keys and to make clause material *derivable* via key agreement rather than *distributed*.
[quote="nkohen, post:4, topic:2217"]
If I’m understanding correctly, during the ECDH-based setup you do have to store the aggregate secret for a disjunction subtree since you cannot non-interactively recompute it
[/quote]
Quoting this: no, you don't have to store the aggregated (to be more precise, **shared**) secret, the secret for the clause is derived based on your secret and public values in the path. So, there is a state (and it's fair to call it stateful construction), but the state is shared among all participants, it can't be corrupted (only lost), and it's literally the set of public keys (representing nodes). 

2. **Prevent OR-gate cheating without requiring everyone in the clause to co-sign the resolution.** If a “resolver” can publish an arbitrary public key for an OR node, they can pick a key known only to them, breaking liveness/semantics for other authorized members (even if the unforgeability of MuSig remains intact). With RSS, your “everyone signs the clause pubkey” approach fixes this, but it changes the coordination/liveness model: now clause resolution requires collecting endorsements from the entire clause (or some approval rule), which can be painful for large/overlapping clauses or partially offline participants. BLISK instead aims for "any eligible member can resolve", and everyone else can verify correctness non-interactively. 
3. **Rotation is where the ZK verifiability really matters.**
In an RSS design, rotation generally means re-sharing/re-distributing clause secrets (interactive + authenticated private channels) or running a fresh DKG-like step per affected clause. In BLISK, we pay proof costs so that updating/resolving internal-node keys remains publicly checkable without re-running a distribution protocol. When one party is updating their key: 1) public keys of all other participants remain the same; 2) but they need to verify proofs and update a common state with the new version.

So I agree ECDH isn’t “cryptographically necessary” in principle; it’s a concrete way to get the local-derivability property (“any authorized member can derive the OR secret”) without introducing fresh replicated secrets. And the ZKPs are there to enforce the well-formedness/consistency of OR resolutions without requiring all clause members to participate in the policy setup (and while allowing individual keys to be updated by their owners non-interactively).

-------------------------

ZmnSCPxj | 2026-02-03 10:59:53 UTC | #6

Correct me if I am wrong, but I believe MuSig-in-MuSig (e.g. `A ^ (B ^ C)`) has no security proof yet?

Or do you flatten them into a single level on compilation?  Your description, especially with regards to using ECDH for OR gates, seems to imply that you retain the structure at all levels of the expression tree?

-------------------------

olkurbatov | 2026-02-03 22:51:03 UTC | #7

BLISK restricts policies to CNF: a single top-level AND over OR clauses, because the only MuSig2 KeyAgg happens at the top.

So an expression like `A ∧ (B ∧ C)` is compiled/normalized to `A ∧ B ∧ C` (three 1-literal clauses), i.e., just standard MuSig2 3-of-3,  no additional security proof is required beyond ordinary MuSig2 for that part.

-------------------------

ZmnSCPxj | 2026-02-03 23:48:24 UTC | #8

I see.  I believe even complex nested AND / OR sequences should be convertible to an equivalent CNF (admittedly it has been a long while since I did digital logic, I kinda wonder at NOT gates, which have no true equivalent in Bitcoin, but if the original expression has no NOT gates and IF gates then the CNF should not have NOT gates either?).

On the other hand, I suppose the MuSig-in-MuSig stuff matters for the use-case where one of the participants is secretly a multisignature (or more generally, itself has a complicated but hidden Boolean-gate policy), which would then not fit in this framework, which seems to require that the entire policy be revealed up front, then compiled to CNF.

In practice the secrecy may not be needed for privacy, but for protocol simpllfication --- you just say `A ^ B` where you are `A` and the other side `B` is free to use a simple singlesig or a complex Boolean-gate policy without having to bother you with TMI.

-------------------------

ZmnSCPxj | 2026-02-03 23:51:58 UTC | #9

On the other hand if we have a generic standardized Boolean-policy-to-CNF compiler, maybe we can have a BOLT spec where you just say "`A ^ B` is the base policy for the channel, both sides are then free to replace their term with another Boolean policy that they reveal to the counterparty, then compile the policy to CNF and use Blisk".

-------------------------

