# K-of-N threshhold key generation scheme by limited shared secrets

ZmnSCPxj | 2026-06-20 06:52:59 UTC | #1

# Introduction

The FROST scheme is based on Verifiable Secret Sharing, where signers *first* perform Shamir Secret Sharing in homomorphic space, then each signer stores the generated share it has of the final key.

However, this is slightly undesirable compared to MuSig2.  Under MuSig2, each signer *already* has a private key that is secured and backed up.  Then, any third party can generate the public key of their n-of-n using only the public keys of the signers.  In particular, there is no requirement of a setup that requires data storage.

Further, FROST technically has a different signing protocol compared to MuSig2.  Thus, while we already have a scheme for nested MuSig2 inside MuSig2, and this scheme has a proof of correctness, we still do not have FROST-in-MuSig2 scheme, much less a proof of its correctness.

In this writeup, I will present a ***novel*** scheme for generating a K-of-N threshhold public key for N signers, with the following advantages:

* If K = N - 1, the signers can use their existing private keys and do not have to store an extra "share" or secret that is created during setup; in fact, setup is minimal for K = N - 1 and involves only exchanging public keys of the signers and then exchanging public keys for the shares (however for K < N - 1, it requires storing additional secret shares).
* The actual signing protocol between the signers is ***exactly*** MuSig2.  This extends out to the existing MuSig2-in-MuSig2 scheme, i.e. we can now have a k-of-n inside an n-of-n, while relying on the MuSig2-in-MuSig2 scheme and its proof.  This is true for both K < N - 1 and K = N - 1.
* It creates shared secrets that can be used to deterministically derive shachain roots for the proposed "multiple shachain" scheme for Lightning.

It has the following disadvantages:

* If K < N - 1, each signer must store multiple secret shares.
* ***NOVEL CRYPTOGRAPHY*** created by a dilettante with ***NO*** mathematical education.

# Recalling `mo_shachains_mo_signers`

Recently, some random dilettante with delusions of cryptographic understanding [resurrected the idea of using multiple shachains for revocation in order to support k-of-n Lightning](https://delvingbitcoin.org/t/towards-a-k-of-n-lightning-network-node/2395/4?u=zmnscpxj).

Specifically, this *dilettante* gave the following scheme:

* For a `k`-of-`n` scheme, create `s` secrets known by subsets of the signers.
* Calculate `s` as "`n` choose `n - k + 1`", or `s = n! / ((n - k + 1)! * (k - 1)!)`.
* For each shared secret `0..s-1`, select `n - k + 1` signers.

So for example, for a 4-of-5 scheme `s = 5! / (2! * 3!)` `s = 10`.  Then for the secrets `[0]...[s-1]`, with the five signers `A` `B` `C` `D` `E`, we can select each secret to be known by the following table:

```
[0] : A B
[1] : A C
[2] : A D
[3] : A E
[4] : B C
[5] : B D
[6] : B E
[7] : C D
[8] : C E
[9] : D E
```

In the table above, it means that the share `s[0]` is known by signers `A` and `B.

Now, if there are **TWO** signers with their own existing private keys, there is an algorithm called ECDH by which ***either*** can generate a secret that is learnable only by either of them; all that they need is the public key of the other.  This case of "each shard is known by 2 signers" happens for K = N - 1; for K < N - 1, more than 2 signers need to know each share, which requires a 3-party ECDH.

In the above example, `A` and `B` can calcualte the shard `s[0]` by ECDH of their keys.

For generalized N-party ECDH, we can use a ritual like the below which we can generalize:

* Example `A` `B` `C`.
* `B` calculates `b * A`, and gives that point, plus a DLEQ proof that `(b * A) / (b * G) == A / G`, to `C`.
* `C` validates the DLEQ proof, then calculates `c * b * A` from the `b * A` given by `B`.  It then gives that point, plus a DLEQ proof that `(c * b * A) / (c * G) == b * A / G` to `A` and `B` (for the general case where number of parties exceeds 3, it must also copy the `b * A` and the DLEQ for that, as well, with each other participant getting a copy of that as well).
* Note that `c * b * A` ***IS*** the shared secret (or more traditionally, the SHA256 of that point), so the signers should use encrypted and integrity communcations.

Then, as noted in the linked post, we can then perform `MuSig2({s[0]...s[9]})` for all secrets, and that MuSig2 combination would now be signable by a 4-of-5 quorum.

The major advantage here is that the shared secrets can then be used to derive shachain roots.  For example, the shared secret `s[0]` can be used to HMAC some deterministic data (such as the peer node ID and some unique identifier for the channel witht that peer, e.g. CLN `dbid` or LDK `channel_key_id`) and the result can be used for the shachain root for that index.

On signing, we need to determine the set of participants (i.e. the quorum `K`) and which ones need to provide nonces et al for which shards `s[i]` they know.  This can be done deterministnically.  Then, the participants simply use a MuSig2 protocol; the only difference is that some of the participants sign for more than one shard.

As noted, use of MuSig2 implies we can use this scheme for a k-of-n Lightning Network node while relying ***only*** on the MuSig2-in-MuSig2 paper, without further requirement to have a FROST-in-MuSig2 proof.

# Practical Use

2-of-3 is one of the simplest and useful multisignature policies; it allows loss of one signer, while requiring that thieves compromise two signers in order to steal.

2-of-3 also matches the K = N - 1 case in this scheme.  This means that a signer can derive the shards with only knowledge of the public keys of the 2 other signers, and without any setup ritual; even a signer with no persistent storage can sign it if you provide the public keys of the other signers (however, do note that a non-blind Lightning Network signer ***must*** have its own mutable persistent storage anyway).

As a concrete example, for a 2-of-3, we can have 3 signers `A` `B` and `C`.  This needs three shards:

```
[0] : A B
[1] : B C
[2] : A C
```

Suppose that for a particular signing session, `A` and `C` are online.  We then decide that `A` will provide partial signatures for `s[0]` and `s[2]`, and `C` provides the partial signature for `s[1]`.  The two online participants then use MuSig2 signing for the shards `s[0]` `s[1]` and `s[2]`, which can also be nested inside another MuSig2 signing party.

Further, it is also directly compatible with the multiple-shachains scheme being proposed for Lightning; we can derive using a hardened derivation from the shards, and then get the derived key as the root for the shachains needed for revocation as well.

-------------------------

