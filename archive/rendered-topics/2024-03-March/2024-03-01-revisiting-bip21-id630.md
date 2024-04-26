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

josibake | 2024-03-01 15:48:10 UTC | #6

[quote="MattCorallo, post:3, topic:630"]
we should simply require the type of the address as the key - ie if its a taproot address the key should be ta or tp, silent payment sp, BOLT12 b12, etc. Those names are shorter but also allow wallets to know where to go look for things they know how to pay, and also know what kind of thing they’re dealing with (if, for example, someone came up with an address type that doesn’t identify itself in a bech32 HRP to keep QR code sizes small, not that anyone should).
[/quote]

This doesn't address the concern of needing to retroactively define keys for the known address types and still requiring new BIPs to define a key to be usable with BIP21.

But forgetting the new keys concern for a second, we could define a new generic `addr` key for BIP21 which is only usable with self-identifying address types (legacy, p2sh, segwit, taproot, any new address with an HRP), or only usable with addresses that have an HRP (bech32, bech32m) since the HRP is functionally equivalent to an extension key. This means existing address types just work and any new address type with an HRP would also just work. We would still need new payment protocols (`b12` for example) to define their own extension keys, but this seems strictly better than needing to define keys for `legacy`, `p2sh`, `bech32`, `bech32m/tr`, `bech32m/sp`, etc. Sure, the sending client would still need to check each `addr` option, but this seems trivial. If we defined `addr` as only allowing addresses with an HRP, its functionally the same since checking for an HRP is the same as checking for an extension key.

-------------------------

josibake | 2024-03-01 16:03:29 UTC | #7

[quote="josibake, post:6, topic:630"]
only usable with addresses that have an HRP (bech32, bech32m)
[/quote]

The more I think about it, this seems like a nice middle ground: update BIP21 to allow anything that already self identifies with an HRP to be included as a parameter. If it doesn't have an HRP or if the HRP doesn't indicate the protocol being used (payjoin, for example), it needs an extension key. This would allow URIs of the form:

* `bitcoin:sp1q..?lno...`
* `bitcoin:bc1p..?lnbc..`
* `bitcoin:sp1q..?bc1p..`
* `bitcoin:bc1p..?pj=<>`

The really nice thing here is we don't need to define a new `addr` key and wallets that support address type X already know how to check for an HRP.

-------------------------

MattCorallo | 2024-03-01 17:16:26 UTC | #8

[quote="RubenSomsen, post:5, topic:630, full:true"]
Similarly, an updated BIP21 should address the question of which address has priority when off-chain addresses have their own custom parameters. E.g. if we take `bitcoin:[address1]?option2=[address2]&b12=[b12_address]` how do you signify that `b12_address` takes precedence? You could put it before `address2` but now you’ve introduced order dependence at the URI level.

Perhaps this is an argument for not making the distinction between on-chain and off-chain addresses, but we’d still need to support the old parameters for backwards compatibility reasons so we’d probably still need to come up with some kind of deterministic order of preference.
[/quote]

I don't believe the recipient should decide the payment instructions they wish to use. The URI should simply list all the payment instructions the recipient is willing to accept and the sender (who nearly always shoulders the fees) should pick the one they prefer. I don't se why there should be any distinction between on-chain and off-chain payment instructions.

> This doesn’t address the concern of needing to retroactively define keys for the known address types and still requiring new BIPs to define a key to be usable with BIP21.

This is a fair point, however I think we can simply mark them as "existing ones" and leave them in the URI body rather than in query parameters. There are already many implementations that assume they're there/place them there, and the ship has kinda sailed on changing that. You could make an argument that we should add an option to put taproot/bech32m instructions in the query parameters to let folks offer both Segwit/bech32 and taproot/bech32m instructions in the same URI, but I don't really see a super compelling use for that, and I think its just too late.

