# Public key recovery for EC leaves in P2MR (BIP-360)

starius | 2026-06-06 22:00:54 UTC | #1

# Public key recovery for EC leaves in P2MR

This note describes a possible [BIP-360](https://bips.dev/360/) optimization using EC public key recovery.

The idea is to have a P2MR output with a cheap EC leaf and an expensive PQ leaf. The EC leaf is the common spend, the PQ leaf is the quantum emergency path. This optimization only changes the EC witness size. It does not make the quantum exposure worse than a normal P2MR EC leaf: the EC public key is still hidden behind the P2MR hash tree until the EC path is spent.

The proposal is to remove the script containing the EC public key from the witness of P2MR's EC spending. It is implemented as a special P2MR leaf type, called here a *recoverable EC leaf*, where the EC public key is recovered from the signature. This makes the size of such a witness close to P2WPKH, even a bit shorter. Without the optimization it would be about 25% larger than P2WPKH (see size comparison below).

## Key Recovery and Key Prefixing

Plain Schnorr supports key recovery from a signature, but [BIP-340](https://bips.dev/340/)-style verification normally does not, because of key prefixing. To prevent related-key attacks on [BIP-32](https://bips.dev/32/) unhardened keys, BIP-340 puts the EC public key into the challenge:

> **Key prefixing** Using the verification rule above directly makes Schnorr signatures vulnerable to "related-key attacks" in which a third party can convert a signature `(R, s)` for public key `P` into a signature `(R, s + a⋅hash(R || m))` for public key `P + a⋅G` and the same message `m`, for any given additive tweak `a` to the signing key. This would render signatures insecure when keys are generated using BIP32's unhardened derivation and other methods that rely on additive tweaks to existing keys such as Taproot.
>
> To protect against these attacks, we choose *key prefixed* Schnorr signatures which means that the public key is prefixed to the message in the challenge hash input. This changes the equation to `s⋅G = R + hash(R || P || m)⋅P`. It can be shown that key prefixing protects against related-key attacks with additive tweaks.

However, this creates a circular dependency and prevents extracting the public key from the signature.

The idea is to put the P2MR root `q`, which commits to the EC public key, into the challenge instead of putting the EC public key itself there. This breaks the circular dependency and makes EC public key recovery possible.

Binding the signature challenge hash `e = H(R || q || m)` to the P2MR root hash `q` still binds the signature to a specific P2MR tree. This prevents reusing or tweaking a signature to recompute a related key `P' = P + t⋅G` under a different tree `q' != q`, because the verifier will compute a different challenge hash `e' = H(R || q' || m)` when verifying the signature.

However, committing only to `q` is not enough if the same P2MR tree contains multiple recoverable EC leaves. The challenge would be the same for all of them, so a related-key signature can be moved from one leaf to another. The fix is to also commit to the exact serialized control block used for this spend. Then the signature is bound both to this P2MR output and to this leaf path inside the tree.

The generic shape is:

```
s⋅G = R + H(R || P || m)⋅P
```

becomes:

```
s⋅G = R + H(R || q || control_block_hash || m)⋅P
```

Mathematically, recovery is just the Schnorr verification equation solved for `P`. Given a signature `(R_x, s)`, lift `R_x` to the even-Y point `R` as in BIP340, compute the challenge `e`, and require `e != 0`. Then:

```
s⋅G = R + e⋅P
e⋅P = s⋅G - R
P = e^-1 ⋅ (s⋅G - R)
```

The recovered `P` is accepted only if it is a valid EC public key and if its compressed 33-byte encoding, using the same `control_block`, recomputes `q`, the P2MR Merkle root.

The Schnorr equation remains linear:

```
s⋅G = R + e⋅P
```

so aggregation-style protocols may be possible, but MuSig2/FROST would need adapted protocol definitions and proofs.

However, this does not preserve normal BIP340-style batch verification. BIP340 batching needs each `P_i` to be an independent public input, which we no longer have. The price of preserving batch verification is revealing `P` in the witness.

## The proposed construction

Use a recoverable EC leaf in BIP-360. It is a special leaf version in the control byte. It does not support script and serves just one purpose: fee-efficient Schnorr. The proposal is to fold this into BIP-360 before activation, because adding it later requires another witness version rather than just another leaf version: the recoverable EC leaf changes both witness parsing and the leaf-hash preimage, so old BIP-360 nodes would not validate it under the normal script-leaf rules.

Current P2MR validation computes the leaf hash and checks the Merkle path before script execution:

```
k0 = TaggedHash("TapLeaf", v || compact_size(size of script) || script)
r  = MerkleRoot(k0, control_block_path)
require r == q
execute script
```

In a recoverable EC leaf, there is no script containing the EC public key. So this must be a special case in P2MR validation logic, after annex removal and before the normal "second-to-last witness element is the script" rule.

Proposed witness shape:

```
<signature> <control-block> [ <annex> ]
```

The signature is 64 or 65 bytes, with the optional sighash byte as in Taproot. There is no script component.

The control block is a single witness element:

```
control_block = control_byte || 32-byte Merkle path element 0 || ...
```

For Merkle path depth `d`, the control block length is `1 + 32×d`. The control byte is `leaf_version | 0x01`, following BIP-360. For example, if we assign a fresh leaf version `0xc4`, then the control block starts with `0xc5`.

The recoverable EC leaf logic is applied if the output is P2MR and the control byte selects this leaf version. For this leaf version, the witness must match this shape. It must not fall through to normal script-leaf or future-leaf validation.

Validation:

```
const RECOVERABLE_EC_LEAF_VERSION = 0xc4

q = P2MR witness program, the 32-byte Merkle root
v = control_block[0] & 0xfe

require control_block[0] & 1 == 1
require v == RECOVERABLE_EC_LEAF_VERSION
require witness shape == <signature> <control-block> [ <annex> ]

hash_type = signature sighash byte, or SIGHASH_DEFAULT if absent
m = TapSighash(hash_type, ext_flag = 0)
control_block_hash = TaggedHash("P2MRRecoverable/control_block", control_block)

require R_x is a valid public nonce encoding
require s is a valid scalar
e = TaggedHash("P2MRRecoverable/challenge", R_x || q || control_block_hash || m) mod n
require e != 0
P = recover_pubkey(R_x, s, e)
require recovery succeeds
require P is a valid EC public key

leaf_hash = TapLeaf(v, bytes(P))
          = TaggedHash("TapLeaf", v || compact_size(33) || bytes(P))
r = MerkleRoot(leaf_hash, control_block_path)

require r == q
```

Here `bytes(P)` is the normal 33-byte compressed public key encoding of the recovered EC public key. The signature still stores only the 32-byte `R_x` public nonce.

The signer needs to know the control block before signing and must check that its own public key leaf and this control block recompute `q`.

Here `m` is exactly the [BIP-341](https://bips.dev/341/) key-path style transaction digest, even though the output being spent is a P2MR output. For the common 64-byte `SIGHASH_DEFAULT` case:

```
m = TaggedHash("TapSighash",
        0x00 || 0x00 ||
        tx.version || tx.locktime ||
        sha_prevouts || sha_amounts || sha_scriptpubkeys || sha_sequences ||
        sha_outputs ||
        spend_type || input_index ||
        [sha_annex if annex is present])

spend_type = 0x01 if annex is present, otherwise 0x00
```

Other sighash modes follow the BIP341 rules for `SIGHASH_ANYONECANPAY`, `SIGHASH_NONE`, `SIGHASH_SINGLE`, etc.

In current BIP341 sighash, `m` already commits to the spent scriptPubKey, so it already commits to `q` indirectly. `q` is still put directly into the recoverable EC challenge (`R_x || q || control_block_hash || m`) so the signature rule is bound to this P2MR output. `control_block_hash` is put there so the signature is also bound to this selected leaf path inside the tree.

`ext_flag = 0` is passed so `m` does not contain `tapleaf_hash`, `key_version`, or `codeseparator_pos`. If `m` contained `tapleaf_hash`, then `m` would depend on `P`, which is not known until after recovery.

This means P2MR would have two sighash styles: normal script leaves use the regular script-path digest, while this recoverable EC leaf uses the key-path style transaction digest with `ext_flag = 0`.

Sigops should be accounted explicitly: this recoverable EC leaf check should charge a fixed validation cost, probably one signature operation. The exact formula is yet to be decided.

### Nonce derivation

The important rule is that the deterministic nonce input must cover everything that can change the challenge: `q`, `control_block`, and `m`. A default [BIP340 signer](https://github.com/bitcoin/bips/blob/2ffcd9a4a1b834434f3c02718242235f9e270e94/bip-0340.mediawiki#user-content-Default_Signing) or a libsecp256k1 Schnorr signer used unchanged is unsafe here. If the same nonce is reused with two different control blocks, the private key is revealed.

The following nonce derivation can be used instead:

```
aux = 32 bytes of auxiliary randomness, or 32 zero bytes
t = bytes(sk) xor TaggedHash("P2MRRecoverable/aux", aux)
nonce_input = t || bytes(P) || q || control_block_hash || m
k0 = TaggedHash("P2MRRecoverable/nonce", nonce_input) mod n
require k0 != 0
R = k0⋅G
k = k0 if R has even Y, otherwise n-k0
```

Here `P = sk⋅G` is the compressed public key committed by this leaf. Unlike BIP340 signing, the secret key must not be negated just to make `P` even-Y, because the committed key is the full compressed key. Only the public nonce `R` is normalized to even-Y, as in BIP340.

## The savings

These numbers apply to any depth-1 P2MR EC leaf, where the EC leaf has one 32-byte Merkle sibling. This is the realistic case for the intended use: one cheap EC leaf and one PQ leaf.

The standard P2MR EC leaf witness has the following components:

```
witness element count: 1 byte
signature element:     1 + 64 = 65 bytes
leaf script element:   1 + 34 = 35 bytes
control block element: 1 + 33 = 34 bytes
-----------------------------------------
total:                 135 bytes
```

The recoverable EC leaf removes the leaf script element, which is 35 bytes:

```
OP_PUSHBYTES_32 <32-byte public key> OP_CHECKSIG
```

so its witness size becomes 100 bytes.

For comparison:

 - P2TR key spend: 66-67 bytes
 - P2WPKH: 107-108 bytes

This does not make a full create-and-spend cycle cheaper than P2WPKH, because the P2MR pkscript is larger. But it makes the common EC spend witness smaller than P2WPKH and reduces the cost of using P2MR in the EC path.

## Acknowledgements

Thanks to Conduition for helping with this proposal, and in particular for finding the in-tree related-key attack and the fix: committing the signature challenge to the control block through `control_block_hash`.

-------------------------

sipa | 2026-06-07 03:44:47 UTC | #2

Thank you for writing this out in detail; I hadn't considered the possibility of using root + Merkle branch instead of the public key in the challenge. My intuition is that this works, and that security properties of BIP340 and script signatures in general will carry over. For more complex constructions built on top, like MuSig2, we'd need new security proofs though. I'm not actually convinced including the Merkle path in the challenge is necessary, though it definitely does not hurt. Do you see any practical attacks from not doing that? A tree which has the same public key in multiple leafs would allow malleation without it, but are there more severe issues?

The downside of this approach is that it gives up the ability for batch validation, which we used as a design requirement in BIP 340-342. It can be debated how important that is, but without it, we could also have saved a byte in P2TR script path spends (the sign of the tweaked output point wouldn't need to be revealed).

Ignoring the exact byte sencodings, and just counting hashes/keys as 32 and signatures as 64, for a 2-leaf output (1 key, 1 script), I believe we get:

| Scheme | Key spend | Script spend |
| --- | --- | --- |
| P2TR | 64 | 32+script+inputs |
| P2MR | 128 | 32+script+inputs |
| This scheme | 96 | 32+script+inputs |

Does that sound right? So it recovers half of the happy-path benefit of P2TR, at the cost of batch validation and black-box usage of the signature scheme, but gains the ability to not reveal the public key until key path spend time.

-------------------------

starius | 2026-06-07 07:12:14 UTC | #3

Hi Pieter,

Thanks for reviewing the proposal!

> Do you see any practical attacks from not doing that? A tree which has the same public key in multiple leafs would allow malleation without it, but are there more severe issues?

If there is a tree with related EC keys `P` and `P' = P + t⋅G` in different leaves of the same tree and `e` does not commit to the control block (i.e. to the path in the tree), then it is possible to forge a signature for `P'` given a valid signature for `P`.

Assume we have an adversary who knows the tree structure, `P`, `P'`, and the tweak `t`. The adversary can query the signer for a signature under `P` on a transaction (`R`, `s`) and then forge a signature for key `P'` from another leaf:

```
s⋅G = R + e⋅P
```

The adversary computes:

```
s' = s + e⋅t
```

Then:

```
s'⋅G = s⋅G + e⋅t⋅G
     = R + e⋅P + e⋅t⋅G
     = R + e⋅(P + t⋅G)
     = R + e⋅P'
```

So `(R, s')` verifies for `P'`. Note that `e` is the same in both signatures, since it does not commit to the control block.

Thanks to Conduition for discovering this in-tree version of the related-key attack!

> Does that sound right? So it recovers half of the happy-path benefit of P2TR, at the cost of batch validation and black-box usage of the signature scheme, but gains the ability to not reveal the public key until key path spend time.

Yes, this is correct.

**UPD**. I assume that in the P2TR case in the table there is only a single script leaf, because the key path is the key spend, not a leaf. For a literal 2-leaf P2TR output, the **script spend** would be 32 bytes larger.

-------------------------

sipa | 2026-06-07 14:09:27 UTC | #4

[quote="starius, post:3, topic:2603"]
If there is a tree with related EC keys `P` and `P' = P + t⋅G` in different leaves of the same tree and `e` does not commit to the control block (i.e. to the path in the tree), then it is possible to forge a signature for `P'` given a valid signature for `P`.
[/quote]

Right, nice catch.


[quote="starius, post:3, topic:2603"]
**UPD**. I assume that in the P2TR case in the table there is only a single script leaf, because the key path is the key spend, not a leaf. For a literal 2-leaf P2TR output, the **script spend** would be 32 bytes larger.
[/quote]

Yeah, I just meant an output that allows spending through either a key or a script path. Whether that's tweak-based of tree-based is an implementation detail of the scheme.

-------------------------

ajtowns | 2026-06-08 07:05:10 UTC | #5

It seems to me like any EC keys used in P2MR constructions like this should be hardened (so not derivable from an xpub), since revealing the pubkey to a quantum capable attacker negates the benefit of using P2MR in the first place.

If you are doing hardened keys, then calculating `t` for the `P vs P'` case is only possible if you know the private keys for both public keys, in which case it's not really "forging" a signature for the other key. (Obviously you've solved that problem already though)

[quote="sipa, post:2, topic:2603"]
Does that sound right? So it recovers half of the happy-path benefit of P2TR, at the cost of batch validation and black-box usage of the signature scheme, but gains the ability to not reveal the public key until key path spend time.
[/quote]

Batch validation is potentially ~~a 20% performance improvement~~ per [secp256k1#1134](https://github.com/bitcoin-core/secp256k1/pull/1134/) while these spends are ~14% more expensive (65.5 vbytes per input vs 57.5 vbytes per input for a taproot keypath spend?), so I think that can already be considered roughly accounted for? Not having to calculate a taptweak for script path spends is a saving on the validation side as well, that might also be worth factoring in.

EDIT: Oh, my bad -- more recent updates on that PR suggests 55%-80% improvements are viable. So if a batchable sig costs 50 weight units, then a non-batchable sig should cost perhaps 77.5 (+55%) to 90 (+80%) weight units, and these sigs are naturally 64 weight units and 96 weight units, so that's still arguable okay per the existing measures...

-------------------------

sipa | 2026-06-08 11:59:49 UTC | #6

[quote="ajtowns, post:5, topic:2603"]
seems to me like any EC keys used in P2MR constructions like this should be hardened (so not derivable from an xpub), since revealing the pubkey to a quantum capable attacker negates the benefit of using P2MR in the first place.
[/quote]

It suffices to not reveal any (unhardened) xpubs with untrusted parties, and (as always in this setting) not reuse keys, I think? Using xpubs to derive the keys itself is fine if they are kept secret, because quantum attackers cannot discover the chaincode.

Of course, using xpubs in this manner makes them practically equivalent to hardened derivation anyway I guess, if you need to guard the chaincode as much as you'd orherwise need to guard the public key?

-------------------------

RubenSomsen | 2026-06-08 19:09:48 UTC | #7

Interesting observation that the hash of the pubkey is just as good as the pubkey itself for the Schnorr challenge and that this can salvage pubkey recovery. While we're talking about this in the context of P2MR, I'd say the observation is more general than that. In theory this could have been applied to taproot script paths when it was first conceived, or could be added with a new leaf version if deemed desirable (big if).

While batch validation breaks, and this may warrant a weight unit increase, there is some nuance - you could still choose to download the pubkeys together with the block, giving you the option to do batch verification (and save CPU) at the cost of bandwidth. And of course at the tip (without blocksonly) you're not batch validating incoming transactions anyway.

**Edit:** I have one additional observation, though I have yet to find a practical use case - since the outpoint is also a commitment to the pubkey and is readily available on the input side, you could use this in the challenge instead of the pubkey (assuming a segwit v2 soft fork), enabling pubkey recovery for taproot key path spends.

-------------------------

conduition | 2026-06-08 23:37:25 UTC | #8

Nice work @starius, very glad to see more eyes on this clever idea.

[quote="ajtowns, post:5, topic:2603"]
It seems to me like any EC keys used in P2MR constructions like this should be hardened (so not derivable from an xpub), since revealing the pubkey to a quantum capable attacker negates the benefit of using P2MR in the first place.

If you are doing hardened keys, then calculating `t` for the `P vs P'` case is only possible if you know the private keys for both public keys, in which case it’s not really “forging” a signature for the other key. (Obviously you’ve solved that problem already though)
[/quote]

[quote="sipa, post:6, topic:2603"]
It suffices to not reveal any (unhardened) xpubs with untrusted parties, and (as always in this setting) not reuse keys, I think? Using xpubs to derive the keys itself is fine if they are kept secret, because quantum attackers cannot discover the chaincode.
[/quote]

Once you start thinking about xpubs post-Q-day things will start getting complicated. Nothing about Boris' idea here has changed that: The security status-quo around related keys and quantum adversaries remains just as complex as it would've been without recoverable EC leaves. 

We're going to have to completely rethink the concept of xpubs when it comes to hash-based signature schemes anyway, since we can't derive secure child keys from a hash-based pubkey. I have posted about this before [here](https://groups.google.com/g/bitcoindev/c/5tLKm8RsrZ0/m/WE-R3z85AAAJ) and I have [a rough sketch for what future xpubs may someday look like here](https://hackmd.io/@conduition/HkyM7uJtZg). But maybe it'd be better to discard the idea of xpubs and just provide batches of addresses to senders instead. This would be far safer because CRQC adversaries cannot crack open an individual address unless they learn the internal EC/HB pubkeys and scripts involved. This way receivers could generate unique hash-based keys for every address, making the quantum-fallback leaf more efficient (they can use unbalanced XMSS trees). 

In any case, today's BIP32 standard should probably not be applied to EC keys in the context P2MR because of how risky/complex it is.

[quote="ajtowns, post:5, topic:2603"]
EDIT: Oh, my bad – more recent updates on that PR suggests 55%-80% improvements are viable. So if a batchable sig costs 50 weight units, then a non-batchable sig should cost perhaps 77.5 (+55%) to 90 (+80%) weight units, and these sigs are naturally 64 weight units and 96 weight units, so that’s still arguable okay per the existing measures…
[/quote]

Technically it could be possible under the current version of BIP360 for a recoverable EC leaf to compete with taproot's witness size using a depth-zero P2MR tree. If the P2MR root hash is just a hash of the recoverable EC pubkey, then you just need a 1-byte control block, plus the 64 byte signature as the witness. This is only a byte or two larger than P2TR's witness in that case, but you don't get any secondary spending paths (e.g. for a PQ leaf). Then the signature weight numbers you quoted might not match up so well.

If we banned depth-zero P2MR trees as I've suggested recently, this would prevent the mismatch.

[quote="RubenSomsen, post:7, topic:2603"]
Interesting observation that the hash of the pubkey is just as good as the pubkey itself for the Schnorr challenge and that this can salvage pubkey recovery. While we’re talking about this in the context of P2MR, I’d say the observation is more general than that. In theory this could have been applied to taproot script paths when it was first conceived, or could be added with a new leaf version if deemed desirable (big if).
[/quote]

Very true. We could've designed a version of P2TR which commits $q = H(P)$ on-chain in the witness program, and the spender reveals either (1) a signature that allows the verifier to recover $P$ and thus $H(P)$, or (2) an internal key $\hat P$ and script $S$ s.t. $P = \hat P + H(\hat P\ \|\ S) \cdot G$.

Originally Boris suggested a variant of this idea as a way to hybridize Schnorr and SHRINCS: 

https://groups.google.com/g/bitcoindev/c/nMO4hyEm5qc/m/8wy900TXBwAJ

It is a very flexible idea but you also have to be *very* careful about how you use it. 

For example, Boris' construction currently defines the new leaf version using just the hash of a single compressed pubkey. But if we allowed a leaf to contain *multiple* EC pubkeys, this would allow related-key attacks: If two related EC keys are used inside the same leaf, then the challenge hash $e = H(R\ \|\ q\ \|\ \mathsf{ctrl\_block}\ \|\ m)$ would be the same for both keys and permit forgeries as @starius showed earlier.

There will clearly be some new security notions needed to formally prove this technique secure. Intuitively, it seems like to avoid related-key attacks we must require $e$ to commit to a hash of the pubkey $q = H(P\ \|\  ...)$, but also $e$ must commit to the other commitment hash inputs (the $...$ part), or else the signature can be rebound to other constituent inputs of $q$.

-------------------------

sipa | 2026-06-08 23:39:06 UTC | #9

[quote="RubenSomsen, post:7, topic:2603"]
**Edit:** I have one additional observation, though I have yet to find a practical use case - since the outpoint is also a commitment to the pubkey and is readily available on the input side, you could use this in the challenge instead of the pubkey (assuming a segwit v2 soft fork), enabling pubkey recovery for taproot key path spends.
[/quote]

Indeed!

But it suffers from the same downsides as other uses of recovery shenanigans: no batch validation, and needing to break the abstraction around the signature scheme.

-------------------------

sipa | 2026-06-08 23:50:04 UTC | #10

[quote="conduition, post:8, topic:2603"]
In any case, today’s BIP32 standard should probably not be applied to EC keys in the context P2MR because of how risky/complex it is.
[/quote]

I agree, but I also think this mostly destroys the appeal of P2MR. For most users, if they can't use BIP32 (or substantially similar things), I am strongly of the opinion they just won't migrate, or do it insecurely. It's already dubious even with the weakest of burdens to adoption (like P2TRv2), but basically an impossibility in my mind if we demand users simultaneously adopt entirely new concepts of what a "wallet" even is.

I'm aware of my own bias here, having (co-)authored BIP32 and other wallet standards. Still, I don't think I am exaggerating when believing that a very significant portion of the ecosystem is effectively built around xpub and public key sharing (including all multisig, escrow services, software watchonly wallets based on descriptors, wallet indexing services, ...), and *none* of that is something one can use in EC-P2MR if one hopes to have their coins survive Q-day (without the expectation of an EC-disabling consensus change, which seems like the only reason to want it in the first place).

-------------------------

conduition | 2026-06-08 23:46:34 UTC | #11

[quote="sipa, post:10, topic:2603"]
I agree, but I also think this mostly destroys the appeal of P2MR. For most users, if they can’t use BIP32 (or substantially similar things), I am strongly of the opinion they just won’t migrate. It’s already dubious even with the weakest of burdens to adoption (like P2TRv2), but basically an impossibility in my mind if we demand users simultaneously adopt entirely new concepts of what a “wallet” even is.
[/quote]

I'm not saying we deprecate HD wallets. Those are still going to be universal. I'm saying BIP32 specifically as a standard will need to be replaced one way or another. We can still use BIP39 seeds, we just need a new derivation standard that is aware of SHRINCS (or whatever other scheme we deploy).

[quote="sipa, post:2, topic:2603"]
Does that sound right? So it recovers half of the happy-path benefit of P2TR, at the cost of batch validation and black-box usage of the signature scheme, but gains the ability to not reveal the public key until key path spend time.
[/quote]


Perhaps it is still plausible to treat the new recoverable signature scheme as a black box. Just think of it as regular non-key-prefixed Schnorr signature, but with extra data prepended to the message.

-------------------------

sipa | 2026-06-08 23:53:38 UTC | #12

[quote="conduition, post:11, topic:2603"]
I’m saying BIP32 specifically as a standard will need to be replaced one way or another. We can still use BIP39 seeds, we just need a new derivation standard that is aware of SHRINCS (or whatever other scheme we deploy).
[/quote]

I think that's too narrow, the problem isn't (just) BIP32. The concern is just how much of the ecosystem is built around processes that share public keys or xpubs. See my edit above for a non-exhaustive list.

-------------------------

conduition | 2026-06-09 01:06:13 UTC | #13

[quote="sipa, post:10, topic:2603"]
I’m aware of my own bias here, having (co-)authored BIP32 and other wallet standards. Still, I don’t think I am exaggerating when believing that a very significant portion of the ecosystem is effectively built around xpub and public key sharing (including all multisig, escrow services, software watchonly wallets based on descriptors, wallet indexing services, …), and *none* of that is something one can use in EC-P2MR if one hopes to have their coins survive Q-day (without the expectation of an EC-disabling consensus change, which seems like the only reason to want it in the first place).
[/quote]

It'd be a hard pill to swallow but it's not the end of the world. I'm sure some use-cases would be fine with "just-a-pack-of-addresses", like maybe watch-only wallets or indexers. For escrow services, one might design non-interactive EC multisig scripts which don't require sharing bare EC pubkeys, but rather a hash of a pubkey. For instance, the good ol' P2PKH script pattern can be generalized to $n$ signers in a way that doesn't require sharing pubkeys before spending. 

Other cases would be more difficult. MuSig2 for instance, which _requires_ participants to share their individual pubkeys. In those cases you just kinda have to trust that your cosigners aren't friends with a CRQC.

i'm hopeful we can get isogenies prod-ready eventually. Isogeny schemes would let us replace BIP32 with a new drop-in standard. Maybe someone will come up with a good compact rerandomization scheme for lattices too at some point. 

But with the EC + hash-based addresses that we'll have in the near-term, it's a non-starter. We're going to have to design any such new standards to encourage users *not to share* their extended keys, because those keys could lead to harvest-now-steal-later attacks, as you say (unless we disable EC spending after Q-day, which I'm all for if we can manage it).

-------------------------

ajtowns | 2026-06-09 16:09:07 UTC | #14

[quote="sipa, post:6, topic:2603"]
It suffices to not reveal any (unhardened) xpubs with untrusted parties, and (as always in this setting) not reuse keys, I think? Using xpubs to derive the keys itself is fine if they are kept secret, because quantum attackers cannot discover the chaincode.

Of course, using xpubs in this manner makes them practically equivalent to hardened derivation anyway I guess, if you need to guard the chaincode as much as you’d orherwise need to guard the public key?
[/quote]

Yeah, I mean it more in the "philosophical" sense in that you essentially have to treat any derivation sources as private, not public -- it just seems much easier to explain "you're familiar with xprv and xpub, these things are essentially xprvs", rather than saying "it's okay to use an xpub here just not in the ways you're familiar with, and if you do it wrong, it will probably actually be fine, unless there's a catastrophic event in which case it won't be, and of course you're only doing this in the first place because you're worried about the catastrophic event".

Also, I think from a descriptor point-of-view you don't want to include the xpub there either (in case you're not keeping descriptors as secure as your private key seed), and would be better off having it appear as something like:

```txt
hpkh([deadbeef/0h/0h/0h]121c5dd3ae08612e2c4bcb958cc0bb6a12739246736fa09fa6d41f1b0d68780d)
```

where you provide the origin (so that you can figure out how to sign if you have the "deadbeef" private key) and the hash of the public key, rather than the public key itself. That sort of descriptor looks more reminiscent of hardened pubkeys to me (eg, no "/1/2" after the hash is possible).

[quote="conduition, post:8, topic:2603"]
If we banned depth-zero P2MR trees as I’ve suggested recently, this would prevent the mismatch.
[/quote]

Using P2MR addresses with no post-quantum signing path does seem fairly pointless...

[quote="sipa, post:10, topic:2603"]
I agree, but I also think this mostly destroys the appeal of P2MR. For most users, if they can’t use BIP32 (or substantially similar things), I am strongly of the opinion they just won’t migrate, or do it insecurely.
[/quote]

I think the use case here isn't really clear; I don't think switching the entire BTC ecosystem to something PQ secure is plausible as a single step. To my mind, there's perhaps three potential phases:

 * people are afraid of quantum but there's no immediate threat: they'd like the ability to park some funds in quantum-safe addresses, and the ability to build tools to prepare for the threat materialising, but all day-to-day activity will continue working in the existing paradigm

 * the quantum threat becomes real and imminent: almost everyone wants to move their coins to a secure address, even if that means, eg shutting down the entire lightning network, not being able to accept silent payments, not being able to easily query their balance or transaction history, etc. when the bombs are falling, you go to the bunker, even if it doesn't have the creature comforts you're used to.

 * Q-day is in the past, and we're building the best things we can build in light of that, with whatever fancy new maths we've come up with in the meantime determining whether our glory days are in the past or the future

Targeting large cold-ish wallets so that they can move into a "bunker" in the near future if desired, while leaving scripting hooks that can make the address format more featureful if/when ECC gets broken and good PQ solutions are found seems like a pretty fair balance to me. I think there is a lot of tooling to be built/updated for even that fairly limited scope of a project though.

-------------------------

AntoineP | 2026-06-09 17:55:58 UTC | #15

[quote="ajtowns, post:14, topic:2603"]
Targeting large cold-ish wallets so that they can move into a “bunker” in the near future if desired, while leaving scripting hooks that can make the address format more featureful if/when ECC gets broken and good PQ solutions are found seems like a pretty fair balance to me.
[/quote]

I think this is the wrong target. Large holders, because they have so much at stake, can be expected to react reasonably quickly during the second step you are describing. However migrating Bitcoin to PQ requires migrating the ~entirety of its current users to stop relying on EC crypto. So i think the target in the near future should be to provide a migration option to those users we expect to be the least reactive, such as we enter step 2 with as few users that haven't opted into a migration as possible.

-------------------------

