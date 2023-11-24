# How many nonce reuse before exposing your Musig2 private key?

t-bast | 2023-11-23 15:08:03 UTC | #1

The [Musig2 BIP](https://github.com/bitcoin/bips/blob/e918b50731397872ad2922a1b08a5a4cd1d6d546/bip-0327.mediawiki) says that signing two distinct messages with the same `secnonce` exposes your private key. While I know we must absolutely not reuse `secnonces`, I wanted to check how the private key extraction worked and the difference with a standard schnorr sig.

And I couldn't figure it out! It seems to me that an attacker would need **three** partial sigs for distinct messages using the same `secnonce`, **two** doesn't seem to be enough?

Here is a simplified write-up of the equations, with a slightly different naming than the BIP to focus only on the important parts:

```
Alice has public key Pa = ka*G
Bob has public key Pb = kb*G
Q is the aggregated public key
R is the public aggregated nonce

b = H("noncecoef" || R || Q || m)
e = H("challenge" || R || Q || m)

=> implies that an attacker cannot choose (rb1', rb2') to influence b' thanks to H's preimage resistance

First signature for message m1:

  sa1 = ra1 + b1*ra2 + e1*a*ka
  sb1 = rb1 + b1*rb2 + e1*a*kb

Second signature for message m2 where Alice reuses ra1 and ra2:

  sa2 = ra1 + b2*ra2 + e2*a*ka
  sb2 = rb1' + b2*rb2' + e2*a*kb

The attacker ends up with:

  sa2 - sa1 = (b2 - b1)*ra2 + a*(e2 - e1)*ka
```

The attacker ends up with one equation but two unknowns (`ra2` and `ka`), so it's not sufficient to extract `ka`? It needs a third signature to obtain a second equation and solve for both `ra2` and `ka`? What am I missing?

-------------------------

ajtowns | 2023-11-24 01:33:32 UTC | #2

I don't think you're missing anything; though perhaps some wagner-ish approach could also cause you to reveal your secret key if you sign many times while only using any single nonce pair twice. With musig2 you effectively have two nonces ($ra_1, ra_2$), so giving you three unknowns ($ra_1, ra_2, ka$) so needing three equations (signatures) to solve for three unknowns seems sensible.

-------------------------

conduition | 2023-11-24 07:12:11 UTC | #3

Neat point, i hadn't thought of this till now. 

I believe you are correct _for musig2._ But the older MuSig1 has no nonce coefficient $b$, and so reusing a nonce even once will expose your signing key the same way that reusing a Schnorr signing nonce will.

If you have three partial signatures under the same nonce, it is possible to extract the secret key, although the algebra involved is a bit overzealous. 

Given:
- $s_1 = r_1 + r_2 b_1 + e_1 k a$
- $s_2 = r_1 + r_2 b_2 + e_2 k a$
- $s_3 = r_1 + r_2 b_3 + e_3 k a$

You can solve for $k$ as:

$k = \frac{(s_3-s_1)(b_2-b_1) - (s_2-s_1)(b_3-b_1)}{a \left( (e_3-e_1)(b_2-b_1) + (e_1-e_2)(b_3-b_1) \right)} $

I wrote a test to sanity check my math and indeed it does work: https://github.com/conduition/musig2/commit/778cc61d466d9cb418414bd2ff7692d2d6dba34d

Also worth noting: Reusing nonces in musig is bad even if you are signing _the same message,_ because other signers can change their nonces and thus change the aggregated nonce, which changes $e$ the same way that changing the message would.

-------------------------

