# bLIP: BOLT 11 Invoice Blinded Path Tagged Field

ellemouton | 2024-06-26 00:10:57 UTC | #1

Hey y'all! 

Want to generate some discussion around this new bLIP proposal before opening a more formal PR to the bLIP repo. See the " Some Questions, Discussion Points & TODO" section at the end for some specific questions & discussion points from my side. Thanks in advance for any feedback!

```
bLIP: xx 
Title: BOLT 11 Invoice Blinded Path Tagged Field
Author: Elle Mouton <elle.mouton@gmail.com>
Status: Draft
Created: 2024-06-20
* Post-History: dates of postings to lightning-dev mailing list, or link to thread in 
  mailing list archive
License: CC0
```

## Abstract

This bLIP defines a new [tagged field][tagged-fields] for the payment invoice
encoding described by [BOLT 11][bolt11] which can be used to communicate 
encoded blinded path information to the payer of the invoice. 

## Copyright

This bLIP is licensed under the CC0 license.

## Rational

Blinded paths have been included in the [BOLT 4 specification][blinded-paths] 
and using them in the context of receiving payments is natively supported in 
the [BOLT 12 Offers proposal][offers]. However, the Offers proposal also 
includes various other dependencies such as onion messaging and various new 
protocol message all of which will require a network wide upgrade before full 
advantage can be taken of the new protocol. Blinded paths themselves are a 
useful tool for privacy and potentially more reliable payment delivery and so
this document proposes a carve-out to the existing [BOLT 11][bolt11] invoice
format so that advantage can be taken of the blinded paths protocol in the
context of payments in a sooner time frame. This will be done by adding a new 
tagged-field type to the BOLT 11 invoice specification that will encode a 
blinded payment path. With this bLIP, the sender and the receiver of a payment 
will need to be aware of the new tagged-field and the receiver will need to 
support route blinding and ideally have direct channel peers who also support 
route blinding in order to start widening their anonymity set.

## Specification

### Feature Bit

A new feature bit, `bolt11_blinded_path`, for the `I` context will be added 
using bits from the experimental range. This is required because the BOLT 11 
tagged-fields pre-date the TLV format and so nodes parsing the invoice will just
skip fields they don't know as there is no concept of "It's OK to be odd". This
feature bit thus allows nodes to fail fast if they do not yet understand the new
tagged field.

### Tagged Field

The proposal is to add the following new [tagged field][tagged-fields] to the 
set defined in [BOLT 11][bolt11]:

- `b` (20): `data_length` variable. One or more entries each containing a 
   blinded payment path for a private route; there may be more than one `b` 
   field. The field uses the `blinded_payinfo` type described below which draws 
   heavily on the proposed encoding of the `blinded_payinfo` subtype defined in 
   the [Offers proposal][offers].

1. subtype: `blinded_payinfo`
2. data:
  * [`u32`:`fee_base_msat`]
  * [`u32`:`fee_proportional_millionths`]
  * [`u16`:`cltv_expiry_delta`]
  * [`u64`:`htlc_minimum_msat`]
  * [`u64`:`htlc_maximum_msat`]
  * [`u16`:`flen`]
  * [`flen*byte`:`features`]
  * [`33*byte`:`first_ephemeral_blinding_point`]
  * [`byte`:`num_hops`]
  * [`num_hops*blinded_hop`:`blinded_hops`]

1. subtype: `blinded_hop`
2. data:
   * [`33*byte`:`blinded_node_pubkey`]
   * [`bigsize`: `cipher_text_length`]
   * [`cipher_text_length*byte`:`cipher_text`]

The `blinded_node_pubkey` of the first `blinded_hop` in a `blinded_payinfo` is
the real public key of the blinded path's introduction node. 

** Note: see discussion section for question around communicating 
`max_cltv_expiry` **

### Requirements

An invoice containing the `b` field type:
- MUST not contain the `r` field type.
- MUST not contain the `s` field type since a payment address in the context of
  blinded payments does not make sense since the recipient is able to use the
  `path_id` in the `encrypted_recipient_data` for the same purpose.
- SHOULD sign the invoice with a private key that is not the same as their 
  public node ID and should not set the destination node (`n` field).
- Each `blinded_path` must fit within the `data_length` size limit. This places
  an upper limit of approximately 7 `blinded_hops` on each path. See the 
  appendix for the estimation calculation.
- If the invoice will be displayed in QR form, then this also places an upper
  limit on the number of `blinded_path` fields that can be added to the 
  invoice.