> But forgetting the new keys concern for a second, we could define a new generic `addr` key for BIP21 which is only usable with self-identifying address types (legacy, p2sh, segwit, taproot, any new address with an HRP), or only usable with addresses that have an HRP (bech32, bech32m) since the HRP is functionally equivalent to an extension key. This means existing address types just work and any new address type with an HRP would also just work. We would still need new payment protocols (`b12` for example) to define their own extension keys, but this seems strictly better than needing to define keys for `legacy`, `p2sh`, `bech32`, `bech32m/tr`, `bech32m/sp`, etc. Sure, the sending client would still need to check each `addr` option, but this seems trivial. If we defined `addr` as only allowing addresses with an HRP, its functionally the same since checking for an HRP is the same as checking for an extension key.

I'm not sure what the advantage of `addr` is over simply reusing the HRP - it just seems less descriptive for no reason. I went ahead and wrote up the concrete set of changes I think make sense at https://github.com/bitcoin/bips/pull/1555

> The more I think about it, this seems like a nice middle ground: update BIP21 to allow anything that already self identifies with an HRP to be included as a parameter. If it doesn’t have an HRP or if the HRP doesn’t indicate the protocol being used (payjoin, for example), it needs an extension key. This would allow URIs of the form:

This doesn't allow, for example, offering both Silent Payment instructions as well as Lightning.

-------------------------

josibake | 2024-03-01 17:30:12 UTC | #10

[quote="MattCorallo, post:8, topic:630"]
This doesn’t allow, for example, offering both Silent Payment instructions as well as Lightning.
[/quote]

Yes it does. Silent payment addresses, lightning invoices, and lightning offers all have an HRP (`sp1`, `lnbc1`, `lno1`). I included this in my examples.

-------------------------

MattCorallo | 2024-03-01 17:41:34 UTC | #11

> Yes it does. Silent payment addresses, lightning invoices, and lightning offers all have an HRP (`sp1`, `lnbc1`, `lno1`). I included this in my examples.

Ah, apologies, I misunderstood the point here to have one parameter. Indeed, we could, however that may break existing implementations. BIP 21 as written would allow this, but not sure if any existing implementations try to parse parameters as K/V and fail if there's no =. Its nice to avoid a few extra characters in QR codes, but not sure its worth that risk.

-------------------------

josibake | 2024-03-01 18:05:00 UTC | #12

[quote="MattCorallo, post:11, topic:630"]
but not sure if any existing implementations try to parse parameters as K/V and fail if there’s no =
[/quote]

If they are requiring an `=` they aren't spec compliant? :) But agree, we don't want to break anything. This is pretty easy to verify, though, since we have a good list of implementations listed at https://bitcoinqr.dev/ which can be used to verify, and this is a somewhat trivial fix if they are requiring an `=`.

This seems less risky then specifying `bitcoin:?key=val`, which seems *more* likely to break existing implementations since a spec compliant implementation *would* expect an address in the root of the URI.

Furthermore, this can be used along side existing `key=val` parameter pairs, so you can even start using the new technique along with the old way in a backwards compatible way. Seems like a no brainer to me.

-------------------------

MattCorallo | 2024-03-01 18:35:38 UTC | #13

> If they are requiring an `=` they aren’t spec compliant? :slight_smile: But agree, we don’t want to break anything. This is pretty easy to verify, though, since we have a good list of implementations listed at https://bitcoinqr.dev/ which can be used to verify, and this is a somewhat trivial fix if they are requiring an `=`.

Yea, I mean if someone wants to do the legwork I'd be happy to see the no-KV version, just not sure its worth someone's time to do that :).

> This seems less risky then specifying `bitcoin:?key=val`, which seems *more* likely to break existing implementations since a spec compliant implementation *would* expect an address in the root of the URI.

The point of this is that it is *only* done for new key types which are already, today, unsupported. So the options are bitcoin:newaddressformat or bitcoin:?k=newaddressformat. The same set of wallets is broken by both, at least as long as those adding support for newaddressformat support the empty-body version as a part of rolling out the upgrade. Luckily this is a bit easier to test - simply provide test cases for both in the BIP defining newaddressformat.

Further, it seems marginally less likely to break for us to go from bitcoin:oldaddressformat?k=newaddressformat to bitcoin:?k=newaddressformat rather than bitcoin:newaddressformat, not to mention its just nice to only have one format/place to look for newaddressformat. Though, the same "people should test this as a part of the rollout" argument applies, of course.

-------------------------

josibake | 2024-03-02 10:49:42 UTC | #14

