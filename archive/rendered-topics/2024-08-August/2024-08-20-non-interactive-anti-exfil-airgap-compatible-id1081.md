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

harding | 2024-08-21 13:08:17 UTC | #5

I note that Maxwell's original description of this protocol says:

> This leaves open a side-channel that has exponential cost per additional bit, via grinding the resulting R' mod order.  Using the FEC schemes that I've discussed before it's possible to leak a message of any size using even a single bit channel and enough signatures...  But it eliminates the obvious and very powerful attacks where everything is leaked in a single signature.
>
> This is clearly less good, but it's only a two-move protocol, so many places which wouldn't consider using a countermeasure could pick this up for free just as an element of a protocol spec.

-------------------------

moonsettler | 2024-08-21 14:06:59 UTC | #6

This protocol leaves very low bandwith attacks open via churning the final nonce point. It's pretty expensive for a low power device to keep churning double point multiplications along with hashes; the device also needs to remember which parts of the seed it had leaked, which might not even be possible with only a firmware modification; or do some message based pseudo random indexing scheme which further increases the difficulty.

And a single factory or user validation test that checks the generation of Q is up to spec would catch it immediately (similar to RFC6979). With low bandwith leaks the low probability random attack is not likely to be viable, and the risk of getting caught on a routine test is very high if the attack is "always on".

(there is ongoing discussion about adding additional proof of work to further decrease the bandwith or make it impractical and/or the time to generate a signature along with the power draw could be used to detect nonce churning where the hw capabilities are known like firmware attacks)

-------------------------

sipa | 2024-08-21 14:32:38 UTC | #7

[quote="moonsettler, post:6, topic:1081"]
It’s pretty expensive for a low power device to keep churning double point multiplications along with hashes
[/quote]

It's certainly expensive, and the cost grows exponentially with the number of bits, but I don't think say a 4-bit grind is beyond the power of small devices, which would suffice to leak a secret with 64 signatures.

[quote="moonsettler, post:6, topic:1081"]
which might not even be possible with only a firmware modification
[/quote]

Well the device could have been constructed maliciously in the first place?

[quote="moonsettler, post:6, topic:1081"]
the device also needs to remember which parts of the seed it had leaked
[/quote]

Using FEC codes that's not necessary. Using that, one can take the secret-to-be-leaked, expand with a several GiB "checksum", and then in every signature leak a few deterministically-selected (e.g. based on the message and device key that's only known to the hacker and/or malicious manufacturer) bits of that checksum. Given the size of the checksum, it's unlikely that any two signatures collide in the position, and as soon as enough bits of the checksum are leaked (regardless of where they are), the attacker can reconstruct the original secret.

(Note that this several GiB checksum isn't actually ever materialized; arbitrary bits of it can be computed on-the-fly).

[quote="moonsettler, post:6, topic:1081"]
And a single factory or user validation test that checks the generation of Q is up to spec would catch it immediately (similar to RFC6979). With low bandwith leaks the low probability random attack is not likely to be viable, and the risk of getting caught on a routine test is very high if the attack is “always on”.
[/quote]

The device could choose to only behave maliciously in certain situations, such as when interacting with large amounts.

If there are scenarios where interactive anti-exfil is just not possible, then none of the above are reasons not to proceed with a scheme like this, as it's probably the best possible. But if people are seriously trying to address the exfiltration scenario, then I wouldn't just dismiss the possibility that exfiltration is still possible through grinding.

-------------------------

moonsettler | 2024-08-21 15:02:28 UTC | #8

[quote="sipa, post:7, topic:1081"]
If there are scenarios where interactive anti-exfil is just not possible, then none of the above are reasons not to proceed with a scheme like this, as it’s probably the best possible. But if people are seriously trying to address the exfiltration scenario, then I wouldn’t just dismiss the possibility that exfiltration is still possible through grinding.
[/quote]

It is certainly possible theoretically. I'm not sure practically it is likely to succeed with QR signers mainly used for cold storage funds. But I can't fully dismiss it either. This algo at least raises the bar significantly.

Can we have a good estimate on how many signatures it would take to leak a 128 or 256 bit seed respectively firth the FEC codes?

(About the proof of work thing, I imagine the SD could have a physically wired led indicator showing that the power draw is max/high (like when generating a signature) and also would advise the user, that if a signature takes more than n seconds, then don't proceed with the transaction. Then it would take some benchmarking to make sure 99.999% of the times normal signature generation falls within, but churning nonce points blows it up. It's not perfect, but fairly simple. PoW difficulty would have to change by device and signature type sadly which the companion SW can in theory pass along with the nonce extra.)

-------------------------

