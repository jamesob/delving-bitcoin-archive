# Unspendable keys in descriptors

salvatoshi | 2023-12-19 13:29:37 UTC | #1

With miniscript on taproot in core, and support coming soon in hardware signers, we will start seeing wallets exploring the new possibilities.

One thing that will probably be useful for certain use cases and that doesn't have clear specs to day is: how to create spending policies with unspendable keys?

That's particularly important in taproot, as one might desire to create a wallet that can only be spent using the script paths; for example, because all the spending conditions require a timelock, or not expressible as a musig/FROST keypath.

It's easy to create a specific unspendable keys or xpubs; for example: `xpub661MyMwAqRbcEYS8w7XLSVeEsBXy79zSzH1J8vCdxAZningWLdN3zgtU6QgnecKFpJFPpdzxKrwoaZoV44qAJewsc4kX9vGaCaBExuvJH57`, constructed by taking the NUMS point suggested in [BIP-0341](https://github.com/bitcoin/bips/blob/9c57fac1a71d20f693607c2490dec9011c205ab5/bip-0341.mediawiki) and attaching a chaincode made of 32 `0` bytes.

However, there are desirable properties that it's not trivial to achieve:

1. unspendable keys should be indistinguishable from a random key for an external observer;
2. in a descriptor with the range operator (like the [wallet policies](https://github.com/bitcoin/bips/pull/1389) compatible with most known wallet account formats), each change/address_index combination must generate a different unspendable pubkey , and they should not be relatable to each other (in order to avoid fingerprinting);
3. the fact that a certain key is unspendable should be easy to detect with full knowledge of the descriptor;
4. additional entropy needed for this key should be avoided or minimized.

Properties (3) and (4) help providing a better user experience when using such spending policies with a hardware signing devices, where minimizing what's shown on-screen is important. For security reasons, any entropy that is part of the descriptor/wallet policy _must_ be inspected by the user when they register the policy on the device, and compared with their backup. A worse user experience does in fact result in worse security in practice, especially with less experienced users - as they tend to skip the paranoid security steps.

A key observation is that the unspendable pubkey _cannot_ depend on information that is not in the descriptor itself (including private _or_ public key material generated from the seed of the participants): that would make watch-only wallets impossible.

It's unclear to me if there is any interesting use case for unspendable keys outside of the taproot keypath, but some of the solutions below might work for that as well.

I've been brainstorming some of the possible approaches, which I list below.

# Solutions
## (s0) Use a fixed unspendable xpub
Just use `xpub661MyMwAqRbcEYS8w7XLSVeEsBXy79zSzH1J8vCdxAZningWLdN3zgtU6QgnecKFpJFPpdzxKrwoaZoV44qAJewsc4kX9vGaCaBExuvJH57` above.

This is easy, but of course it only satisfies (3) and (4) and is giving up on (2) and (3)

## (s1) Use a root xpub with unspendable pubkey and random chaincode

Instead of using the above xpub, on could generate a random unspendable xpub by using a random chaincode instead of a fixed constant.

As long as the xpub is followed by a `/*` in the descriptor (as it's common in today's wallets that use `xpub/<0;1>/*`), this satisfies (1) and (2). It also satisfies (3), since the compressed public key of the root descriptor is fixed.

It is not ideal for (4), as the additional entropy must be part of the backup, and be part of the information inspected on the hardware signer screen during registration of the wallet policy.
 

## (s2) BIP-0341 approach: `H + r*G`

[BIP-0341](https://github.com/bitcoin/bips/blob/9c57fac1a71d20f693607c2490dec9011c205ab5/bip-0341.mediawiki) suggests using a pubkey generated as `H + r*G`, where `H = lift_x(0x50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0)` is the NUMS point mentioned above, and `r` is a random number in `0..n-1` where `n` is the curve order.

One can imagine a KEY expression `unspend(r)` (with a fixed chaincode so it can be used in further derivations).

Of course, since `r` is chosen from a rather large set, this has the same properties as the previous approach: (1), (2) and (3), not optimal for (4).

## (s3) Entropy from the taptree (keypath only)

Based on an idea from Antoine Poinsot: define a new fragment, say `tree(TREE)` (taptree only, without keypath), where the keypath is computed as a NUMS point with the previous approach, where `r` is the hash of the taptree.

This is ideal w.r.t goals (2), (3) and (4). Fingerprinting is not ideal, as the fact that there is no keypath is revealed at spending time.

One slight caveat compared to the approaches based on generating a provably unspendable xpub is that descriptors using the new fragment are not interoperable with wallets that didn't implement the new fragment type.

## (s4) Recycle the entropy of the descriptor

As the "entropy" of the unspendable key must be public to anyone that knows the descriptor, one approach could be to use the entropy in the descriptor itself to generate the unspendable pubkey.

A typical descriptor used in wallets today contains a number of xpubs, followed by `/<M;N>/*` (that is, usually `/0/*` for receive addresses, or `/1/*` for change addresses.

One could generate an unspendable xpub as follows:
- generate the pubkey as `H + r*G` above, where `r` is the SHA256 of the concatenation of all the compressed pubkeys in the descriptor that have a `/<M;N>/*`, in some canonical order (e.g. left-to-right in the descriptor)
- choose a fixed chaincode, like all zeros.

EDIT: perhaps simpler:
- use the fixed unspendable pubkey 
- use `r` for the chaincode.

The KEY expression could be modified to allow a special marker that represents this special unspendable xpub, say `_`.

For example, a taproot descriptor for a wallet with a single leaf that is an old-style multisig would be:

```text
tr(_/<0;1>/*,multi_a(xpub1/<0;1>/*,xpub2/<0;1>/*)
```

(which can be more succinctly represented as `tr(_/**,multi_a(@0/**,@1/**))` as a descriptor template for wallet policies)

The `_` could easily be converted to the actual pubkey if the descriptor needs to be imported in software that doesn't understand `_`.

This approach would generate the same unspendable xpub for both a descriptor, but also the corresponding [wallet policy](https://github.com/bitcoin/bips/pull/1389) (when a compatible wallet policy exists).

All the properties (1-4) above are satisfied.

TBD:
- Should more than one `_` be allowed in the descriptor? Should the same xpub generated every time? Or should it only be allowed in the taproot keypath?
- Probably `_` not followed by `/<0;1>/*` should be disallowed.
- Is there any simpler way to get all 4 properties?

# Conclusions

Among the approaches that do not change any standard, (s1) is probably the most practical today: it's straightforward for hardware signers to detect such xpubs. The UX is a bit worse, but not by a lot – as there is anyway plenty of other information that the user has to inspect anyway.

Among the forward-looking approaches that do require a syntax addition to descriptors, (s4) seems ideal in the spirit, but it feels a bit dirty and unsatisfactory when applied to descriptors. The scheme is much cleaner in the context of wallet policies, since they already separate the *descriptor template* from the xpubs, and are designed as a much more restricted language.

Implementing this only for wallet policies might also be an option if they become a more widely adopted standard to backup descriptor-based wallets, but that is probably premature today.

I look forward to your ideas and any cleaner approaches that I might have missed.

-------------------------

sipa | 2023-12-19 13:35:30 UTC | #2

At TabConf 2022 we had a little meeting where we discussed avenues for improvement to miniscript/descriptors, which included an idea for unspendable keys: https://gist.github.com/sipa/06c5c844df155d4e5044c2c8cac9c05e

I just wanted to post it here as background; I'll comment on your ideas here when I've had a chance to digest it.

-------------------------

salvatoshi | 2023-12-19 15:00:06 UTC | #3

Thanks @sipa for the additional context!

One observation is that `unspend(HEXCHAINCODE)` from your gist is actually compatible with the approach (s4) if the `HEXCHAINCODE` is deterministically computed according to the specs above, or any similar ones.

So the TL:DR is: you do need *some* good entropy for the unspendable key, but you don't need *new* entropy.

-------------------------

josibake | 2023-12-19 14:49:37 UTC | #4

[quote="salvatoshi, post:1, topic:304"]
unspendable keys should be indistinguishable from a random key for an external observer;
[/quote]

Why is (1) a desirable property? I can think of one example (BIP352 - Silent payments) where we would want to standardize the use of H in protocols that require a provably unspendable keypath. Curious if you have any counter-examples where a standard public NUMS is not desirable.

-------------------------

sipa | 2023-12-19 14:52:51 UTC | #5

[quote="josibake, post:4, topic:304"]
Why is (1) a desirable property? I can think of one example (BIP352 - Silent payments) where we would want to standardize the use of H in protocols that require a provably unspendable keypath. Curious if you have any counter-examples where a standard public NUMS is not desirable.
[/quote]

Especially in this setting, revealing a recognizably unspendable internal key at spending time (remember, internal key is revealed in script path spends) would instantly reveal to the world that this was a script-only taproot output.

-------------------------

AntoineP | 2023-12-19 14:55:19 UTC | #6

What Pieter said. Plus not revealing it to the whole world by default doesn't make it unprovable.

-------------------------

josibake | 2023-12-19 15:09:50 UTC | #7

[quote="sipa, post:5, topic:304"]
would instantly reveal to the world that this was a script-only taproot output
[/quote]

This part I understand; I'm asking for an example of _why_ revealing this to the world is bad. It certainly _feels_ safer to not reveal it, but when discussing this with @RubenSomsen in the context of BIP352 where it would be beneficial to reveal to the world that this was a script-only spend, I struggled to come up with examples of why it would be bad to reveal this.

-------------------------

salvatoshi | 2023-12-19 15:12:37 UTC | #8

It is a form of fingerprinting. You can always reveal this information yourself if you want/need it, but it's great if the standards don't force you to do so.

-------------------------

josibake | 2023-12-19 15:21:22 UTC | #9

[quote="salvatoshi, post:8, topic:304"]
It is a form of fingerprinting.
[/quote]

More of a fingerprint than the script itself? My thinking here is that in these protocols that require a provably unspendable keypath, the scripts themselves are likely sufficiently complex to be a fingerprint and must be revealed with every spend, anyways.

-------------------------

AntoineP | 2023-12-19 15:23:41 UTC | #10

It seems satisfying all those properties would prevent the possibility of creating addresses using [partial descriptors](https://github.com/bitcoin/bitcoin/issues/24114). Since you necessarily need additional information which isn't onchain.

-------------------------

sipa | 2023-12-19 15:30:02 UTC | #11

[quote="josibake, post:9, topic:304"]
More of a fingerprint than the script itself? My thinking here is that in these protocols that require a provably unspendable keypath, the scripts themselves are likely sufficiently complex to be a fingerprint and must be revealed with every spend, anyways.
[/quote]

I can certainly imagine specific use cases where this will be the case, and revealing that the key path was unspendable isn't an additional privacy loss over revealing the script itself.

But I also don't think this is universally true. Maybe the script is just covering some combination of participants unable to MuSig sign a key path. Maybe there are very distinguishable scripts in the script tree, but there are others which aren't. And I hope you'd agree that we shouldn't adopt a standard that *forces* participants to reveal their key path was unspendable.

FWIW, even with the $P = H+rG$ approach mentioned in BIP341 (with secret $r$) you can prove to contract participants that the key is unspendable, even without revealing $r$ (by producing a BIP340 signature for key $P-H$, which has private key $r$).

-------------------------

sipa | 2023-12-19 15:26:33 UTC | #12

[quote="AntoineP, post:10, topic:304, full:true"]
It seems satisfying all those properties would prevent the possibility of creating addresses using [partial descriptors](https://github.com/bitcoin/bitcoin/issues/24114). Since you necessarily need additional information which isn’t onchain.
[/quote]

I believe that's why we preferred an option where the entropy was just in the descriptor itself.

-------------------------

AntoineP | 2023-12-19 15:33:53 UTC | #13

Maybe we want different properties for descriptors and wallet policies. We could both have the entropy be explicitly described in a descriptor (`unspend()`), and omitted for wallet policies since this standard would mandate the content of `unspend()` be deterministically derivable from the rest of the descriptor.

So properties (1), (2), (3) for output descriptors. And property (4) in addition for wallet policies.

-------------------------

salvatoshi | 2023-12-19 15:40:22 UTC | #15

That’s a good point, I think you’re right!

The only issue is that people today backup descriptors and not wallet policies, so the discrepancy is a UX problem at registration time.

I think people should backup wallet policies to represent *accounts* in software wallets, except for the exotic use cases where wallet policies don’t work.

-------------------------

josibake | 2023-12-19 16:59:33 UTC | #16

[quote="sipa, post:11, topic:304"]
And I hope you’d agree that we shouldn’t adopt a standard that *forces* participants to reveal their key path was unspendable.
[/quote]

Certainly agree on this point, but also don't want to leave wallet users with the impression that this is chosen as a standard because revealing that only the script path was usable is _always_ bad for privacy.

[quote="sipa, post:11, topic:304"]
(with secret rrr) you can prove to contract participants that the key is unspendable
[/quote]

Good to know! For BIP352, the scenario I had in mind for revealing that the keypath was unusable is a coinjoin, where Alice wants to coinjoin her provably unspendable keypath UTXO and Bob wants to make a payment to a SP address. It sounds like in this scenario Alice could provide a signature for $P - H$ to a coordinator, instead of the coordinator requiring that her script path spend show that $H$ was the internal key.

EDIT: nevermind, this doesn't work. It's about the *receiver* knowing that the taproot spend was a script-only spend, and AFAICT there is no way to do this in a non-interactive way outside of making it public that only the script path was usable.

-------------------------

RandyMcMillan | 2023-12-19 18:56:24 UTC | #17

Maybe the simple solution is to rot13 the xpub

xpub\<rot13\>

-------------------------

wydengyre | 2023-12-28 02:34:27 UTC | #19

Approach s2, corresponding to the descriptor changes document linked by @sipa above, seems straightforward to implement and very simple to explain. Yes, it comes at the expense of having to include some extra information in the descriptor. This seems like a worthwhile tradeoff given its simplicity.

-------------------------

salvatoshi | 2023-12-28 16:38:10 UTC | #20

sipa's linked approach is similar but not identical to `s2`, as in his version the argument of `unspend()` alters the *chaincode*, while in the version described in my notes above I was instead generating a different pubkey.

They are identical in terms of security properties, but I think sipa's approach is better as it's more straightforward to verify that one such xpub is unspendable (just look at the pubkey), while in the approach in `s2` one has to explicitly redo the computation to verify how the xpub is generated.

 Anyway, my current thinking is that sipa's approach is probably the cleanest for descriptors, while wallet policies (now in the process of being finalized as [BIP-0388](https://github.com/bitcoin/bips/pull/1389)) could add a deterministic way of computing the `HEXCHAINCODE` from the remaining keys, as suggested above by @AntoineP.

-------------------------

AntoineP | 2024-01-16 11:03:51 UTC | #21

I'm now implementing [Taproot support in Liana](https://github.com/wizardsardine/liana/issues/55). Ideally i'd like to derive unspendable internal keys in a way which would be forward compatible with an eventual standard used in wallet policies. This way, once support is implemented in signing devices, our users won't have to "verify" a meaningless internal key on their device's screen. A friendlier "no keypath spend" UX could be presented.

It seems the less ugly way of getting all 4 properties is what Salvatore [suggests in s4](https://delvingbitcoin.org/t/unspendable-keys-in-descriptors/304#s4-recycle-the-entropy-of-the-descriptor-6) (in the simpler version). From a wallet policy, the unspendable internal key is an `xpub/<0;1>/*` such as:
- The `xpub`'s key is the NUMS suggested in BIP341: `H = lift_x(0x50929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0)`.
- The `xpub`'s chaincode is the `sha256` of the concatenation of the key part (as a compressed pubkey) of all the xpubs in the wallet policy. (In the wallet policy standard all key expressions must be xpubs with derivation path of the form `/<m;n>/*`.) "Left to right" order as they appear in the string representation, which is just a depth-first search.

Of course, this can also be expressed in descriptors using `unspend(computed chaincode)/<0;1>/*` as the internal key.

-------------------------

salvatoshi | 2024-01-16 14:20:19 UTC | #22

An issue with using left-to-write is that since wallet policies have a separate list of xpubs (that are explicitly referenced by `@index` in the descriptor template), the natural order of the keys would be the one in the list, which is not guaranteed to match the left-to-right order in the descriptor template.

E.g.:\
descriptor_template: `"tr(_,{pk(@1/**),pk(@0/**)})"`\
keys: `["xpubA", "xpubB"]`

Here the left-to-right order doesn't match with the natural order `pubkeyA||pubkeyB`.
If the `_` (or whichever other expression) to represent the deterministic NUMS key is only defined for wallet policies, I'd favor the wallet_policy-native approach.

-------------------------

AntoineP | 2024-01-16 14:34:11 UTC | #23

Your approach requires to have the wallet policy at hand, whereas mine can be used on any wallet-policy-compatible descriptor.  I think it's a nice property to have.

-------------------------