## Universality

This proposal is a temporary measure that will allow users to start making use
of blinded paths in the context of payments and thereby taking advantage of the 
potential privacy and payment success rate benefits that in they will in theory 
provide. Once the Offers protocol along with its new invoice format has been 
widely deployed, then there will be no use for this BOLT 11 carve-out. Due to 
the forcasted temporary use of the new field, it makes sense to be in bLIP form
rather than adding this in a more temporary way to the spec via a BOLT update 
proposal.

## Backwards Compatibility

BOLT 11 states that the reader of an invoice "MUST skip over unknown fields". 
This means that an un-updated reader of an invoice that includes the new tagged 
field would skip it when parsing the invoice. The proposal also adds a new 
feature bit to the invoice feature bit vector and so this gives nodes an 
indication that the invoice includes something they do not yet understand. 

## Reference Implementations

The proposed encoding of the new BOLT 11 tagged-field is added to the LND 
implementation in [this PR][impl].

## Appendix

### Example invoice string

The following string is an example of a BOLT11 invoice containing 2 blinded 
paths, one with 1 hop and one with 3 hops. Note that this uses the 
79 approximation for the cipher text calculated in the "Size Restrictions" 
calculation further along in the appendix. Without the hrp part, this is 1153 
bytes which is still under the QR-code limit. 

```
lnbc241pveeq09pp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdqq55zqqqqq2qqqqqpgqyzqqqqqqqqqqqqyqqqqqqqqqqqvsqqqqlnxy0ffrlt2y2jgtzw89kgr3zg4dlwtlfycn3yuek8x5eucnuchqps82xf0m2u6sx5wnjw7xxgnxz5kf09quqsv5zvkgj7d5kpzttp4qz7qp8qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqgszjwfxyxwcalnnxvmw9dn52uxaj648vwktx4jwdcm8kwzdscdq5qzwqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqk8t6endgpc99824amqzk9japgu8synwf3wx4qp4ej2r0h8rghypsqyuqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqq5gcqqqqqpqqqqqqyqq2qqqqqqqqqqqqqqqqqqqqqqqqpgqqqqk8t6endgpc99824amqzk9japgu8synwf3wx4qp4ej2r0h8rghypsqsygpf8ynzr8vwleenxdhzke69wrwed2nk8t9n2e8xudnm8pxcvxs2qp8qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqnp4q0n326hr8v9zprg8gsvezcch06gfaqqhde2aj730yg0durunfhv667625ha7mj2grjtpfqhzdwdp5ej26ff4zk7lth0e7caprk0pzd49jj5jg89e5ppny8e6lmhv4ak3wvr76z5d66qxdseda3jeauz8fmmsqq2gke3
```

### Size Restrictions

#### `data_length` Limit

In order to conform to any existing BOLT 11 invoice parser, each new tagged
field must use the `data_length` encoding defined there. This means that the
maximum size of any _single_ encoded blinded path is 639 bytes.

What follows is a rough estimation of the maximum number of hops we can include
in a single blinded path. It assumes that each hop's cipher text is the same
length.

##### Cipher Text Size Estimation

First, a rough estimation of the average cipher text length is required. A
forwarding node in a blinded path will receive a cipher text payload containing
the following data:

- `padding`: optional
- `short_channel_id` of 8 bytes
- `payment_relay`:
    * 2 byte `cltv_expiry_delta`
    * 4 byte `fee_proportional_millionths`
    * 4 byte `fee_base_msat`
- `payment_constraints`:
    * 4 byte `max_cltv_expiry`
    * 8 byte `htlc_minimum_msat`
- `allowed_features`: optional

If we [assume that the `allowed_features` vector is not
set][empty-allowed-features], then this comes to a total of 30 mandatory bytes.

For the recipient node, it will receive a cipher text payload containing:

- `padding`: optional
- `path_id`: let's assume that this is 32 bytes like the existing payment
  address.
- `payment_constraints`:
    * 4 byte `max_cltv_expiry`
    * 8 byte `htlc_minimum_msat`

This comes to a total of 44 bytes for the recipient's cipher text.

The padding field should be used by recipients to pad cipher text blobs so that
all are the same size. Since the calculated recipient cipher text blob size (44)
is larger than that of the forwarding nodes (30), we can assume that all the
cipher text blobs will have a size of around 44 bytes.

##### `blinded_hop` Size Estimation

The total number of bytes required for a single `blinded_hop` is:

    = 33+bigsize_len(cipher_text)+len(cipher_text)

