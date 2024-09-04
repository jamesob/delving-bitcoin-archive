# Non interactive anti-exfil (airgap compatible)

moonsettler | 2024-08-22 12:18:28 UTC | #1

An anti-exfil variant for airgaped signing devices using PSBTs that works with the QR code based traditional signing workflow. This proposal is anti-exfil protection level 2.

## Anti-exfil protection levels (proposed):
* Level 0: No protection, chosen nonce can immediately leak private key or seed
* Level 1: Deterministic nonce (RFC6979), enables FAT, UAT, no protection from "evil maid" and low probability leaks
* Level 2: Verifiable nonce tweak with external entropy, bandwidth restricted exfil channel, enables FAT, UAT, limited protection from "evil maid" firmware attacks
* Level 3: Negotiated nonce, exfil channel fully plugged, requires multiple rounds of communication

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

moonsettler | 2024-08-21 15:12:12 UTC | #8

[quote="sipa, post:7, topic:1081"]
If there are scenarios where interactive anti-exfil is just not possible, then none of the above are reasons not to proceed with a scheme like this, as it’s probably the best possible. But if people are seriously trying to address the exfiltration scenario, then I wouldn’t just dismiss the possibility that exfiltration is still possible through grinding.
[/quote]

It is certainly possible theoretically. I'm not sure practically it is likely to succeed with QR signers mainly used for cold storage funds. But I can't fully dismiss it either. This algo at least raises the bar significantly.

Can we have a good estimate on how many signatures it would take to leak a 128 or 256 bit seed respectively for the FEC codes?

(About the proof of work thing, I imagine the SD could have a physically wired led indicator showing that the power draw is max/high (like when generating a signature) and also would advise the user, that if a signature takes more than n seconds, then don't proceed with the transaction. Then it would take some benchmarking to make sure 99.999% of the times normal signature generation falls within, but churning nonce points blows it up. It's not perfect, but fairly simple. PoW difficulty would have to change by device and signature type sadly which the companion SW can in theory pass along with the nonce extra.)

-------------------------

sipa | 2024-08-21 15:06:55 UTC | #9

[quote="moonsettler, post:8, topic:1081"]
Can we have a good estimate on how many signatures it would take to leak a 128 or 256 bit seed respectively firth the FEC codes?
[/quote]

They're essentially perfect, information-theoretically speaking. If you can do $2^b$ grinding steps per signature, you can leak $b$ bits per signature. If so, with $128 / b$ signatures you can leak a 128-bit seed.

-------------------------

moonsettler | 2024-08-21 15:08:10 UTC | #10

[quote="sipa, post:7, topic:1081"]
Well the device could have been constructed maliciously in the first place?
[/quote]

Yes. That's a different threat level from just malicious software/firmware, which also opens up the possibility of various other forms of side channel or radio leaks. I'm afraid anti-exfil can't fully cover that scenario.

-------------------------

moonsettler | 2024-08-21 15:33:51 UTC | #11

[quote="sipa, post:7, topic:1081"]
Using that, one can take the secret-to-be-leaked, expand with a several GiB “checksum”, and then in every signature leak a few deterministically-selected (e.g. based on the message and device key that’s only known to the hacker and/or malicious manufacturer) bits of that checksum. Given the size of the checksum, it’s unlikely that any two signatures collide in the position, and as soon as enough bits of the checksum are leaked (regardless of where they are), the attacker can reconstruct the original secret.
[/quote]

Does this assume he knows / can guess which signatures are derived from the same seed? Doesn't this have a combinatoric blowup if the attacker does not have that knowledge (I know the transaction graph is working against us here)?

-------------------------

sipa | 2024-08-21 17:30:10 UTC | #12

[quote="moonsettler, post:11, topic:1081"]
Does this assume he knows / can guess which signatures are derived from the same seed? Doesn’t this have a combinatoric blowup if the attacker does not have that knowledge (I know the transaction graph is working against us here)?
[/quote]

Yes, you need a guess for a subset of transactions carrying the gring signal. I think it's possible to design schemes that permit a smallish subset of incorrect values at the cost of more correct values and more complex decoding algorithm, but if you're talking a huge set of signatures, this won't help you find ones carrying the signal efficiently.

But yes, it's dangerous to assume the attacker cannot infer or at least make good guesses about which transactions might carry the relevant signatures.

-------------------------

moonsettler | 2024-08-22 07:58:37 UTC | #13

*Can we turn this into a quadratic hashing problem for the attacker, by introducing a 4 bit checksum of our own that needs to be ground out?*

Ofc it's not quadratic if it's a constant. I guess that's just hardening with c iterations. Something like 2 seconds target for a single signature should work well, because then attempting to leak more than 1 bit at a time would start to be really noticeable.

Any problems with this approach?


#### Added the following to Q&A section:
Q: Are low bandwidth attacks still possible?

Yes, theoretically the SD with malicious firmware could churn the final R point to leak information using for example FEC codes (Pieter Wuille on delving) with 2<sup>n</sup> rounds of churning SD can leak the seed phrase by creating `bits(seed)/n` signatures. For example if the attacker chooses to leak 4 bits each time it takes 32 signatures to leak a 128 bit (12 word) seed phrase. Due to the public transaction graph and plainly observable wallet characteristics the attacker can make good guesses at which signatures could belong to the same seed.

* The attacker can just generate a random `q`, normally this can not be detected
* The verifier needs to know the private key to verify generation of `q` (same as RFC6979)
* The attacker can encrypt and obfuscate his own channel
* The attacker decides his channel bandwidth making it impossible to estimate "seed health"
* Transactions may have multiple inputs making signing them not deterministic in time
* Said attack is likely to be "always on" and would be caught by verifying generation of `q`
* Evil maid scenario is still problematic, factory and user acceptance tests are already passed
* Tamper evident storage of signing devices is still heavily recommended
* This is not a huge concern for cold storage scenarios with very infrequent signing
* This protocol offers protection from immediate catastrophic leaks via chosen nonce
* 24 words are better in this scheme than 12 words

-------------------------

moonsettler | 2024-09-03 21:28:46 UTC | #14

### Dark Smoothie
I was not sure where to put this, so here we go:

Many have heard about Dark Skippy already, while it is trivial to leak the private key using chosen nonce and it only takes a single signature, Dark Skippy demonstrated the ability to practically leak a 128 bit (12 words) seed using merely 2 signatures from the same device. Here is an alternative (Dark Smoothie), that works if a transaction is signed with 2 inputs locked by the same private key x and can leak 256 bits (24 words) without any fancy math or churning:
```text
k1 = hash(attacker_secret|e1)·x
k2 = 256_bit_secret
z = e1+hash(attacker_secret|e1)
s1 = k1+e1·x = x·z
x = s1·mod_inv(z)
s2 = k2+e2·x
k2 = s2-e2·x
```
`256_bit_secret` leaked on an encrypted channel to the attacker! Worst part is, this works as a low probability attack, that may be triggered by something like for example "dusting" an address with a TXID that conforms to some specific 32 bit checkum. That means such a signing device will statistically always pass factory and user tests expecting RFC6979 noncegen, but can still be triggered to leak the secrets without physical access.

Both attacks are in theory "defeated" by level 2 or 3 Anti-Exfil if the companion app is not malicious.

-------------------------

