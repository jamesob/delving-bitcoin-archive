# E2E encryption for BOLT11/BOLT12 description/note field was this discussed before?

4xvgal | 2026-07-27 08:38:26 UTC | #1

Hi all,

I checked several implementations and services — BTCPayServer, LND, CLN, LDK, Breez SDK (Liquid/Spark/Greenlight), Spark, bark, Arkade — and it looks like: **regardless of BOLT11 vs BOLT12, and regardless of custodial or non-custodial, the note field is exposed in plaintext not to the payer’s intended final receiver, but to whatever infra is actually issuing/processing the invoice**.

Here’s a example why this matters in practice. Say you buy a VPN subscription or an eSIM using a custodial Lightning wallet. To reconcile the payment with your order, the receiver (merchant backend) commonly puts order metadata directly into the invoice `description` — things like order ID, which product/plan you bought, sometimes even which merchant/reseller it came from. Since the wallet provider is the one issuing/forwarding the invoice, they can read this `description` too. They don't just know "user X sent Y sats at time Z", they also know **what** you bought, **from whom**, and **when**.

**Has E2E encryption for the note/description field been discussed already?** If there's an existing thread or prior proposal, please point me to it. If not, I'll share the draft as a separate thread.

-------------------------

