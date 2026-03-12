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

