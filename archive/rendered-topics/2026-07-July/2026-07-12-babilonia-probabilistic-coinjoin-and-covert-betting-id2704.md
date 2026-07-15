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

AdamISZ | 2026-07-15 12:42:51 UTC | #2

A couple of addendums: first, about @pGerhart 's paper as noted above (well let's call it GTT 26 from now on, there are 3 authors). Second, about applying this specific protocol (i.e. Babilonia) to coinswap and not coinjoin; a topic I actually bothered to delve into yesterday after talking to one of the coinswap devs [1].

## About GTT 26

I wrote here in a couple of places that it's a fundamentally more powerful construction. On closer inspection: I still think that's true, but I also think Babilonia gains something quite important from its simpler construction, in short: **it lets you do joins (one tx flow) and not only swaps (two separate flows)**.  In a nutshell, GTT 26  uses an OPRF that looks like $H(x, sk \cdot H_{G}(x))$ where $x$ is your input data, $H_G$ is a hash into the curve and $sk$ is the (dealer's) secret key. This lets the dealer make a secret choice of $x \in \mathbb{Z}_p$ and by passing it through the OPRF get a value $y_{win}$ that can be committed to in $Y_{win}$ as a group element (and therefore key). This means that the commitment the dealer publishes can be drawn from an exponentially sized set (i.e. $\mathbb{Z}_p$) which is the big advantage: you can represent very small probabilities this way.

On the other hand, since this OPRF Evaluation function requires the dealer's secret key, we have to set up: 1/ that the player can blind their evaluation request (think about how Chaumian cash works; it's similar) and 2/ that they have to atomically pay the dealer for this evaluation using a swap structure - what they call "evaluation as a service". While in Babilonia the player gets (if guessed correctly) the required answer directly from $a_c = ctxt - f(t)$ without interaction. That seems to be the intrinsic difference between the two structures: one allows very low probabilities by drawing from an exponential set, and thus supporting e.g. lotteries (which could be super useful), the other does not, but allows a simpler "one transaction" design, because the player can calculate the result from the single adaptor produced in one spend event.

TLDR: my current understanding is that *this* protocol (Babilonia) has advantages specifically related to coinjoin/payjoin/steganographic payment structures, keeping a betting function that is more limited (countable outcomes), but not realistically (at least, afaik) allowing a lottery-like function. [2]

## Coinswap via Babilonia?

I realized after some thinking that applying the same mechanism to private coinswap isn't trivially the same as the coinjoin construction presented in the paper, but at the same time it's not crazily different. Also note of course that GTT 26 as mentioned above, fits here.

The 10000ft view is: Alice is maker, funds a tx that will pay out to Bob, the taker. Bob does the same in reverse. Alice as maker collects fee $f$, so Bob ends with $X - f$ and Alice ends with $X + f$ (ignoring network fee).

Next step is the reasoning "$f$ partially breaks amount correlation, but very poorly: first, because it is inevitably small (less than 1% or 0.1% would be almost always true), and second because order books for the market are public."

The Babilonia framework application would be: break the amount correlation not only by increasing the concrete values of $f$ in specific transactions, but by varying them an amount that is both (a) large and also (b) private and (c) statistically fair, i.e. $f + \delta$ where $\delta \gt f$ (not required to be $\gt$ but *can* be).

In order to achieve this, the simple description is: make *two* distinct output transactions paying to Bob on Alice's branch, instead of 1. They pay to $K_i = W_{B,i} + A_i$ (in the notation of the paper), so when Bob collects his $ctxt$ as normal, knowing it will decrypt to $A_{c},\ c \in {1,2}$, he will be able to spend exactly one of the two presigned transactions, so he will broadcast that one. And of course one pays him $X - f - \delta$ while the other pays him $X - f + \delta$ (this can be arranged with change outputs and overfunding by Alice). The adaptor is as per Babilonia some $T$ s.t. $t$ decrypts the thimble ($A_c$) for *one of* of these output transactions.

I got Claude to write some more details, more aimed at bitcoin engineers who don't want reams of cryptography; *maybe* it helps you understand, if you're curious: https://gist.github.com/AdamISZ/ebae2d5f547f56c6f8164db19f718795

Why does this matter? People doing swaps probably generally *don't* want to make bets, right? Well, I think it matters a lot: coinswap as a concept is almost crippled by amount correlation, and *without* amount correlation, it's almost ludicrously powerful. Amount splitting is not only expensive and complex, it's also a partial and brittle solution. This technique, while it has a cost (measurable stochastically, because the outcomes are fair) completely breaks amount correlation, because the delta between the two branches is negotiated in private, and represents an actual fund transfer.

[1] Referring to the project at https://github.com/citadel-tech/coinswap 

[2] There are of course other differences, weight of ZKP, onchain footprint, security reductions etc.

-------------------------

