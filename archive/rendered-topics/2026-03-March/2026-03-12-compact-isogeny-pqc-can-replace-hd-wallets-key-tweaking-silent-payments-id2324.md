# Compact Isogeny PQC can replace HD wallets, key-tweaking, silent payments

conduition | 2026-03-12 18:33:29 UTC | #1

Other posts here have explored some of the bleeding-edge of lattice-based crypto: https://delvingbitcoin.org/t/post-quantum-hd-wallets-silent-payments-key-aggregation-and-threshold-signatures/1854

...But even at their best, lattice crypto schemes require several kilobytes of key+sig. Isogeny crypto is much more compact at less than 300 bytes of key+sig, does not accumulate errors, and can be used to do much of the same stuff.

https://conduition.io/cryptography/isogenies-intro/

I published this article just yesterday after spending the last few months researching isogenies. I'm still learning, so there may be some inaccuracies, it's a difficult subject. I gave my best shot at making it approachable to bitcoin engineers who are already familiar with classical ECC.

Here's a short version to give you a taste.

- Isogenies are functions that map points between different elliptic curves.
- _Endomorphisms_ are isogenies that map points on a curve to itself, e.g. scalar multiplication is an endomorphism.
- Given two curves $E_1$ and $E_2$, we believe it is a hard problem to find an isogeny between them. This is called the "supersingular isogeny path problem" (SIPP).
- For every elliptic curve $E$ there exists a mathematical object called the "endomorphism ring" of $E$, denoted $\text{End}(E)$, which is the ring of all endomorphisms on $E$.
- Finding $\text{End}(E)$ given only $E$ is a hard problem, called the "endomorphism ring problem" (ERP).
- Some supersingular elliptic curves have well-known and friendly endomorphism rings. One such curve is $E_0: y^2 = x^3 + x$
- The SIPP has been proven equivalent to the ERP. Namely:
    - Given curves $E_1$ and $E_2$ and their endomorphism rings $\text{End}(E_1)$, $\text{End}(E_2)$, we can efficiently compute an isogeny $\varphi : E_1 \rightarrow E_2$ (or $E_2 \rightarrow E_1$).
    - Given curves $E_1$ and $E_2$, one endomorphism ring $\text{End}(E_1)$ and an isogeny $\varphi : E_1 \rightarrow E_2$, we can efficiently compute the endomorphism ring $\text{End}(E_2)$.

For cryptography, we typically generate a secret isogeny $\varphi : E_0 \rightarrow E$, and use our knowledge of $\text{End}(E_0)$ to compute $\text{End}(E)$. Our secret key is either $\varphi$ or equivalently $\text{End}(E)$. 

Now we have some new powers which nobody else in the universe, hopefully even quantum-enabled attackers, can do:

- if given an isogeny $\phi_1: E \rightarrow E'$, we can efficiently compute an isogeny $\psi_1: E_0 \rightarrow E'$
- if given an isogeny $\phi_2: E_0 \rightarrow E'$, we can efficiently compute an isogeny $\psi_2: E \rightarrow E'$

