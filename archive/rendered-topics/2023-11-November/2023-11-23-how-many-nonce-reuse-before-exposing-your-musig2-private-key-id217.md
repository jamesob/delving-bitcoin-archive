# How many nonce reuse before exposing your Musig2 private key?

t-bast | 2023-11-23 15:02:56 UTC | #1

The [Musig2 BIP](https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki) says that signing two distinct messages with the same `secnonce` exposes your private key. While I know we must absolutely not reuse `secnonces`, I wanted to check how the private key extraction worked and the difference with a standard schnorr sig.

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