[quote="MattCorallo, post:13, topic:630"]
Yea, I mean if someone wants to do the legwork I’d be happy to see the no-KV version, just not sure its worth someone’s time to do that :).
[/quote]

You can also do this in a way that ensures you won't break existing implementations: just include an optional `=v` dummy value at the end. Still smaller than the kv approach and after a long enough period has passed where we are sure clients are using the new logic, receivers can drop the dummy values to save even more space in the QR code.

So for a new client, something like "MAY include the dummy value `=v` at the end of the address. This is to ensure backwards compatibility with older clients" and for the sender, "MUST check for the value `=v` at the end of the address and remove it. This value is used to ensure backwards compatibility with older clients."

[quote="MattCorallo, post:13, topic:630"]
not to mention its just nice to only have one format/place to look for newaddressformat
[/quote]

This is another reason to prefer the no-KV approach: there is no more ambiguity around putting a new address in the root vs putting it in a kv pair. Senders just need to split the URI on `?` and look for their preferred HRP/`extensionkey`. This is also nice for new clients supporting the new address type: all they need to do is be able to recognize and parse the new address type, so no extra test cases needed for checking the root for the new address vs checking the `kv` pairs for the new address.

-------------------------

john | 2024-03-04 22:09:34 UTC | #15

I'm not sure if this is the right place to bring this up, but I have been hoping to see BIP21 expanded to allow split payments, for example:

`bitcoin:address=3FfBeGA1brVESSUeA6RBZYeiNHJmFWnWQu&amount=0.123&label=Payment_A&address=3BA3YuxeY8bsaK14DUY1f4X7WhqmJesUda&amount=0.000123&label=Fee_A`

Granted, this involves wallets upgrading to offer such payments. But I thought I would bring it up here while param structure/nomenclature for BIP21 is being discussed.

For reference: https://x.com/john_zaprite/status/1506112407990677507?s=20

-------------------------

MattCorallo | 2024-03-05 02:47:02 UTC | #16

[quote="josibake, post:14, topic:630"]
You can also do this in a way that ensures you won’t break existing implementations: just include an optional `=v` dummy value at the end. Still smaller than the kv approach and after a long enough period has passed where we are sure clients are using the new logic, receivers can drop the dummy values to save even more space in the QR code.
[/quote]

Mmm, fair point, though now we're saving two chars to avoid a K/V pair? I'm not really sure its worth it, and if at some point we move on from bech32m-based addresses or something that has a less-visible HRP it avoids needing to parse the whole blob. Not like any of this matters all that much, though, really, might as well flip a coin at this point.

> This is another reason to prefer the no-KV approach: there is no more ambiguity around putting a new address in the root vs putting it in a kv pair. Senders just need to split the URI on `?` and look for their preferred HRP/`extensionkey`. This is also nice for new clients supporting the new address type: all they need to do is be able to recognize and parse the new address type, so no extra test cases needed for checking the root for the new address vs checking the `kv` pairs for the new address.

Hmm? You still have to split on &s to separate the various addresses, as well as parse K-V pairs for other parameters (like comments, amounts, lightning, etc), so you can't avoid any of that complexity no matter what. I also want to highlight here that we *really* shouldn't be assuming that we'll always and forever use bech32(m) for any new address type, so we don't want to bake that in as a super deep assumption (though doing something special for bech32(m) is kinda maybe reasonable?).

> I’m not sure if this is the right place to bring this up, but I have been hoping to see BIP21 expanded to allow split payments, for example:

This seems like a pretty separate conversation, and also one that is going to break compatibility with all existing wallets :/. Not sure how to go about such a large-scale change.

-------------------------

john | 2024-03-05 07:46:30 UTC | #17

> This seems like a pretty separate conversation, and also one that is going to break compatibility with all existing wallets :/. Not sure how to go about such a large-scale change.

Understood. Yes, it’s definitely a much larger conversation.

-------------------------

josibake | 2024-03-05 13:17:49 UTC | #18

[quote="MattCorallo, post:16, topic:630"]
Mmm, fair point, though now we’re saving two chars to avoid a K/V pair? I’m not really sure its worth it, and if at some point we move on from bech32m-based addresses or something that has a less-visible HRP it avoids needing to parse the whole blob
[/quote]

