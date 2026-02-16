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

olkurbatov | 2026-02-04 14:44:56 UTC | #10

[quote="ZmnSCPxj, post:8, topic:2217"]
On the other hand, I suppose the MuSig-in-MuSig stuff matters for the use-case where one of the participants is secretly a multisignature
[/quote]
Yes, it would be very useful! The main limitation is that we can't (or at least I don't know how yet) have a multisignature public key as a child of an OR gate; ECDH doesn't work in this case. 

[quote="ZmnSCPxj, post:9, topic:2217, full:true"]
On the other hand if we have a generic standardized Boolean-policy-to-CNF compiler, maybe we can have a BOLT spec where you just say “`A ^ B` is the base policy for the channel, both sides are then free to replace their term with another Boolean policy that they reveal to the counterparty, then compile the policy to CNF and use Blisk”.
[/quote]
Wow, that's another great idea, I didn't think about it from this side. So, for instance, two organizations can create a channel based on their internal policies and make a cooperative close only if the policies of both are satisfied? Need only to think about how to do BLISK-based HTLCs.

-------------------------

ZmnSCPxj | 2026-02-05 00:29:43 UTC | #11

[quote="olkurbatov, post:10, topic:2217"]
Need only to think about how to do BLISK-based HTLCs.
[/quote]

This seems trivial?  An HTLC is just `(A & preimage) || (B & blockheight)` and you can just compile the individual policies for `A` and `B` and compile them into two single pubkey point, and just use tapleaves for `(policy-A &  preimage)` and `(policy-B & blockheight)`?  You kinda need SCRIPT for hashlocks anyway.  An optimization is to also add a `policy(A & B)` keypath spend.

-------------------------

ZmnSCPxj | 2026-02-05 00:28:17 UTC | #12

The goal here is to have a generic k-of-n Lightning Network node.  Channels require a `A & B` between the channel parties, and for this "k-of-n LN node" future, either of `A` or `B` or both may actually be a k-of-n.

Now, I believe ANY k-of-n policy can be iterated as a DNF, e.g. for a 2-of-4 of `A B C D`, that would be `((A & B) | (A & C) | (A & D) | (B & C) | (B & D) | (C & D))`.  And the DNF form can be converted to an equivalent CNF form (they are just normal forms and I believe any nested AND-OR expression can be converted to either form).

Thus, we can consider ANY general policy of nested AND and OR gates, then flatten them to AND-of-OR and feed it to Blisk.

Now the issues here for a k-of-n LN node are two-fold:

* Creating a new channel state basically requires a partial signature from the other side, but without the full signature being generated on our side --- the point is that the full signature can only be safely generated on unilateral closure.  The CNF at the Blisk level may require the creation of the FULL signature, which is unsafe --- storing the full signature is unsafe as old state can now be posted by anyone who gets access to your stored full signatures.  I would need to look more deeply into this to see if this is actually safe; my intuition is that there will be at least one term in the CNF that is controlled only by signatories of one side, which would serve as the decision-makers for when to do a unilateral close later (and create the full signature).
* After creating a new channel state, we need to invalidate old state.  There is simply no way to do this that still respects the shachain requirement of BOLT, so we should "just" drop the shachain requirement.  Without the shachain requirement, the public key for the revocation key can be generated using flatten-to-CNF-then-Blisk and then at revocation time, the same calculations are done on private keys this time, then the resulting private key is the revocation private key that you hand over to the other party to revoke your old state.

Both the above can be sidestepped by switching to Decker-Wattenhofer, BTW: each state change both creates the new state and invalidate old state in a single atomic step of creating the FULL signature under Decker-Wattenhofer.  And Decker-Wattenhofer, unlike Decker-Russell-Osuntokun, does not require a consensus change.

Also this scheme of "flatten to CNF form and then Blisk" may be considered a generalization of this technique: https://delvingbitcoin.org/t/flattening-nested-2-of-2-of-a-1-of-1-and-a-k-of-n

-------------------------

nkohen | 2026-02-05 15:24:09 UTC | #13

[quote="ZmnSCPxj, post:6, topic:2217"]
Correct me if I am wrong, but I believe MuSig-in-MuSig (e.g. `A ^ (B ^ C)`) has no security proof yet?
[/quote]

Just wanted to chime in here since it appears relevant for this discussion, but I have a paper titled "Nested MuSig2" that should be appearing on the ePrint Archive within a few days detailing a secure way to nest MuSig2 keys! This should provide more flexibility in how private sub-AND clauses want to be.

