# Human Readable Bitcoin Payment Instructions

MattCorallo | 2024-02-11 01:16:06 UTC | #1

Various Bitcoin and other cryptocurrency applications have developed human-readable names for payment instructions over time, with marketplace adoption signaling strong demand for it from users. Having a standard way to resolve a human-readable name into Bitcoin payment instructions is a very useful primitive. I've written up (and implemented) a draft BIP to do this using DNSSEC at https://github.com/TheBlueMatt/bips/blob/d46a29ff4b4ac27210bc81474ae18e4802141324/bip-XXXX.mediawiki which has a number of advantages over the existing LN Address as well as being generic for all Bitcoin payment forms (see the rationale section for more info).

By resolving DNS names to bitcoin: URIs, it is also highly extensible - any Bitcoin payment scheme that defines a bitcoin: URI query parameter doesn't have to do anything and can support this resolution scheme (as long as they have a static invoice format, so no BOLT11).

-------------------------

ajtowns | 2024-02-11 03:10:47 UTC | #2

Did you consider an approach like having `matt@mattcorallo.com` first resolve `_bitcoin-payment.mattcorrallo.com` to an LN node's address via DNSSEC, and then you query that node via an LN onion message for a bolt12 invoice for `matt` ? For domains shared between many users (eg `x.com` or `visa.com` or `bankofamerica.com`), that might provide substantially better privacy, and might also be easier to deploy since you don't have to export your list of LN users into your DNS db?

-------------------------

MattCorallo | 2024-02-11 05:55:22 UTC | #3

The BIP doesn't specify that, but that's supported for the LN usecase! Because the various parts of the overall flow are relevant to different groups the total spec is somewhat confusingly split across three places, but most of the lightning-specific stuff is at https://github.com/lightning/blips/pull/32 and indeed includes a flow like that (but we make the payer only do a single DNS lookup by requiring such deployments use a wildcard to return the "ask node X for the offer" result).

The current bLIP spec there may change in that currently it adds two round trips (one to ask for a standard offer, then one to turn that offer into an invoice_request), but I may want to change that to an offer where you include the user/domain in the invoice_request. Needs some discussion with lightning folks to figure that out I think.

-------------------------