I'm not sure where you're getting two chars from? My point was that bech32(m) addresses _already_ are a key-value pair, i.e. `HRP (key) 1 (=) <data> (value)`, so requiring a key for them is redundant and creates more work for wallets since they now need two ways of recognizing the address: one for normal use, and one for identifying the BIP21 specific key. It's a nice side effect that we can reduce the QR code size, but that's not the main benefit.


[quote="MattCorallo, post:16, topic:630"]
Hmm? You still have to split on &s to separate the various addresses, as well as parse K-V pairs for other parameters (like comments, amounts, lightning, etc), so you can’t avoid any of that complexity no matter what.
[/quote]

My point was about removing the ambiguity about what goes in the root vs what goes in the keys. Parsing the URI simplifies to "look for your preferred HRP or KV protocol," with all the other parsing remaining the same.

[quote="MattCorallo, post:16, topic:630"]
I also want to highlight here that we *really* shouldn’t be assuming that we’ll always and forever use bech32(m) for any new address type, so we don’t want to bake that in as a super deep assumption (though doing something special for bech32(m) is kinda maybe reasonable?).
[/quote]

I'm not. My assumption is that it is the best option for the foreseeable future (and already standard across bitcoin and lightning). It also allows us to address the current problem of existing address types that *are* bech32(m) encoded and *don't* have a BIP21 extension key in a simple and efficient manner. We also automatically support any new address type that is bech32(m) encoded. For everything else we can keep using key-value pairs.

-------------------------

MattCorallo | 2024-03-07 15:02:09 UTC | #19

[quote="josibake, post:18, topic:630"]
I’m not sure where you’re getting two chars from?
[/quote]

I was comparing hrp=... to ...=, which saves only a few chars, depending on the length of the hrp (two in the case of silent payments).

[quote="josibake, post:18, topic:630"]
My point was about removing the ambiguity about what goes in the root vs what goes in the keys. Parsing the URI simplifies to “look for your preferred HRP or KV protocol,” with all the other parsing remaining the same.
[/quote]

I don't see how "what goes in the root vs what goes in the keys/values" is relevant? That is addressed with the new (suggested) wording in the BIP 21 change PR: "Future address formats SHOULD instead be placed in query keys as optional payment instructions to provide backwards compatibility during upgrade cycles. After new addres types are near-universally supported, or for recipients wishing to avoid a standard on-chain fallback, the bitcoinaddress part of the URI MAY be left empty."

There's explicitly only one place to look for any given address type given that wording, never two. Whether its a K/V location or just a URI parameter with no K/V doesn't impact that.

[quote="josibake, post:18, topic:630"]
I’m not. My assumption is that it is the best option for the foreseeable future (and already standard across bitcoin and lightning). It also allows us to address the current problem of existing address types that *are* bech32(m) encoded and *don’t* have a BIP21 extension key in a simple and efficient manner. We also automatically support any new address type that is bech32(m) encoded. For everything else we can keep using key-value pairs.
[/quote]

I don't believe there are any address types which are not "either a base64 P2SH or P2PKH address, bech32 Segwit version 0 address, bech32m Segwit address" *and* don't have an existing BIP21 extension key. The ship has sailed for BOLT 11 lightning payments, those will always be K/V (with the "lightning" key), we can't practically ever change that. The only question is what to do for Silent Payments and BOLT12. We could do K/V or not K/V, it doesn't matter *that* much, but K/V fits a bit nicer into existing code that parses all the URI parameters into K/V pairs (cause there is currently nothing that uses BIP 21 without all parameters being K/V pairs, AFAIK), but wastes a few bytes for it.

-------------------------

juscamarena | 2024-04-26 20:36:49 UTC | #20

I agree with Ruben here that address1 should be whatever the receiver is comfortable will be compatible and cheaper for them to accept.  I previously worked at Bitrefill building a bit part of the payment experience, we moved to using segwit bec32 addresses and have not seen any issues and wouldn't necessarily want to default to a more expensive but more backward compatible address.

One thing we would have liked would be to to be able to set a taproot address so wallets that do support it can opt in to send to that instead.

-------------------------

