# Privately sending payments while offline with BOLT12

andyschroder | 2024-09-14 07:01:38 UTC | #1

# Overview

I have an idea for devices to authorize payments from their remote online node while the device authorizing the payment has no direct internet connection. The receiver needs their node to be online though. I think we would need a slight adjustment to the BOLT12 `invoice` for this approach to be secure.

I'd appreciate any thoughts and criticism you may have!

# Consider this scenario

1. Buyer is at home with their phone phone connected to the internet (say it is running a Zeus wallet connected to a remote node).
2. Buyer requests many `invreq_payer_id` and `invreq_paths` to be pre-computed on the node and stores them on the phone along with the secret keys corresponding to all of the `invreq_payer_id`. Each `invreq_payer_id` is assigned a validity time and budget and can be manually revoked later (in case the phone gets stolen).
3. Buyer leaves home with their phone.
4. Buyer's node remains at home connected to the internet.
5. Buyer arrives in a foreign land and can no longer connect to the internet.
6. Buyer gets hungry
7. Buyer finds a seller of food.
8. Buyer needs to make payment for the food.
9. Seller's point of sale device wants to accept payment and has an internet connection.
10. Buyer's node back at home still has an internet connection.
11. Seller's point of sale provides a BOLT12 `offer` over NFC or QR.
12. Buyer's device reads the `offer` and responds with a signed `invoice_request` over NFC, QR, or bluetooth using some of the `invreq_payer_id` and `invreq_paths` it pre-computed before leaving home. It ignores the `offer_paths` or `offer_issuer_id` fields of the offer since it is not sending the reply as an onion message. It also sends a `reply_path` for sending an `invoice` back as an onion message.
13. Seller's point of sale device receives the `invoice_request`.
14. Seller's point of sale device responds with an onion message `invoice` back to the buyer's node over the internet.
15. Buyer's node back at home receives the onion message `invoice` and pays the `invoice`.
16. Seller's point of sale device receives the payment and seller provides food.
17. Buyer eats.


# Changes to BOLT12 needed

The main flaw I see here is that when the buyer's node receives the invoice, they have no way to check the `invoice` matches the `invoice_request` they authorized. In BOLT12, it seems to be the sender's job to check all the fields in the `invoice` that they match their original `invoice_request` before paying. Could the signature in field `240` in the `invoice_request` be copied over to the `invoice` to field `239` so that the payer can verify they trust the invoice they received? It seems like this could also add to a lot of scalability and performance advantages for node implementations because they won't have to keep track of all the outstanding `invoice_requests` that they have sent out (and periodically garbage collect them), they will just have to keep a list of `invreq_payer_id`. The logic here is that the sender's node should be able to trust the signature from the `invreq_payer_id` it pre-generated for the buyer before the buyer went offline.



# Workflow Advantages

- Buyer/Sender does not need to be online. Seller does not need to provide a general purpose internet connection to its customers.
- Sender privacy is maintained.
- Seems very simple, may be able to be baked into a thin 85.6 mm × 53.98 mm smart card and not even need a phone.
- Could work with all point of sale devices and wallets.
- No webserver or SSL needed.
- Could be much more secure than alternatives such as https://github.com/theDavidCoen/LNURL-withdrawPOS .

-------------------------

ZmnSCPxj | 2024-09-14 11:30:29 UTC | #2

I have a long-ago tweet with a similar idea, i.e. the payer sends encrypted data to the payee, plus an onion-message path to its LSP or its high-uptime node.

Note that an LSP (instead of a high-uptime remote-controlled node) can be used here if the mobile phone of the payer is the client of that LSP.  We only need to send the necessary messages (`update_add_htlc` et al) over onion-message instead of normal BOLT8 tunnel.  Thus, I propose:

1. Have some standard by which a remote control can talk to a high-uptime home node via BOLT8.
2. Have a separate other standard by which a high-uptime payee can create a "BOLT8 tunnel" over onion messages, so that the payee can assist the payer in contacting either its LSP or its high-uptime home node.  The LSP or the high-uptime home node then translates the onion messages to BOLT8 messages and treats them as another BOLT8 tunnel.

This expands the usability of the scheme to allow for both remote-controlled home nodes and mobile phone-with-LSP nodes to be the payer here.

-------------------------

andyschroder | 2024-09-14 15:48:20 UTC | #3

You are talking about a custodial or non-custodial LSP?

-------------------------

t-bast | 2024-09-17 08:13:05 UTC | #4

[quote="ZmnSCPxj, post:2, topic:1134"]
Have some standard by which a remote control can talk to a high-uptime home node via BOLT8.
[/quote]

That was exactly the goal of https://github.com/lightning/blips/pull/28, which can provide good security by pairing your remote node with a hardware wallet (e.g. Ledger or Coldcard). I still thinks it would be very useful to standardize the control messages in this bLIP.

-------------------------

andyschroder | 2024-09-23 03:34:23 UTC | #5

[quote="t-bast, post:4, topic:1134"]
That was exactly the goal of [bLIP-0028: Paratonnerre by t-bast · Pull Request #28 · lightning/blips · GitHub ](https://github.com/lightning/blips/pull/28), which can provide good security by pairing your remote node with a hardware wallet (e.g. Ledger or Coldcard). I still thinks it would be very useful to standardize the control messages in this bLIP.
[/quote]

Does bLIP-0028 do any onion messaging?

Also, at a high level, how does bLIP-0028 differ from what https://vls.tech/ attempts to do?

bLIP-0028 looks cool for a lot of use cases, but I'm wondering if it is much more complex in its feature set than what I've proposed for simple invoice payment with a merchant.

-------------------------

andyschroder | 2024-11-11 02:38:36 UTC | #6

[quote="andyschroder, post:1, topic:1134"]
Could be much more secure than alternatives such as [GitHub - theDavidCoen/LNURL-withdrawPOS: A reference flow for open source implementations of POS devices able to read LNURL-withdraw links via NFC ](https://github.com/theDavidCoen/LNURL-withdrawPOS) .
[/quote]

Related: I've discovered that there is a better LNURL solution that uses deterministic rolling withdrawal URI's with a special NFC chip: https://github.com/boltcard . The reference above appeared to use static LNURL withdrawal links, which seemed mostly insecure to me. The Bolt Card solution seems "okay" (despite all the architectural drawbacks of LNURL and the fact that you can't pre-authorize exact amounts).

-------------------------