We can use these abilities to instantiate signature schemes - see [this section of the article](https://conduition.io/cryptography/isogenies-intro/#Naive-Unsafe-Signing). But mainly i want to show you how, given these simple rules, we get unhardened BIP32 key derivation. 

All we need to do is generate a pseudorandom isogeny $\psi: E \rightarrow E'$ which maps the public key to some new uniformly-random curve $E'$. This has been done before by existing schemes in a different context (e.g. producing challenge isogenies in SQIsign). The keyholder can recompute $\psi$ and then use their knowledge of $\text{End}(E)$ to find $\text{End}(E')$, and voila, they've updated their secret key. If needed the keyholder can use the publicly-known constant $\text{End}(E_0)$ to find an updated secret isogeny $\varphi' : E_0 \rightarrow E'$. 

<img src="upload://85Y7pw2t65y5TB6iq9vpgM4L9Kr.svg">

This same approach can be reused to recreate silent payments and taproot-style key tweaking (commitment inside a pubkey).

Multisignature isogeny schemes that can compete with SQIsign and PRISM for compactness are unfortunately still lacking, but I hope that by drawing attention to this topic, I can convince more bitcoin devs to start researching this subject and contribute to it the same way we've contributed to multisignature schemes in the domain of classical ECC.

-------------------------

AdamISZ | 2026-03-18 19:36:26 UTC | #2

Thanks for the effort in writing this article! I agree it’s great to introduce people (including me) to some newer ideas that might actually end up being important down the line.

I found it interesting in reading your discursive lead-through from “naive” to “SQI-sign” that you are showing an instantiation of what looks to me like the general Schnorr-like protocol. Having said that, there are obviously going to be differences.

But to explain what I'm even talking about: the general meta-argument goes something like: given **any** one-way mapping with a "homomorphism" between "groups", one can construct a zero knowledge proof of knowledge.

(scare quotes because the pattern is even more general than that).

Back in ECDL land, you’d write the reasoning as: to build a zkpok (which a signature scheme is just a trivial extension of by appending a message, basically), start with an instance of the “language”, that is: a public key $P$ and its corresponding private key $x$. The map from private to public is what must have the aforementioned two properties: one way (computationally), and also possessing the homomorphism (i.e. that $x_1 + x_2$ has pubkey $P_1 + P_2$), take the secret (privkey) and then bind it to a challenge value which hashes-in the public key and the message. i.e. just respond with privkey $\times$ challenge.

Except, whoops, that only binds, it does not hide (so no ZK). Hiding is trivially removed by division, because the challenge is public.

To create the ZK effect, create *another instance of the language* just for one time use: in ECDL land we do $R = kG$. Blind the response with that by applying the homomorphism again: the privkey of $R + $ challenge $\times P$ is ($k$ + challenge $\times x$).

Once you grok the meta-pattern you can apply it to anything more (even,  much more) complicated, in ECDL land. A good example: Pedersen commitments take the form $C = xG + rH$. They are one way and they have a homomorphism (check what happens when you add two such commitments $C_1, C_2$), so you can build a commit-challenge-response protocol and get a signature out of them. your "other instance of the language" here is no longer $R = kG$ but an $R$ which is itself a Pedersen commitment. Etc.

The argument that such "zero knowledge proofs of knowledge" are sound, i.e. can't be forged, is that with rewinding, you can actually extract the secret by replaying with the same "commitment" (the $R$ part). But notice that that requires inverses in the field ($x = \frac{(s_2 - s_1)}{(e_2 - e_1)}$).

Is it all "the same" with isogenies? Presumably not :grinning_face_with_smiling_eyes:. But I'm curious how much maps over, since obviously I don't know much of anything about this field:

Here:

* the one-way function is from isogeny $\phi$ to a curve $E_{pk}$
* the "homomorphism" is not actually a homomorphism, but it's some kind of "composable map". It's function (isogeny) composition, right? Which isn't commutative for example (i.e. it's nothing like doing a $\circ$ b in an abelian group). But if I understood your post correctly, there is a thing analogous to inverses: the "dual" you mention, $\widehat{\phi}$. I just asked an LLM if I'm right that they're analogous and it said something like no, not really, the right analog is "invertibility of ideals in the quaternion algebra", so I stopped there for now :grinning_face_with_smiling_eyes:
* Then the solution for ZK is, as you refer to in the post, exactly the same as our familiar "nonce": it's the commit step in commit-challenge-response $\Sigma$-protocols. There's a "nonce" isogeny $\phi_{Comm}$ and its corresponding curve $E_{Comm}$.

Are the proofs of soundness and zk-ness similar to the Schnorr ones? Do we use rewinding for the first, and transcript simulation for the latter?

But back to your article, which again, I very much appreciate. I have a strong gut instinct to agree with you that we are not yet paying sufficient attention to whether the primitives we end up using for a PQ algo have "nice" properties beside being at all workable for generating and verifying signatures. You focused first on the 'rerandomization' property, very natural. I tend to think 'aggregatable' and 'batch verifiable' might be the most important ones. (and iiuc you're saying that unfortunately aggregation is not easy, here). Also the NUMS thing is very unfortunate but yeah we could maybe accept it if everything else is good.

Another concern might be, if the rest of industry ends up focused on hash based signatures because they have a different priority set than Bitcoin, then we might be stuck with too limited practical support and experience. With EC we could rely on deployment that already existed by 2009.

-------------------------

conduition | 2026-03-20 19:19:24 UTC | #3

[quote="AdamISZ, post:2, topic:2324"]
* the one-way function is from isogeny $\phi$ to a curve $E_{pk}$
[/quote]

Kind of, though there are caveats. In more rigor, it'd be better to say the one-way function we rely on is the mapping of an isogeny $\varphi : E_0 \rightarrow E$ to its codomain curve $E$. The base curve $E_0$ is quite important because $\text{End}(E_0)$ is a known constant. You can't use any old isogeny as a secret key, because you need to be able to compute (and hide) the endomorphism ring $\text{End}(E)$. 

[quote="AdamISZ, post:2, topic:2324"]
the “homomorphism” is not actually a homomorphism, but it’s some kind of “composable map”. It’s function (isogeny) composition, right?
[/quote]

In a way, you're on the money here. For two isogenies $\varphi: E_0 \rightarrow E_1$ and $\psi: E_1 \rightarrow E_2$ you can compose them into $\psi \circ \varphi: E_0 \rightarrow E_2$. The resulting composed isogeny will have degree $\deg \varphi \cdot \deg \psi$, so it's much more complex to represent, but it does exist, and you can condense it to a lower-degree isogeny with some work.

However, there is no way (AFAIK) to "compose" elliptic curves like you can add points in classical ECC, so the analogy falls apart there.

<details>
<summary><b>Aside on homomorphisms</b></summary>

Interestingly, isogenies are actually _group homomorphisms:_ for an isogeny $\varphi: E \rightarrow E'$ we always have $\varphi(P) + \varphi(Q) = \varphi(P + Q)$. However, in my limited experience this doesn't actually come up much in practice and it's more of a structural requirement, a necessary property for any isogeny.
</details>

[quote="AdamISZ, post:2, topic:2324"]
Which isn’t commutative for example (i.e. it’s nothing like doing a \circ b in an abelian group).
[/quote]

Correct. In my above example, you could not compose $\varphi \circ \psi$, because what does it even mean to evaluate $\varphi$ on a point in $E_2$? It's undefined.

CSIDH gets around this: It implements a commutative group action where an _oriented_ isogeny can produce some _commutative action_ on any elliptic curve, mapping it to another one. From this, non-interactive DH key exchange kinda falls into our lap. 

SQIsign and PRISM work in a more general way, with no orientations. Their performance is much faster and more compact as a result, but they are way less flexible than CSIDH. This is probably why all existing isogeny multisig schemes I know of are built on top of CSIDH. 

<details>
<summary><b>Aside about SIDH</b></summary>

SIDH used unoriented isogenies for key-exchange by abusing a neat fact: If you have isogenies $\varphi: E_0 \rightarrow E_a$ and $\psi: E_0 \rightarrow E_b$, with kernels $\ker \varphi$ and $\ker \psi$ then you can compute something called a _pushforward_ isogeny $\varphi * \psi$. This is the isogeny you get if you (1) take $\ker \psi$ and "push" it through $\varphi$, and then (2) convert the resulting image $\varphi(\ker \psi) \subset E_1$ into a new isogeny which has that image as its kernel (e.g. by using Velu's formulas). More precisely, $\ker(\varphi * \psi) = \{ \varphi(P) : P \in E_0 \text{ s.t. } \psi(P) = \infty \}$.

Seems weird to do this, but after computing the converse pushforward $\psi * \varphi$ as well, it just so happens that the (output curve) codomain $E_{ab}$ of $\psi * \varphi$ is the same as that of $\varphi * \psi$. This is how SIDH achieved key agreement, but now we know because of Kani's lemma that way we _compute_ the pushforward always ends up revealing the isogenies involved, so it's not a secure key exchange.
</details>

[quote="AdamISZ, post:2, topic:2324"]
But if I understood your post correctly, there is a thing analogous to inverses: the “dual” you mention, $\widehat{\phi}$.
[/quote]

Very apt analogy, yes. Given any isogeny $\varphi: E \rightarrow E'$, it's easy to compute its dual $\widehat \varphi: E' \rightarrow E$. The dual is not exactly an inverse, because for complicated math reasons, $\widehat \varphi(\varphi(P)) = P \cdot \deg \varphi$. But for the purposes of solving the isogeny path problem and finding endo-rings, that doesn't actually matter much. 

Since isogenies all have an obvious and unique dual, they turn the set of all supersingular elliptic curves into vertices on a graph, where each edge is an isogeny. A good chunk of the theory work on IBC is done in terms of "the supersingular isogeny graph", because we can reason about the security properties of schemes using graph theory, which is pretty cool. 

[quote="AdamISZ, post:2, topic:2324"]
Are the proofs of soundness and zk-ness similar to the Schnorr ones? Do we use rewinding for the first, and transcript simulation for the latter?
[/quote]

They are indeed quite similar. BIP340 Schnorr and SQIsign are both a fiat-shamir transform of some sigma protocol, so in both cases we can prove security by proving (1) correctness, (2) special soundness, and (3) honest-verifier zero-knowledge, which the SQIsign authors have done using special-degree isogeny oracles. PRISM is different since it doesn't have any commitment, so its security proof is less tight (though still plausibly valid if their assumptions are correct).

[quote="AdamISZ, post:2, topic:2324"]
But back to your article, which again, I very much appreciate. I have a strong gut instinct to agree with you that we are not yet paying sufficient attention to whether the primitives we end up using for a PQ algo have “nice” properties beside being at all workable for generating and verifying signatures. You focused first on the ‘rerandomization’ property, very natural. I tend to think ‘aggregatable’ and ‘batch verifiable’ might be the most important ones. (and iiuc you’re saying that unfortunately aggregation is not easy, here)
[/quote]

Thank you! I focused mostly on rerandomization because it was an easy consequence of the structure of IBC, and surprisingly few other researchers seem to have noticed it, and none have thought of applying it to bitcoin.

I don't know whether signature aggregation or batch verification are possible or easy in IBC, i certainly haven't heard of any concrete schemes. But this is exactly why i wrote the article: aggregation and batch verification are things the wider cryptographic world often doesn't care so much about - They are solutions to problems which only blockchains tend to encounter. Most of the rest of the world only cares if signatures and pubkeys fit neatly in a TLS certificate and don't take too long to verify. If we want these features, we'll need to apply ourselves to the task. If we do, i'm sure we can discover analogues to MuSig, DahLIAS, or other cool things, though they will probably have steep tradeoffs at first.

I suspect if it is possible to achieve signature aggregation or batch verification on PRISM or SQIsign, it would be by a recursive application of Kani's lemma on some carefully built (very convoluted) set of high-dimensional (HD) isogenies. In general, HD isogenies have this neat property of allowing you to "aggregate" isogenies, though with very strict rules. IDK if we could do that yet, still learning. Take a look at the variant [SQIsignHD](https://eprint.iacr.org/2023/436) to see how they used this technique to compress sigs down to 109 bytes, at the cost of verifier performance.

-------------------------

AdamISZ | 2026-03-21 14:54:33 UTC | #4

[quote="conduition, post:3, topic:2324"]
Interestingly, isogenies are actually *group homomorphisms:* for an isogeny \varphi: E \rightarrow E' we always have \varphi(P) + \varphi(Q) = \varphi(P + Q). However, in my limited experience this doesn’t actually come up much in practice and it’s more of a structural requirement, a necessary property for any isogeny.
[/quote]

I'm confused about this one in particular: I must have some mischaracterization in my head, because I thought this is the basic property we need for a zkpok: we need a one way function with a homomorphism, so the latter part here is exactly the formula you wrote above?

Another question from the part below that: we talked about duals being inverses (or rather, not) and we talked about soundness proofs: but the most specific reason I thought about inverses here is because you need inverses to make a soundness proof go through (specifically special soundness or even more specifically 2-special-soundness, here).

OK, I took the time to look at [this](https://arxiv.org/abs/2603.09899) doc, and I can see it's exactly there, on p 19, as I'd expect: the soundness proof uses duals "just like" (kinda, heh) you use an inverse in a soundness proof in DL based crypto.

One last comment or question, though it may not be something that has any specific answer. In your article you address the core question of PQ-ness: "As far as we know, there is no efficient quantum algorithm to solve this problem efficiently." What's the sort of general thinking behind the confidence that this will remain hard? For hash-based stuff it's "hashes break all the algebraic structure" right. We still get Grover, but that's not enough of an "exploit" to break the fundamental exponential hardness, it just "softens" it. So for this isogeny stuff, is it that, we're not using simple group properties like discrete log, which get exposed to cycle finding and whatnot, but instead we're doing something more like graph search?

-------------------------