-------------------------

juja256 | 2026-02-05 15:40:27 UTC | #14

[quote="ZmnSCPxj, post:12, topic:2217"]
Also this scheme of “flatten to CNF form and then Blisk” may be considered a generalization of this technique: https://delvingbitcoin.org/t/flattening-nested-2-of-2-of-a-1-of-1-and-a-k-of-n

[/quote]

Hey! It’s literally how our [reference implementation](https://github.com/zero-art-rs/blisk) works. For any input S-expression policy drawn in monotone boolean functions (only OR, AND gates) and not necessary in CNF the compiler compiles it to CNF under the hood. Moreover the compiler supports keywords that express precompiled policies: e.g.  `(threshold 3 A B C D E)` compiles to 3-of-5 threshold policy in CNF. I think it’s one of the most amazing features in BLISK: it could handle merely any monotone policy (even a large one such as 11-of-15 threshold Liquid federation policy). You could try out by writing some S-expressions, compiling them, resolving and enduring MuSig2 session as described in the [example](https://github.com/zero-art-rs/blisk/blob/143ffbd0866084a52142ccc7d8e85351ce5a5496/src/signer.rs).

-------------------------

olkurbatov | 2026-02-16 20:53:40 UTC | #15

We can also extend BLISK with DLC-style adaptor points. Precisely, we can perform adaptor/condition-dependent compilation.

## The Idea
Consider the policy $A\land(B \lor C)$ - Alice is required, plus one of Bob or Carol. Now suppose we want a fallback: if some external condition is met (a price change, timeout, etc.), Bob or Carol can spend without Alice.

We can do this by wrapping Alice's input node in a disjunction with an adaptor point $T_a$ derived from an oracle's pre-committed parameters:
$$T_a = R_i + \mathcal{H}(P_O, R_i, e)\cdot P_O,$$
where $P_O$ is the oracle's long-term public key, $R_i$ is a pre-committed nonce for attestation slot $i$, and $e$ is the event descriptor (e.g., encoded price or timestamp).

Before the oracle signs $e$, nobody knows $\mathsf{dlog}(T_a)$. After attestation, the oracle publishes 
$$s_i = r_i + \mathcal{H}(P_O, R_i, e)\cdot sk_O,$$
which is exactly the secret key for $T_a$.

The modified policy becomes $(A\lor T_a)\land(B \lor C)$. After we compile this policy, we receive a single root key (like in a classic BLISK). 

> An important part here is a proof-carrying compilation: when the $(A\lor T_a)$
OR gate is compiled, Alice must prove $T_a$ and $\mathsf{ECDH}(A, T_a)$ were correctly constructed from the agreed oracle parameters $(R_i, P_O, e)$. Without this proof, Alice could substitute a point with a known discrete log and keep her veto forever, while the fallback is silently dead. The mentioned $\Pi.\mathsf{Prove}$ step handles this; it just gets a slightly richer relation.

After the oracle signed an appropriate event, Bob or Carol extracts $sk_{T_a} \gets s_i$ from the oracle's public signature, derives the OR-gate shared secret via $\mathcal{H}(sk_{T_a} \cdot P_A)$, and signs. In other words, the policy collapses to $B \lor C$. The verifier still checks one signature against the same root key; no difference is visible.

Pros:
- If the oracle signs a defined event properly, the adaptor unlocks, and the access structure changes automatically with no participant interaction
- The oracle follows a standard DLC attestation protocol and doesn't even know that some policies depend on it (for the timelock case, imagine the server (it can be MPC or your own time server), which signs a new timestamp every 1000 ms)
- Different gates can reference different oracles and events* (we can apply DLC to the entire circuit, to some subtree, or even to a single user or key)
- The existence of oracle conditions, trigger events, and policy structure remains hidden

> *If we have a condition logic like:
> ```
> if (e == 1) then A
> else if (e == 2) then A + B
> ...
> else if (e == n) then A + B + ... + N
> ```
> it can still be compressed to a single key after BLISK compilation

Limitations and caveats:
- Liveness of the oracle and the need to monitor it
- Irreversibility: once attested, $T_a$ is permanently unlocked
- Each adaptor point is bound to a specific $(R_i, e)$ pair. Nonce reuse breaks security; the usage of a different nonce from the committed one breaks the adaptor

-------------------------

