# Revisiting BIP21

josibake | 2024-03-01 14:51:14 UTC | #1

Recently had a discussion on [BIP352](https://github.com/bitcoin/bips/pull/1458#issuecomment-1954819980) regarding specifying a BIP21 extension for silent payment addresses and I felt it would be better to open this as a broader topic since its not BIP352 specific and this also came up in the newly proposed [human readable payment instructions](https://github.com/bitcoin/bips/pull/1551#discussion_r1485390796) BIP proposal. 

## BIP21 (the spec)
Here is a quick summary of the relevant (for this discussion) parts of [BIP21](https://github.com/bitcoin/bips/blob/b3701faef2bdb98a0d7ace4eedbeefa2da4c89ed/bip-0021.mediawiki):

* URIs must have a base58 encoded (legacy) address at the root
* New address types are specified via extension parameters, e.g. `?p2sh=`, `?bech32=`, `?bech32m=`
* New payment protocols are specified via extension parameters, e.g. `?pj=`, `?lightning=`
* URIs should not represent an exchange of personal information, but a one-time payment
* (not mentioned in BIP21, but worth mentioning for this discussion) For privacy conscious users, URIs should only be used over a secure channel i.e. not posted in public, even when only used once

In theory, this provides a fully backwards compatible URI scheme. You can imagine a BIP21 URI of the form `bitcoin:1..?p2sh=3..?bech32=bc1q..?bech32m=bc1p..?lightning=<bolt11>` which almost guarantees that any sender who supports BIP21 would be able to successfully make a payment. 

## BIP21 (in the wild)
In practice, however, BIP21 is implemented and used in a non-spec compliant way, namely:

* URIs are used as static personal identifiers, leading to address reuse and loss of privacy
* URIs use *any* on-chain address for the root, e.g. `bitcoin:bc1pxxx`
* URIs are used with the root omitted, e.g. `bitcoin:?b12=<>`
* Extensions for new address types and new payment protocols were never defined

This means if I were to use a URI of the form `bitcoin:bc1p..`, the payment will fail if the sender doesn't understand taproot and there is no way for me to specify a URI of the form `bitcoin:1..?bech32m=bc1p..`. 

As such, its not possible to use BIP21 in a fully backwards compatible way. Furthermore, there have been a number of improvements in address formats since BIP21 was written, namely:

* `bech32m` encoded addresses have a self-identifying HRP, a versioning scheme, and are forwards compatible
* New protocols for static identifiers are being built:
  * BIP352: an on-chain address^[While technically not an encoding of a scriptPubKey, a silent payment address is a `bech32m` encoded ECDH-based address generating a taproot scriptPubKey(s) for the receiver. Conceptually, this is different than something like a `?pj=` extension or a `?lightning=` extension which represent different bitcoin payment protocols.] that is safe to post in public without loss of privacy and without on-chain address reuse
  * BOLT12: a static invoice that can be used non-interactively

HRPs are functionally the same as an extension key and new proposals for non-interactive, private payments allow BIP21 URIs to safely be used statically as a personal identifier.

## A proposed solution

We could define extensions for existing address types so that URIs of the form `bitcoin:1..?bech32m=bc1p..` are possible. But this seems less ideal now given that newer address types already have an HRP and its not clear to me where this should be documented. This also means wallets would need to update with support for each new extension keys (despite already supporting the address types). Each new address format / payment protocol would also need to define an extension key, but learning from history indicates this approach does not work. I think it's more likely we would find ourselves in the exact same position in a few years.

@RubenSomsen has a proposal^[Source: https://github.com/bitcoin/bips/pull/1458#issuecomment-1969380724] which I think is more aligned with how BIP21 is used today and more future proof:

> While I agree that adding a new field for every possible address type is one way to solve this, the much more straightforward way to go about this is to extend BIP21 to allow for more than one address field. E.g. `bitcoin:[address1]?option2=[address2]&option3=[address3]` etc.
>
> The advantage of this approach is that there's less of a chance of implementation splintering (e.g. some implementations may only recognize `bitcoin:[SP address]` and others `sp=[SP address]` - the exact opposite of what a standard like BIP21 is meant to prevent) and newly introduced address types won't require a new field, so BIPs like this one won't need to be concerned with how they're going to fit into BIP21.

With this proposal, you specify the address type you would like to receive to at the root of the URI, and provide a number of fall back options (if desired) for backwards compatibility, e.g. `bitcoin:sp1qxxx?option1=bc1pxxx&option2=bc1qxxx&option3=175xxx?`. This doesn't require existing BIPs to change anything and would work with any new address format out of the box, making BIP21 forward compatible with any address format proposals.

I think there are advantages to keeping this new "fallback option" scoped to only on-chain addresses, but you could extend this to any self-describing payment instructions, where the root of the URI indicates the receivers preference, and fall backs are specified as `options`, e.g. `bitcoin:<bolt12>?option1=sp1q..?option2=bc1p..?option3=bc1q..`. The benefit here is BIP21 would be forward compatible without requiring each new protocol to specify an extension. Not specifying an extension could happen for a number of reasons:

* They forget/are unaware of BIP21
* They don't care/feel its out of scope for their BIP
* They don't agree with BIP21 usage for their protocol

Allowing BIP21 users to use specify payment protocols without putting the burden on each individual protocol to define a BIP21 extension alleviates the concern of having history repeat itself with new extension keys not being defined.

Curious to hear what others think and happy to help collaborate on a BIP21 update, if folks agree that's the best way forward.

-------------------------

josibake | 2024-03-01 14:35:17 UTC | #2

Reading the category descriptions a bit closer, I realized this is probably better suited for the #implementation category

-------------------------

MattCorallo | 2024-03-01 14:47:51 UTC | #3

At a high level I agree that BIP21 should be rewritten to just allow stuff in the parameters and no body, but we should simply require the type of the address as the key - ie if its a taproot address the key should be ta or tp, silent payment sp, BOLT12 b12, etc. Those names are shorter but also allow wallets to know where to go look for things they know how to pay, and also know what kind of thing they're dealing with (if, for example, someone came up with an address type that doesn't identify itself in a bech32 HRP to keep QR code sizes small, not that anyone should).

-------------------------

RubenSomsen | 2024-03-01 14:55:29 UTC | #4

[quote="josibake, post:2, topic:630, full:true"]
Reading the category descriptions a bit closer, I realized this is probably better suited for the #implementation::category category
[/quote]

Moved from protocol design to implementation

-------------------------

RubenSomsen | 2024-03-01 15:29:08 UTC | #5

If we were to go with `bitcoin:[address1]?option2=[address2]&option3=[address3]` then there's an argument to be made that the order of preference should be `address2`, `address3`, `address1` (i.e. `address1` is always last) because `address1` is necessarily going to be the least modern and most backwards compatible address format, and thus likely least desirable.

Similarly, an updated BIP21 should address the question of which address has priority when off-chain addresses have their own custom parameters. E.g. if we take `bitcoin:[address1]?option2=[address2]&b12=[b12_address]` how do you signify that `b12_address` takes precedence? You could put it before `address2` but now you've introduced order dependence at the URI level.

Perhaps this is an argument for not making the distinction between on-chain and off-chain addresses, but we'd still need to support the old parameters for backwards compatibility reasons so we'd probably still need to come up with some kind of deterministic order of preference.

-------------------------

josibake | 2024-03-01 15:32:57 UTC | #6

[quote="MattCorallo, post:3, topic:630"]
we should simply require the type of the address as the key - ie if its a taproot address the key should be ta or tp, silent payment sp, BOLT12 b12, etc. Those names are shorter but also allow wallets to know where to go look for things they know how to pay, and also know what kind of thing they’re dealing with (if, for example, someone came up with an address type that doesn’t identify itself in a bech32 HRP to keep QR code sizes small, not that anyone should).
[/quote]

This doesn't address the concern of needing to retroactively define keys for the known address types and still requiring new BIPs to define a key to be usable with BIP21.

But forgetting the new keys concern for a second, we could define a new generic `addr` key for BIP21 which is only usable with known address types (legacy, p2sh, segwit, taproot), or only usable with addresses that have an HRP (bech32, bech32m) since the HRP is functionally equivalent to an extension key. This means existing address types just work and any new address type with an HRP would also just work. We would still need new payment protocols (`b12` for example) to define their own extension keys, but this seems strictly better than needing to define keys for `legacy`, `p2sh`, `bech32`, `bech32m/tr`, `bech32m/sp`, etc. Sure, the sending client would still need to check each `addr` option, but this seems trivial. If we defined `addr` as only allowing addresses with an HRP, its functionally the same since checking for an HRP is the same as checking for an extension key.

-------------------------

