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

harding | 2024-08-05 20:53:19 UTC | #2

[quote="t-bast, post:1, topic:1046"]
However, when sending money to their friends and family, users expect some form of mutual authentication: when Alice sends money to Bob, she wants Bob to know it came from her.
[/quote]

Sorry if this is a foolish question, but is this something that requires cryptography to solve?  If Alice wants Bob to know that a payment came from her, why not simply add a text `from` field to the onion that gets encrypted to the recipient that can say: `from: Alice`?

That would allow someone else to pay Bob while claiming to be Alice, but is that a risk?  (That's an honest question.)  It seems like it might actually be a feature: I occasionally make payments on behalf of other people, e.g. when I order a pizza but my partner will be picking it up; it'd be nice if I could simply set the `from` field to the name of the intended payer.

-------------------------

t-bast | 2024-08-07 10:28:25 UTC | #3

[quote="harding, post:2, topic:1046"]
It seems like it might actually be a feature: I occasionally make payments on behalf of other people, e.g. when I order a pizza but my partner will be picking it up; it’d be nice if I could simply set the `from` field to the name of the intended payer.
[/quote]

I think this isn't mutually exclusive with what I'm proposing (this can be done with the `payer_note` field already), but I think we do need the mutual authentication option.

As you note, the issue with a payment note that isn't mutually authenticated is that it cannot be trusted. When sharing (some) offers publicly, this creates a serious risk of phishing because anyone sending a payment to you can include whatever message they'd like. Even if you don't share your offer publicly, someone else to whom you sent it may leak it. I think that by default wallets should show payment notes as "untrusted", and somehow explain to the user that whoever wrote that message may not be who they claim to be, unless mutual authentication was successfully performed.

Attackers can be very creative with phishing, so I'd like an easy way for users to know whether they can trust payment notes or not. If the payment really comes from one of your trusted contacts, you know you can trust it. Otherwise, you should be cautious with whatever message is included (but you can of course still trust it if it seems to make sense, for example in your pizza scenario).

-------------------------

harding | 2024-08-07 14:12:45 UTC | #4

[quote="t-bast, post:3, topic:1046"]
Attackers can be very creative with phishing, so I’d like an easy way for users to know whether they can trust payment notes or not. If the payment really comes from one of your trusted contacts, you know you can trust it.
[/quote]

The creativity of attackers is part of my concern. I assume the expected UI works like this:

1. Alice clicks/scans an offer created by Bob that contains his contact key.
2. Alice's wallet says "Do you want to add the recipient to your contacts list?"  Alice clicks Yes and the wallet prompts her to enter Bob's name.
3. The wallet completes the payment flow.
4. At some later point when Bob sends a payment to Alice, it shows up as _From Bob_.

Can Mallory abuse that flow to get her key associated with Bob's name?  Maybe she offers a giveway using the merchant-pays-user flow:

1. Mallory creates a page promising that the first 100 responders will receive 100,000 sats for them plus 100,000 sats for their best friend.
2. The instructions on the page say: "click this link and then, in your wallet, enter the name of your best friend who you also want to receive 100,000 sats.  Hurry up, only the first 100 responders will receive 100,000 sats"
3. Alice quickly follows the instructions, associating Mallory's key with Bob's name.
4. Mallory performs whatever evil it's possible to perform with access to forge payments from Bob.

Admittedly, that's a lot more work for Mallory than just lying in the `payer_note` field.  But I think it's a lot easier to accustom users to the expectation that the `payer_note` is arbitrary text that can contain lies than it is to prevent creative phishers from being able to associate their keys with the names of other people and organizations.  If phishers are able to obtain access to a trusted field, that may magnify the damage they can do over only having access to fields that are known to be untrustworthy.

Some additional problems:

- What happens when a contact key gets compromised?  For example, an organization contact key used with thousands or millions of customers.
- What happens when Bob uses multiple wallets?  For example, he sometimes sends payments to Alice from his mobile wallet; other times, he pays from his desktop wallet with a different seed.  Will Alice's wallet allow associating the same name with multiple contact keys?  Will there be significant user consternation and support issues if some of Bob's payments show up as untrusted?

I'm sorry to be producing [stop energy](http://radio-weblogs.com/0107584/stories/2002/05/05/stopEnergyByDaveWiner.html).  My thinking is that it might be both easier and safer to simply train users that nothing about a payment should be trusted except the amount.

-------------------------

t-bast | 2024-08-09 07:01:37 UTC | #5

[quote="harding, post:4, topic:1046"]
Alice quickly follows the instructions, associating Mallory’s key with Bob’s name.
[/quote]

You'd do a great scammer :smiley: 

But I don't think this kind of attack makes sense: wallets should make it perfectly clear that you are adding a contact that will be associated with the given "payment code" (or some other wording). I don't see how Mallory would be able to lure Alice into this flow in a credible way, Alice knows that she's not paying Bob and this payment code wasn't generated by Bob.

[quote="harding, post:4, topic:1046"]
What happens when a contact key gets compromised? For example, an organization contact key used with thousands or millions of customers.
[/quote]

If the contact key gets compromised, it likely means the organization's seed got compromised, so they lost all their money...I think having their contact key compromised is the least of their concern if that happens!

But this is indeed annoying, because the only thing the organization can do is notify its users that their contact key has changed. This is somewhat similar to having your users' passwords being stolen, in which case you must notify them all. This isn't great, but I don't think it's critical either.

[quote="harding, post:4, topic:1046"]
What happens when Bob uses multiple wallets?
[/quote]

You should be able to associate multiple keys/offers to contacts. I don't see a reason why that wouldn't work.

[quote="harding, post:4, topic:1046"]
My thinking is that it might be both easier and safer to simply train users that nothing about a payment should be trusted except the amount.
[/quote]

Unfortunately, I don't think we can do that: businesses and protocols will likely want to use that `payer_note`, and will train users the other way, telling them to look at this field and ignore the wallet's recommendation...so I'd rather have a way to make this safer, even though it will never be perfectly safe since keys can be compromised :confused:

-------------------------

vincenzopalazzo | 2024-09-02 15:06:10 UTC | #6

Thanks, t-bast, for working on this. I had been thinking about implementing something similar in CLN a while back.

[quote="harding, post:2, topic:1046"]
Sorry if this is a foolish question, but is this something that requires cryptography to solve? If Alice wants Bob to know that a payment came from her, why not simply add a text `from` field to the onion that gets encrypted to the recipient that can say: `from: Alice`?
[/quote]

This makes sense if you're not building a payment system that requires verification. For example, today you can use one of the Bolt12 methods used by Ocean to send a payout to a miner, and someone could try to send a payment with a spam description. This is one reason why Ocean includes a description, so if a spammer sends a payment, the miner just ends up with more money.

Currently, there's no way to identify that the payer wasn’t Ocean without going to Ocean and asking, 'Hey, was this you?' Ocean can then prove with the `invoice_payer_id` that it was someone else.

[quote="harding, post:1, topic:1046"]
When Alice wants to pay Bob (who is one of her trusted contacts), Alice can optionally decide to use her own `contact_key` to reveal her identity to Bob. Bob should only learn that the payment comes from Alice if Alice is also on Bob’s contacts list.
[/quote]

Personally, I prefer Option 1 because it’s simple (and it aligns with my design :slight_smile: so I’m biased here).

However, I’m not sure I fully understand the use case for Option 2. Why do you think a user might need a per-contact `invreq_payer_id`?

Probably regarding privacy for the `invreq_payer_id`? Do you think this could be an additional feature?

Finally, I also think that bLIP-31 is a bit overcomplicated for a feature like this, but I want to think more about it. It's possible I haven't encountered a use case where this protocol could be used for exchanging contact IDs.

-------------------------

