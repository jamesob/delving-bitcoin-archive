# Bird of Prey 2: non-malleable Schnorr + PQ signatures

sipa | 2026-05-22 21:44:03 UTC | #1

This [paper](https://link.springer.com/chapter/10.1007/978-3-032-25317-0_8) just presented at EuroCrypt 2026 may be interesting to some. This is isn't a proposal, or even suggestion, for use in Bitcoin. However, given the amount of interest in the discussion of PQC signature schemes, I thought it would be cool for people to be aware.

The question explored is how to construct hybrid SUF-CMA (called "non-malleable signatures" in Bitcoinese) signature schemes, if you have two individually SUF-CMA signature schemes. Specifically, one Schnorr-like one, and another possibly PQC one.

The naive solution is just to sign with both schemes once, and concatenate the signatures. That is unforgeable if at least one of the schemes remains unbroken, but not non-malleable. This is easy to see: if one of the schemes is broken, an attacker can learn the corresponding private key, and replace the signature of that scheme with a new one. They cannot sign a message of their choice as long as the other scheme remains secure, but they succeeded in malleating.

The `BoP-2` scheme in the paper offers a solution specifically for combining (a) a signature scheme based on a Fiat-Shamir transformed identification scheme like Schnorr and (b) an arbitrary other scheme that is used as a black box. If I understand it correctly (I only saw the presentation, and skimmed the paper), it would translate to something like this if applied to BIP340 + some arbitrary PQC scheme:

To sign message $m$ using private key $(d_\text{schnorr}, d_\text{pqc})$ and public key $P = (P_\text{schnorr}, P_\text{pqc})$:
* Generate random nonce $k$, and $R = k \cdot G$ (same as BIP340).
* Compute $e = \text{H}(R \,||\, P \,||\, m)$.
* Create signature $\sigma_\text{pqc}$ with key $d_\text{pqc}$ on message $e$.
* Let $s = k + d_\text{schnorr} \times \text{H}(\sigma_\text{pqc})$.
* Return $\sigma = (s, \sigma_\text{pqc})$. 

To verify signature $(s, \sigma_\text{pqc})$ on message $m$ against public key $P = (P_\text{schnorr}, P_\text{pqc})$:
* Recover $R = s \cdot G - \text{H}(\sigma_\text{pqc})\cdot P_\text{schnorr}$.
* Recompute $e = \text{H}(R \,||\, P \,||\, m)$.
* Verify signature $\sigma_\text{pqc}$ with key $P_\text{pqc}$ on message $e$.

Note that through the need to recover $R$ from $s$, this is not batch-verifiable, but saves 32 bytes. I believe it's easy to construct an alternative that just puts $R$ in the signature, but batch-verifiability may also just be an unmaintainable property in a PQC setting. For security, this also relies on $\sigma_\text{pqc}$ being an informational-theoretical commitment to $e$ (and thus not reliant on the computational hardness of the signature scheme).

-------------------------

AdamISZ | 2026-05-23 14:05:08 UTC | #2

Thanks for flagging that.

Seems like a pretty important paper.

The way I'm getting the gist here is: 'default' stuff out there, in industry, and NIST and whatnot, is doing hybrid by concat. And this makes sense when you care about EUF and not SUF. But some things require SUF and Bitcoin is one of them.

So "nesting" exists, but it's hard to get it to have the properties you want: if you *just* sign one scheme with the other, you don't get SUF from the fact that the  *outer* scheme is SUF. (To concretize: we're imagining: outer scheme is bip340 and inner scheme is something PQ).

In the paper they try to argue (probably right, I have no idea) that this problem can't really be avoided if you just treat the two sig schemes as black-box.

So the trick (in BOP-2 which is one of three different constructions here) is to *not* treat Schnorr as black-box. They treat it as an ID scheme (which it is) and make the challenge in such a way that the SUF property is inherited, even if the PQ scheme doesn't have it.

Do I have that roughly right?

About batch-verify, a couple of questions:

How crucial is batch verification, today, in Bitcoin?

For your point that it might just be not possible in the PQ regime: what about isogenies, I wonder. I guess hash-based is a non-starter, and lattices .. well there's some linearity there right? But from what I'm reading, no dice. And isogenies even if sound might not be performant enough for this stuff. Shrug, no idea :slight_smile: 

About including R or not: I think if I'm reading this right, we're publishing $\sigma_{sch}$ and $\sigma_{pq}$ so since the latter is the dominant element, does it matter?

-------------------------

sipa | 2026-05-23 14:57:01 UTC | #3

[quote="AdamISZ, post:2, topic:2514"]
Do I have that roughly right?
[/quote]

That sounds exactly right.

[quote="AdamISZ, post:2, topic:2514"]
For your point that it might just be not possible in the PQ regime: what about isogenies, I wonder. I guess hash-based is a non-starter, and lattices .. well there’s some linearity there right? But from what I’m reading, no dice. And isogenies even if sound might not be performant enough for this stuff. Shrug, no idea :slight_smile:
[/quote]

I hadn't considered the possibility of batch-verification of the PQC signature scheme, only about whether it's worth retaining Schnorr batch verifiability. If the PQC signature scheme is significantly more expensive per signature (not necessarily per byte) than Schnorr, really only the batch verifiability of the former matters. As for how feasible it is with various PQC classes, no idea.

[quote="AdamISZ, post:2, topic:2514"]
About including R or not: I think if I’m reading this right, we’re publishing \sigma_{sch} and \sigma_{pq} so since the latter is the dominant element, does it matter?
[/quote]

Possibly not.

-------------------------

