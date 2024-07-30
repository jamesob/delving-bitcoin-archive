# Bolt 12 Trusted Contacts

t-bast | 2024-07-30 15:12:09 UTC | #1

Bolt 12 makes it possible for lightning wallets to provide a UX that is very close to traditional payment applications. By attaching offers to metadata describing who those offers belong to, users can manage their "contacts list" and easily pay them without any interaction (no need to manually share a Bolt 11 invoice for every payment). This is the kind of payment experience non-technical users expect.

However, when sending money to their friends and family, users expect some form of mutual authentication: when Alice sends money to Bob, she wants Bob to know it came from her.

This may seem at odds with the privacy benefits of Bolt 12, which explicitly hides the identity of both the payer and the recipient. The goal of this post is to create a standard way for payers to selectively reveal their identity when paying Bolt 12 offers.

Such a feature makes a lot more sense if it works seamlessly across different wallets, so it should become a standard.

There are two parts to this feature, which are somewhat orthogonal:

- contact key distribution
- contact mutual authentication

## Contact Key Distribution

Every node that wants to support this feature must generate a `contact_key` that they will use to identify themselves to their trusted contacts. This key can for example be derived from the node's seed using a dedicated BIP32 derivation path.

In order to advertise this key to trusted contacts, I see two options:

- (optionally) include it in offers
- (optionally) include it in BIP21 URIs

Note that users may want to only reveal this key to their trusted contacts over a secure communication channel. Wallets should thus keep creating offers and BIP21 URIs that do NOT include this `contact_key` whenever users want to be paid privately (e.g. for offers posted on social media).

### Option 1: TLV field in offers

We define a new odd TLV field that can be included in Bolt 12 offers:

- type: 23 (`offer_contact_key`)
- data:
  - [`point`:`contact_key`]

I like this option because it is very simple to integrate to existing lightning implementations. It also allows directly adding someone to our contacts list when paying one of their offers (if it contains this field).

One tiny drawback is that users who want to create multiple offers with the same `contact_key` end up duplicating this field in every offer.

@rustyrussell mentioned using the existing `issuer` field, but I think that a dedicated field makes more sense here?

### Option 2: BIP21 query parameter

We define a new optional query parameter that can be included in BIP21 URIs (or rather [Matt's proposed replacement for BIP21](https://github.com/bitcoin/bips/pull/1555)):

```txt
bitcoin:?lno=lno...&lnck=lnck...
```

The key is encoded using bech32m with the `lnck` prefix, for example `lnck1qfuvmn3va9l0c54jtjs4q494n6nqjw3s7jn09gwh6j42v6q3tqtngkrs462`.

This removes the potential duplication across multiple offers from the previous option, but forces wallets to use BIP21 URIs everywhere instead of plain offers.

One drawback of this option is that it creates bigger QR codes.

### Questions

Should we allow this `contact_key` to be rotated, or is it a static identity like your `node_id`? If we rotate it, our contacts won't be able to identify our payments until they've received the new key, which is quite an annoying drawback.

## Contact Mutual Authentication

We now assume that key distribution is solved: users have a list of their trusted contacts. Each trusted contact contains the `contact_key` and a list of Bolt 12 offers that can be used to pay this contact.

When Alice wants to pay Bob (who is one of her trusted contacts), Alice can optionally decide to use her own `contact_key` to reveal her identity to Bob. Bob should only learn that the payment comes from Alice if Alice is also in Bob's contacts list.

There are many ways we can achieve that. I'm going to list three options below, starting from the simplest one.

### Option 1: directly use `invreq_payer_id`

Bolt 12 provides an `invreq_payer_id` field that is used to sign `invoice_request`s.
When paying a trusted contact, we could directly set this to our `contact_key`.
The receiver can then match the `invreq_payer_id` with their own contacts list to identify the payment.

While this is a very simple option, it has a few drawbacks:

- senders will sign `invoice_request`s to unrelated contacts with the same key
- senders will reveal their `contact_key` to recipients who may not have them in their contacts list

The second point may not be an issue though: the sender wanted to identify themselves anyway.

Using the `invreq_payer_id` to match contacts has an interesting benefit: the `invreq_payer_note` can be safely displayed. This field is provided by payers in their `invoice_request`, but they may contain spam or phishing. Once we know a payment comes from one of our contacts, it shouldn't be spam or phishing and can thus be displayed alongside the payment, which provides a nice UX.

### Option 2: derive per-contact `invreq_payer_id`

This option is similar to the previous one, but we derive a different `invreq_payer_id` for each contact:

- Let $`contact\_key = contact\_privkey * G`$
- $`ss = SHA256(contact\_privkey_{payer} * contact\_key_{recipient}) = SHA256(contact\_key_{recipient} * contact\_key_{payer})`$ (ECDH between the two `contact_key`s)
- $`b = SHA256("bolt12\_contact" || ss)`$
- $`invreq\_payer\_id = b * contact\_key`$

This solves the drawbacks from the previous option without adding too much complexity. The `invreq_payer_id` can be derived once and stored when adding the contact to our contacts list. Similarly, we can derive and store the `invreq_payer_id` each contact will use when paying us, to efficiently match incoming payments.

A potentially useful side-effect of this scheme is that if we throw away our `contact_privkey`, past payments we received will appear to be from random nodes. It becomes impossible to match them to the contact that sent us that payment. This can be used as a "panic button" when we don't want our payment history to be revealed.

### Option 3: use bLIP-31

@MattCorallo defined a protocol for mutual message authentication in [bLIP 31](https://github.com/lightning/blips/pull/31). He proposed using this protocol for the mutual authentication part of this feature. The protocol would then become:

- the "initiator" is the payment recipient receiving an `invoice_request`
- in the onion message they send back with their `invoice`, they would include the `init bytes`
- the "message-sender" is the payer receiving an `invoice` containing some `init bytes`
- in the payment onion they create, they would include the `encrypted_nonce`

I'm not convinced this is the right approach for this feature, because:

- it's more complex and it doesn't seem to add anything useful compared to the previous options
- the main goal of this protocol is to send an encrypted message, which isn't what we're trying to do here

I also see two drawbacks when using this for a contacts list feature:

- since onion messages are limited to 65kB, the "initiator" cannot include more than ~1350 keys in their init bytes
  - this means that we cannot have more than 1350 contacts
  - while this seems ok for "normal" people, it is unnecessarily limits what people can build on top of bolt 12
  - if for example twitter included a feature where you could be tipped by your followers, limiting this to 1350 people wouldn't work
- since the payer needs to include a payload for the recipient, it won't work when the sender uses trampoline but the recipient doesn't

Overall, while I think bLIP 31 is a useful protocol, I don't think it's the right one for this specific feature.

## Conclusion

I'd like to get feedback from other implementations and developers who work on wallet software.
Can you comment on your preferred option for this feature, and anything you'd like to change?
Once we have rough consensus, I'll update [bLIP 42](https://github.com/lightning/blips/pull/42) accordingly.

-------------------------

