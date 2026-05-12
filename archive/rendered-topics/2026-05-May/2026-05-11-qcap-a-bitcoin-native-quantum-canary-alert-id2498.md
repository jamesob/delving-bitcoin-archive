# QCAP: A Bitcoin-Native Quantum Canary Alert

qatkk | 2026-05-11 16:44:38 UTC | #1

Bitcoin’s security depends on the computational hardness of the discrete logarithm problem over elliptic curves (ECDLP). A sufficiently powerful cryptographic relevant quantum computer could easily solve the ECDLP, thus undermining the security assumptions underlying Bitcoin’s current digital signature schemes. Yet the timeline for such breakthrough remains fundamentally uncertain and estimations range from decades to “sooner than we think,” but there is no clear consensus within the field.

Moreover, the core challenge is not only estimating the timeline for such a breakthrough, but the absence of reliable and objective signals of progress. On one hand, markets reward hype, so companies are in that game and governments have incentives to hide their advancements. On the other hand, the technical community has no unified way to measure quantum progress against cryptographic hardness.

The Quantum Canary Address Generation Protocol (QCAP) proposes a practical answer: a Bitcoin taproot address that can only be spent by someone who can solve the discrete logarithm problem on a weaker elliptic curve. The protocol is decentralized, requires no trusted authority, and provides verifiable proofs of correct construction.

QCAP is executed by $n$ participants, where any one of them may act as the coordinator. Each participant contributes a secret value $d_i$, such that $\hat{d} = \sum_{i=1}^{n} d_i$ remains unknown as long as there exist one honest participant. The coordinator’s role is limited to verifying proofs, aggregating keys, and publishing the canary address.

## The Core Idea: Two Curves, One Secret

The core of QCAP lies in a simple observation: we want to publish a challenge on the Bitcoin blockchain that is easier than breaking secp256k1, while remaining natively compatible with Bitcoin’s existing protocol. In practice, this means encoding it into a standard Bitcoin address. The naive approach, publishing a point on a weaker curve like secp192r1, fails because a quantum computer solving it learns nothing about Bitcoin’s curve. As a result, there is no cryptographic link to a standard Bitcoin address. Our approach instead uses the same secret scalar on both curves with the following structure:

* Bitcoin curve (secp256k1): Publish public key $P = \hat{d} \cdot G$ where $\hat{d}$ is a distributedly computed secret scalar and $G$ is the generator of Bitcoin’s curve.

* Weaker curve (secp192r1): Publish public key $P’ = \hat{d} \cdot G’$ where $G’$ is the generator of the weaker curve.

If $\hat{d}$ is small enough to fit in the scalar field of secp192r1 (which has order $n_{192}$), then solving ECDLP on the weaker curve reveals $\hat{d}$, which can immediately be used to spend from the Bitcoin address. The main challenge is designing the key-generation process such that no participant gains a computational or informational advantage over others, while ensuring that the protocol remains secure as long as at least one participant is honest.

## Multi-Party Computation of the Shared Secret

QCAP generates $\hat{d}$ using inputs from $n$ different participants:

$$
\hat{d} = d_1 + d_2 + \cdots + d_n \pmod{n_{192}}
$$

Each participant, $i$, independently:

* Chooses secret $d_i $ from the weaker curve’s field.
* Computes its Bitcoin public key: $P_i = d_i \cdot G$.
* Computes its weaker curve public key: $P’_i = d_i \cdot G’$.
* Publishes $(P_i, P’_i)$.

The protocol’s public keys are then derived by aggregating participants’ public keys:

$$
P = \sum_{i=1}^n P_i = \hat{d} \cdot G
$$

$$
P' = \sum_{i=1}^n P'_i = \hat{d} \cdot G'
$$

As long as a single participant is honest and does not leak his $d_i$, nobody, not even the coordinator, learns $\hat{d}$. Even though no single party can obtain the key, a malicious participant may still introduce inconsistencies between the public key on the weaker curve and Bitcoin’s curve. It is therefore necessary to ensure that $P$ and $P’$ really do share the same secret scalar.

