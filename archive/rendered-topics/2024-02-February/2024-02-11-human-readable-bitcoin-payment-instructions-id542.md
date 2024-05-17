# Human Readable Bitcoin Payment Instructions

MattCorallo | 2024-02-11 01:16:06 UTC | #1

Various Bitcoin and other cryptocurrency applications have developed human-readable names for payment instructions over time, with marketplace adoption signaling strong demand for it from users. Having a standard way to resolve a human-readable name into Bitcoin payment instructions is a very useful primitive. I've written up (and implemented) a draft BIP to do this using DNSSEC at https://github.com/TheBlueMatt/bips/blob/d46a29ff4b4ac27210bc81474ae18e4802141324/bip-XXXX.mediawiki which has a number of advantages over the existing LN Address as well as being generic for all Bitcoin payment forms (see the rationale section for more info).

By resolving DNS names to bitcoin: URIs, it is also highly extensible - any Bitcoin payment scheme that defines a bitcoin: URI query parameter doesn't have to do anything and can support this resolution scheme (as long as they have a static invoice format, so no BOLT11).

-------------------------

ajtowns | 2024-02-11 03:10:47 UTC | #2

Did you consider an approach like having `matt@mattcorallo.com` first resolve `_bitcoin-payment.mattcorrallo.com` to an LN node's address via DNSSEC, and then you query that node via an LN onion message for a bolt12 invoice for `matt` ? For domains shared between many users (eg `x.com` or `visa.com` or `bankofamerica.com`), that might provide substantially better privacy, and might also be easier to deploy since you don't have to export your list of LN users into your DNS db?

-------------------------

MattCorallo | 2024-02-11 06:00:07 UTC | #3

The BIP doesn't specify that, but that's supported for the LN usecase! Because the various parts of the overall flow are relevant to different groups the total spec is somewhat confusingly split across three places, but most of the lightning-specific stuff is at https://github.com/lightning/blips/pull/32 and indeed includes a flow like that (but we make the payer only do a single DNS lookup by requiring such deployments use a wildcard to return the "ask node X for the offer" result).

That doesn't result in the privacy advantage that you note, but I also wanted to keep this generic for all of Bitcoin (including on-chain, fedimint, cashu, silent payments, whatever) so kind of want to avoid explicitly baking in a lightning onion message dependency. Of course that means its annoying to add on-chain addresses for large custodians, but maybe after lightning they can invest in figuring out how to run bind and create a zonefile...turns out even with a million or ten records it works just fine :)

The current bLIP spec there may change in that currently it adds two round trips (one to ask for a standard offer, then one to turn that offer into an invoice_request), but I may want to change that to an offer where you include the user/domain in the invoice_request. Needs some discussion with lightning folks to figure that out I think.

-------------------------

t-bast | 2024-02-12 12:41:49 UTC | #4

That was exactly one of the flows I proposed in https://gist.github.com/t-bast/78fd797a7da570d293a8663908d3339b#option-1-use-dns-records-to-link-domains-to-nodes

I think it is useful and simple indeed, especially for lightning providers who want minimal DNS operational burden.

EDIT: this is covered in the bLIP that Matt created: https://github.com/lightning/blips/pull/32

This is the `omlookup` path: a domain that has many users would create a single DNS record `*.user._bitcoin-payment.domain.` containing a blinded path to themselves (which may actually be a 0-hop blinded path directly exposing their `node_id`). Clients then use that blinded path to request an offer for a specific user via an onion message.

-------------------------

sjors | 2024-02-22 09:48:16 UTC | #5

How do you want to handle testnet and signet(s)? Perhaps `._bitcoin-testnet-payment` and `._bitcoin-signet-payment`?

I have this itchy feeling of wanting to avoid the word "bitcoin" in both the subdomain and the record, to make both filtering and creating a map of all domains "involving" bitcoin require at least a few more seconds of effort.

Why use BIP21 style URI's inside the text record? And why have a single big record for all payment types? For example instead of `matt.user._bitcoin-payment.mattcorallo.com. 3600 IN TXT "bitcoin:?b12=lno1q...` you could have `matt.user.b12._bitcoin-payment.mattcorallo.com. 3600 IN TXT "lno1q...`.

I guess one argument could be that using a single big record makes it hide what the payer is looking for. But you can just query all of them. Or is a single query easier to implement?

One record per address type potentially avoids having to deal with the 255 character limit for TXT records.

-------------------------

MattCorallo | 2024-05-17 17:39:17 UTC | #6

Sorry, I don't generally check delving so usually miss replies, feel free to reply on the BIP PR.

[quote="sjors, post:5, topic:542"]
How do you want to handle testnet and signet(s)? Perhaps `._bitcoin-testnet-payment` and `._bitcoin-signet-payment`?
[/quote]

I have not given it any thought, honestly. 

[quote="sjors, post:5, topic:542"]
I have this itchy feeling of wanting to avoid the word “bitcoin” in both the subdomain and the record, to make both filtering and creating a map of all domains “involving” bitcoin require at least a few more seconds of effort.
[/quote]

Maybe? The protocol will always be fairly identifiable, including addresses which are going to be fairly easy to google and figure out, I think.

[quote="sjors, post:5, topic:542"]
Why use BIP21 style URI’s inside the text record?
[/quote]

Because its already a well-defined format for communicating multiple payment instructions in a way that most wallets support. Why reinvent the wheel?

[quote="sjors, post:5, topic:542"]
And why have a single big record for all payment types? For example instead of `matt.user._bitcoin-payment.mattcorallo.com. 3600 IN TXT "bitcoin:?b12=lno1q...` you could have `matt.user.b12._bitcoin-payment.mattcorallo.com. 3600 IN TXT "lno1q...`.
[/quote]

Because then a sender has to add extra logic to enumerate the set of supported formats and send multiple queries. Most senders already have logic to handle dealing with a single BIP 21 with multiple payment instructions, so we should just capitalize on that by giving them a simple API to fetch a BIP 21 to feed into their existing logic.

[quote="sjors, post:5, topic:542"]
One record per address type potentially avoids having to deal with the 255 character limit for TXT records.
[/quote]

No such limit exists.

-------------------------

