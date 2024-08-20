# Non interactive anti-exfil (airgap compatible)

moonsettler | 2024-08-20 15:16:48 UTC | #1

An anti-exfil variant for airgaped signing devices using PSBTs that works with the QR code based traditional signing workflow.

## Signing protocol:
```text
x: private key
X: public key
m: message to sign
n: nonce extra

vc: vector_commit()
```
### 1. signing device
```text
q = vc(x, m, n)
Q = q·G
k = q + vc(Q, m, n)
R = k·G
e = hash(R|X|m)
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
R = Q + vc(Q, m, n)·G
R, s
```
### 4. verify
Schnorr verify
```text
e = hash(R|X|m)

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