## Proving Consistency Across Curves: DLEQAG

The vulnerability: a malicious participant could choose one $d_i$ for $P_i$ and a different $d’_i$ for $P’_i$. Then $P’$ would be derived from $\hat{d}’ = \sum d’_i \neq \hat{d}$, and solving ECDLP on the weaker curve would not reveal the correct secret needed to spend the corresponding Bitcoin output.

QCAP prevents this problem using **Discrete Logarithm Equality Across Groups (DLEQAG)**, a zero-knowledge proof technique from [Chase et al. (2022)](https://eprint.iacr.org/2022/1593.pdf). This proof basically demonstrates that the scalar corresponding to two points (i.e. public keys) on two different curves is equal (without revealing the scalar).

The protocol only aggregates public keys with valid proofs. This prevents any participant from introducing mismatched secrets.

## Bitcoin Integration: Taproot and OP_RETURN

Once all participants’ proofs have been verified and public keys have been aggregated into $P$, the coordinator creates a **taproot address** from a tweaked public key:

$$
P_t = \text{tweak}(P, D)
$$

where $D$ is a hash commitment to all the protocol data (all participants’ proofs, public keys, and public information).

This serves two purposes:

* Commits the protocol data to the address itself: anyone can verify the derivation was honest.
* Enables spending with knowledge of the tweak: a quantum computer that solves the weaker ECDLP and finds $\hat{d}$ can adjust it to account for the tweak and spend the output.

The funding transaction includes an `OP_RETURN` output linking to all protocol data stored on IPFS:

```
OP_RETURN <CID of proofs and public keys>
```

Anyone can:

* Download the IPFS file using the CID.
* Verify the cryptographic proofs.
* Confirm the final address was generated honestly.
* Check the Bitcoin bounty.

## Implementation Notes

This approach is formalized in the research paper [“QCAP: A Quantum Canary Address Generation Protocol”](https://eprint.iacr.org/2026/618.pdf), authored by me and my coauthor @nebula-21, under the supervision of our advisors at [DISCRYPT](https://www.discrypt.cat) research group. A proof of concept [implementation](https://github.com/QCAP-org/QCAP) is also available, demonstrating that the core mechanics function as intended. For those interested in the deeper technical details or implementation specifics, both are available—but they’re worth reading after understanding the core idea from this post.

A sample of the generated address and corresponding proofs can also be found on testnet as follows:

Transaction on testnet: `ebfcee0b5cd1a54f7079dd058d27a6d87316adc42438c6fc05ca64b0234a628c`

IPFS link: `https://ipfs.io/ipfs/bafybeid356uqgs7mvhotv7mcgfabbk7w7fglg3gctjabcsh47dym7gu2aa`

The implementation demonstrates feasibility but isn’t production-ready. There are still several areas for hardening:

* Decentralized participant coordination (currently out-of-band).
* Robust network handling for malicious coordinators.
* Engagement of community members in the QCAP execution.
* Mainnet deployment and community fundraising.

-------------------------

AdamISZ | 2026-05-11 19:32:01 UTC | #2

Looks like a pretty solid idea to me, at least, in theory![1].

On the 2022 DLEQAG paper, I noticed when I was researching something similar recently, an [MRL note with an algo attributed to Poelstra](https://www.getmonero.org/resources/research-lab/pubs/MRL-0010.pdf)

The Chase et al Paper focuses on a case of bit length of the secret significantly smaller than the smaller of the two groups, and then offers ways to extend it.

Poelstra's sort of vanilla-sigma-protocol approach won't be super efficient, but that is not needed here afaict. While it should work for any value in the order of the smaller group.

(Correct me if some part of that is wrong!)

Another part: you didn't mention how the multiparty secret generation algorithm will work? I guess there's a range of slightly different approaches but the template of DKG as in FROST seems to fit well, albeit there is no thresholding here; the point is to have a PoK of the contribution[2], mainly. Maybe if you wanted to get serious, look into powers of tau setup ceremony engineering style considerations so that you could get like 100s to 1000s of free participants.

[1] In practice, I don't find that canary ideas when it comes to real (and huge) financial risk are very convincing. But one thing's for sure, they don't hurt!

[2]To state the obvious, if you just add participants keys together you're open to key subtraction attacks.

-------------------------

garlonicon | 2026-05-12 08:30:35 UTC | #3

[quote="qatkk, post:1, topic:2498"]
Weaker curve (secp192r1):

[/quote]

There are four elliptic curves, which were generated at the same time, and which have similar properties: [secp160k1](https://std.neuromancer.sk/secg/secp160k1), [secp192k1](https://std.neuromancer.sk/secg/secp192k1), [secp224k1](https://std.neuromancer.sk/secg/secp224k1), and [secp256k1](https://std.neuromancer.sk/secg/secp256k1). If you calculate half of the generator, then you can see this:

```
secp160k1:         48ce563f89a0ed9414f5aa28ad0d96d6795f9c62
secp192k1: 0554123b78ce563f89a0ed9414f5aa28ad0d96d6795f9c66
secp224k1:       3b78ce563f89a0ed9414f5aa28ad0d96d6795f9c63
secp256k1:       3b78ce563f89a0ed9414f5aa28ad0d96d6795f9c63
```

Currently, even secp160k1 was still not yet broken, so we probably should start from that. Maybe even there could be more curves, designed specifically as a challenge, for example something like that: https://github.com/vjudeu/curves1000/tree/master/bits

Which means, that we could have 128-bit curve, then 129-bit curve, and so on, up to some 255-bit curve.

-------------------------

nebula-21 | 2026-05-12 16:45:20 UTC | #4

The chosen curve was mainly selected because it has weaker security, and since we were aware that similar curves exist, this one had better support for the proof of concept implementation.

The idea of having multiple canary addresses is a good one and was never discarded. However, there is also the issue of how much the reward would be split. If the idea does not gain enough traction, the canary bounty being split among multiple addresses might not be sufficient to trigger a CRQC.

-------------------------

garlonicon | 2026-05-12 18:51:30 UTC | #5

> However, there is also the issue of how much the reward would be split.

There are existing N-bit private keys on Bitcoin, for example: https://mempool.space/tx/08389f34c98c606322740c0be6a7125d9860bb8d5cb182c02f98461e5fa6cd15

However, there are many issues with this setup:

* Puzzles from 161 to 256 were [cleared, and moved to easier unsolved ones](https://mempool.space/tx/5d45587cfd1d5b0fb826805541da7d94c61fe432259e68ee26f4a04544384164), because the author thought, that going beyond 160-bit values doesn’t make sense. But it does, if keys are not hashed, so it was probably a mistake.
* There is no proof, that private keys are weak. There are many examples, that it is probably the case, but even if the author could attach more proofs, by using weaker curves, he simply couldn’t have enough knowledge to do that.
* The author can sweep all coins at any time. If you use for example secp160k1, then you know for sure, that all private keys are in range from 1 to `0x100000000000000000001b8fa16dfab9aca16b6b2`. And if you have a DLEQ proof, then you know, that the same key is used for a given Bitcoin address. It is mathematically possible to create N-bit puzzles, where everyone would need to solve it, to move them, including the puzzle creator.
* The progress is not proven trustlessly to everyone, but only to the creator: everyone else has to trust, that the author is not simply sweeping next coins every few months or years, and the actual progress is being made.

So, I think even preparing some code, which would allow attaching proofs for what we have today, would push us at least some steps forward. Because currently, we have just some examples, where people try to solve next puzzles, while heavily trusting the creator, that everything is prepared correctly. Which is far from ideal, but this is what we have here and now.

-------------------------