If we use the estimated `cipher_text` size of 44 bytes, then 
`bigsize_len(cipher_text)` is 1 and so this comes to 78 bytes for a single 
`blinded_hop`.

##### `blinded_payinfo` Size Estimation

The total number of bytes required for the encoding of a single
`blinded_payinfo` entry is:

    = 4+4+2+8+8+2+len(features)+33+1+(num_hops*len(blinded_hop))
    = 68+len(features)+(num_hops*len(blinded_hop))

If we take the estimate of 78 bytes per `blinded_hop` and if we assume an empty
feature vector then this comes to:

    = 68+(num_hops*78)

The maximum number of hops in a single blinded path can then be calculated to
be:

    639 = 68+(num_hops*78)
    num_hops = 7

#### QR code limit

Another soft maximum value to keep in mind is the maximum number of bytes that
can fit into a [QR code][qr] which is 2,953 bytes. This is a soft maximum
because this only applies if the invoice is in fact being transmitted via QR
code. This limit does not apply if the invoice is being transmitted via other
protocols such as LNURL. In the cases where the limit does apply, then two
variables will be at play:

- The number of blinded paths
- The number of blinded hops within each path (which will always also be
  restricted by the `data_length` maximum).

The exact limit on the number of blinded paths that can be included depends on
the size of other fields in the invoice. It is worth noting that an invoice with
a blinded path should not contain any `r` (route hint) fields.


# Some Questions, Discussion Points & TODOs: 

1. In terms of communicating the `max_cltv_expiry` value to the sender, [the 
   route blinding proposal doc][rb-proposal] suggests the possibility of just 
   re-using the existing BOLT 11 expiry field. However, I think it could 
   potentially be better to communicate it more explicitly as mentioned by 
   t-bast [here][max_cltv_expiry] for [Offers][offers]. The question then is, do
   we add _another_ BOLT 11 field for this value where one expiry will apply to 
   all the routes or is there an argument for having an expiry _per_ route?
2. Currently, we swap the first "blinded_node_id" out for the real introduction 
   key as a way to save space (since we don't need the introduction node's 
   blinded key but still want to give it encrypted data). But perhaps this is too
   hacky/confusing, and we should instead just create separate "intro_node" and 
   "intro_node_encrypted_data" fields. The downside to that would be that 
   "num_hops" is then off-by-one.
3. In terms of the invoice signature, currently it doesn't mean much other than 
   "this has not been tampered with since creation". This isn't really worse than
   today for a node that only has connection via hop hints. But potentially 
   worth doing something like an `n`-of-`n` MuSig2 using the `n` destination 
   public keys used for the `n` blinded paths. 
4. Does it make sense to add an experimental tagged field range for BOLT 11 in 
   the bLIP2 spec or would this be over-kill since BOLT 11 is in any-case being 
   phased out?

[tagged-fields]: https://github.com/lightning/bolts/blob/c562d91ace0e95bec3c6f8758969eaf3627f23c8/11-payment-encoding.md#tagged-fields
[bolt11]: https://github.com/lightning/bolts/blob/c562d91ace0e95bec3c6f8758969eaf3627f23c8/11-payment-encoding.md
[blinded-paths]: https://github.com/lightning/bolts/blob/c562d91ace0e95bec3c6f8758969eaf3627f23c8/04-onion-routing.md#route-blinding
[offers]: https://github.com/lightning/bolts/pull/798
[impl]: https://github.com/lightningnetwork/lnd/pull/8752
[qr]: https://en.wikipedia.org/wiki/QR_code#Information_capacity
[lnurl]: https://github.com/lnurl/luds
[rb-proposal]: https://github.com/lightning/bolts/blob/c562d91ace0e95bec3c6f8758969eaf3627f23c8/proposals/route-blinding.md?plain=1#L274
[max_cltv_expiry]: https://github.com/lightning/bolts/pull/798/files#r1053000804
[empty-allowed-features]: https://github.com/lightning/bolts/blob/c562d91ace0e95bec3c6f8758969eaf3627f23c8/04-onion-routing.md?plain=1#L253

-------------------------

t-bast | 2024-06-26 11:28:27 UTC | #2

Since both the payer and recipient would need to add code to support this extension, why not simply allow paying a Bolt 12 invoice (without going through the offer / invoice_request flow)?

This ensures that people who want to leverage this intermediate step only add code that is useful to later support the full Bolt 12 flow instead of changing Bolt 11?

-------------------------

