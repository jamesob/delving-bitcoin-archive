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

