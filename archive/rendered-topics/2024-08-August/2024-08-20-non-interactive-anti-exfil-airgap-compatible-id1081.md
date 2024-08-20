# Non interactive anti-exfil (airgap compatible)

moonsettler | 2024-08-20 15:15:29 UTC | #1

An anti-exfil variant for airgaped signing devices using PSBTs that works with the QR code based traditional signing workflow.

## Signing protocol:
```
x: private key
X: public key
m: message to sign
n: nonce extra

vc: vector_commit()
```
### 1. signing device
```
q = vc(x, m, n)
Q = q·G
k = q + vc(Q, m, n)
R = k·G
e = hash(R|X|m)
```
Schnorr signature
```
s = k + x·e
```
ECDSA signature
```
r = R.x
s = k⁻¹·(m + r·x)
```
### 2. return to wallet app
```
Q, s
```
### 3. wallet app calculates
```
R = Q + vc(Q, m, n)·G
R, s
```
### 4. verify
Schnorr verify
```
e = hash(R|X|m)

s·G ?= R + e·X
```
ECDSA verify
```
r = R.x

s⁻¹·m·G + s⁻¹·r·X ?= R
```

[https://gist.github.com/moonsettler/d4eb59c62a2b8f104c72603231b73a41](https://Gist, Q&A, earlier discussion)

-------------------------

