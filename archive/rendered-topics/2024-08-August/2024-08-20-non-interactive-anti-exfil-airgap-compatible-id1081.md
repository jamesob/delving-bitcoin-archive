# Non interactive anti-exfil (airgap compatible)

moonsettler | 2024-08-20 18:40:40 UTC | #1

An anti-exfil variant for airgaped signing devices using PSBTs that works with the QR code based traditional signing workflow.

## Signing protocol:
```text
x: private key
X: public key
m: message to sign
n: nonce extra
```
### 1. signing device
```text
q = hash(x, m, n)
Q = q·G
k = q + hash(Q, m, n)
R = k·G
e = hash(R, X, m)
```
Schnorr signature
```text
s = k + x·e
```
ECDSA signature
```text
r = R.x
s = k⁻¹·(m + r·x)
```
### 2. return to wallet app
```text
Q, s
```
### 3. wallet app calculates
```text
R = Q + hash(Q, m, n)·G
R, s
```
### 4. verify
Schnorr verify
```text
e = hash(R, X, m)

s·G ?= R + e·X
```
ECDSA verify
```text
r = R.x

s⁻¹·m·G + s⁻¹·r·X ?= R
```

[https://gist.github.com/moonsettler/d4eb59c62a2b8f104c72603231b73a41](Gist, Q&A, earlier discussion)

-------------------------

reardencode | 2024-08-20 15:26:52 UTC | #2

Some clarifications for readers:
* `n` is a uniformly random value provided by the host wallet app
* `vector_commit` is a cryptographically strong hash of the concatenation of the values - this can be either standard across both bip340 and ecdsa with moon's proposal or match other hashes commonly used in each signing variant (e.g. tagged hash for bip340 and double sha256 for ecdsa)
* this scheme has echoes of the double nonce scheme used in MuSig2 and FROST but because the host has no need to hide its secret value, the secret can be directly sent to the hardware signer. This simplifies the nonce commitment and combination process and allows this nonce combination to be used directly in ECDSA (because the signer knows the combined secret nonce) where in MuSig2-style nonce aggregation the full secret nonce is never known by either party

-------------------------

sipa | 2024-08-20 17:02:38 UTC | #3

Calling it a vector commitment is perhaps a bit confusing, as that term exists in cryptography with a very different [meaning](https://eprint.iacr.org/2011/495.pdf).

I believe this is roughly Scheme 2 from https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-March/017667.html, for ECDSA instead of Schnorr (which is based on an idea [posted](https://bitcointalk.org/index.php?topic=893898.msg9861102#msg9861102) by Greg Maxwell in 2014).

-------------------------

moonsettler | 2024-08-20 18:53:47 UTC | #4

i replaced `vc` with `hash` which should be understood as a cryptographically secure hash commitment to a list of parameters.

```text
[Scheme 2: deterministic nonce, S2C tweak]
* SW generates random t, and requests a signature by sending (Q,m,t) to HW.
* HW computes k0=H(d,m,t), R0=k0G, k=k0+H(R0,t), R=kG,
  s=k+H(R,Q,m)d, and sends (R0,R,s) to SW.
* SW verifies sG=R+H(R,Q,m)Q, R=R0+H(R0,t)G, and publishes sig (R,s) if all
  is good.
```

this is indeed essentially the same! in this case my main question is answered. the scheme is good enough. i think we can proceed with formal specification, and maybe incorporation of support into secp256k1 lib, because people are in fact looking forward to use it.

-------------------------

