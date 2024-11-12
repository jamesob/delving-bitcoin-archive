# Expanding on BOLT12

andyschroder | 2024-09-29 04:18:33 UTC | #1

## Summary

Now that BOLT12 has been merged (https://github.com/lightning/bolts/pull/798) and we are starting to get some implementations out in the wild, I'm wondering where the discussion is on further improvements of it? Should certain extensions be added to BLIPs and certain changes be made to the BOLT? Is there an active discussion forum for BOLT12 somewhere else besides this site?

## Other Known Proposed Changes

One thing I've noticed in BOLT12 is that an `invoice_request` writer must copy all fields from the `offer` (including unknown fields) and an `invoice` writer must copy all non-signature fields from the `invoice_request` (including unknown fields). This seems like we have some flexibility to in a potentially backwards compatible way add more fields as desired?

So far, I've found a mention of a new `user` field in the `invoice_request` that has been suggested (https://github.com/lightning/blips/blob/b6c3e8c17028926f7c5ae254f419456fe3c4bf13/blip-0032.md?plain=1#L86).

Is there anything else besides `user` that has already been proposed to add to BOLT12? I have some other things I'd like to bring up:


## Automatic Refunds

I'd like to propose for `offer` writers is an optional boolean field of `refund_invoice_required` and if exists and set to true, `invoice_request` writers are required to include a `refund_invoice` field with an encoded (https://github.com/lightning/bolts/blob/247e83d528a2a380e533e89f31918d7b0ce6a0c1/12-offer-encoding.md?plain=1#L73) refund invoice included in it (there seems to be no human-readable prefix defined for invoices in BOLT12, so I'm proposing the prefix `lni`). If the `invoice_request` writer does not include the `refund_invoice` field, the `offer` issuer does not necessarily have to issue an `invoice`.

The logic behind these proposed fields is that a merchant may want to implement (and potentially require) an automatic refund workflow without needing to go through the "merchant-pays-user flow" QR/NFC/link cycle with the other party. Potentially, the point of sale device only has a static offer printed on a sticker and no screen at all to provide an `invoice_request` for a refund.

Alternatively, rather than providing an encoded refund `invoice` in the original `invoice_request`, an encoded refund `invoice_request` could be included in the `invoice` instead and the refund `invoice` could be sent when buyer wants their refund issued.



## Maximum `invoice_request` amount

We also have `offer_amount` as the minimum amount required for successful payment (https://github.com/lightning/bolts/blob/247e83d528a2a380e533e89f31918d7b0ce6a0c1/12-offer-encoding.md?plain=1#L241). There are use cases where we might want to enforce a maximum amount. For that, I'd like to propose `offer_max_amount` in the `offer`. A few examples for why I think this could be useful:

1. You have a variable amount of product to sell, but you don't want to commit to selling more than you actually have.
2. You know you don't have enough inbound liquidity to sell beyond a certain quantity of product.
3. You are using HOLD invoices and don't want them to be too large. If the buyer wants more product, they will need to make multiple payments instead.
4. You are not sure if you like your customer yet and you don't want to "imply" that you are committing to some large business deal with them by accepting much more from them without having further interactions to determine they are a good customer.
5. You want to accept only an exact amount in order for the customer to communicate a clear and finite scope to the business deal.

This maximum amount could be enforced by the offerer, they could just not provide an invoice if the invoice_request is for an amount that is too high for them, but it would be good to be able to communicate this in advance. This still might not be totally enforcable since HTLC acceptance just requires a minimum be met (I don't think the HTLC enforce a maximum amount), but it could at least avoid many situations where the merchant may want to discourage accepting more than a certain amount.




## `invoice_request` expiration

I'm wondering about with BOLT12, why is there no expiry in an `invoice_request` like we have on `offer` and `invoice`? With the merchant-pays-user flow, the `invoice_request` seems like it should be able to expire after a certain point. Just like an invoice can be canceled, surely, this can be enforced on the node end (just don't pay an invoice if received), but it seems like there should be a way to communicate this in advance to the other party that may send the invoice. Also, if an offerer is too slow to respond with an `invoice` in response to an `invoice_request`, might we want to let them know don't even bother sending me a `invoice` anymore.

## Conclusion

I look forward to your feedback.

-------------------------

andyschroder | 2024-09-30 06:32:47 UTC | #2

Here's another proposal that I missed. This one suggests it should be a bLIP, not a revision of the BOLT. That makes sense to me since this proposal is optional and BOLT12 should work normally if the user doesn't have/want this proposed feature.

https://delvingbitcoin.org/t/bolt-12-trusted-contacts/1046

-------------------------

andyschroder | 2024-10-05 15:57:36 UTC | #3

[quote="andyschroder, post:1, topic:1167"]
there seems to be no human-readable prefix defined for invoices in BOLT12, so I’m proposing the prefix `lni`
[/quote]

Despite not being defined in the spec, it looks like CLN already uses the `lni` prefix for their fetchinvoice (https://docs.corelightning.org/reference/lightning-fetchinvoice) and pay (https://docs.corelightning.org/reference/lightning-pay) RPC commands.

-------------------------

accumulator | 2024-10-16 09:00:18 UTC | #4

I want to mention one other idea that would be useful to have in the BOLT12 spec.

**Bundled Payments**

The gist is to have an invoice with two distinct preimages and amounts in a single invoice. 

The use case is for services that require the prepayment of a mining fee in order for a non-custodian exchange to take place:
  - Submarine swaps
  - JIT channels

A more detailed description of the proposal can be found here : 
- https://lists.linuxfoundation.org/pipermail/lightning-dev/2023-June/003977.html
- https://lists.linuxfoundation.org/pipermail/lightning-dev/2023-June/003990.html

-------------------------

andyschroder | 2024-11-12 00:21:51 UTC | #5

Another thing to note about refunds. BIP70 had a refund field. See https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki#payment . It worked quite well for things like [The Bitcoin Fluid Dispenser](http://andyschroder.com/BitcoinFluidDispenser/2.3/) .

-------------------------

