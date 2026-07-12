# Babilonia - probabilistic coinjoin and covert betting

AdamISZ | 2026-07-12 19:07:36 UTC | #1

Inspired by the discussion [here](https://delvingbitcoin.org/t/emulating-op-rand/1409) about trustless onchain coin-flips, [1] I came up with a protocol that has the following properties:

* small onchain footprint (two payment-size transactions with one confirmation wait, though a third is often needed but without extra negotiation)
* indistinguishable from normal payments and/or payjoins in the happy path
* invisibly executes a fair bet between the two participants
* is actually a payjoin (or kind of two payjoins, depending on how you squint at it; the point is, it's two normal payments)
* uses very very simple ZKP techniques, no heavy machinery

Please see details at https://github.com/AdamISZ/babilonia-paper

Paper made from 100% human I swear! Or maybe 95% :grinning_face_with_smiling_eyes:

One detail to give a flavor of the thinking: "pot sweeteners" means you can have one party incentivize the other to participate *without* watermarking the transactions in the way that Joinmarket does - that's because it's actually a transfer of value.

A 10000ft view might be: payments break subset sum, and bets are just taking that one step further: even when you have no coffees or Alpaca socks to buy, you can participate in the creation of a steganographic obfuscation effect, *without* having to trust a central entity. Just like non-probabilistic payjoins that are for paying a merchant, the effect is quite small on a per-tx basis but perhaps not small at scale (hence 'pot sweeteners' and similar ideas may really matter, here).

One detail that is minor to the paper, but quite fun to think about is the use of decoys in BIP324 traffic for 2 nodes to negotiate a protocol like this, if it is done correctly and Sybiling is addressed, it creates properly invisible p2p negotiation.

Speaking of which, an implementation of the described protocol is at https://github.com/AdamISZ/babilonia . Unlike the paper, this is almost entirely LLM coding (Claude Opus 4.8 to be precise), with minimal human input. But it seems to work well in my tests so far. For those who prefer to read code rather than read a paper, sorry: even the docs there are not really cleaned up for human consumption, though I probably will do that at some point if there's interest.

An example protocol execution on signet:

https://mempool.space/signet/tx/59bb7633314b35d2edcff7997281c0254e9a9b36faf22dbc643d4a319d92f064

https://mempool.space/signet/tx/1112032da666df1efaae5c2e8d567bfa200e0a94dd1c5aa8e1e89a14b32f4c44

Finally perhaps worth mentioning that *swaps* using this technique may have even better properties, though there are tradeoffs (again, it's briefly discussed in-paper).

Welcome any thoughts on the cryptography, the protocol, applications, whatever.

[1] as discussed in the paper, the other paper that Paul Gerhart introduced in that thread, is really a more powerful construct along similar lines: so you may feel free to read that instead. What I describe in Babilonia is more limited but has certain minor advantages that fit this coinjoin implementation, I think.

-------------------------

