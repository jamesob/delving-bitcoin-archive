# op_CAT vs op_CTV vs XMR

securitybrahh | 2024-12-03 10:46:57 UTC | #1

I have been very skeptical about monero but I don't think bitcoin can become cash, so something has to be cash.

Either Lightning takes that role or we have to rely on monero.

I don't know how much taproot and segwit has reduced the [fungibility] (https://sethforprivacy.com/posts/fungibility-graveyard/) of bitcoin and how much sender privacy is there in lightning (maybe if you trust the node/channel the sender is on), but for something to be cash, it has to retain price, and I think monero is it (for now)

I actually like the CTV soft fork or we can just have pools via [OP_CAT]. (https://github.com/taproot-wizards/purrfect_vault/issues/1#issuecomment-2513445570) But monero is it.

I believe Bitcoin is money but monero is cash, as Bitcoin is not fungible & lightning adoption will be slow, I try to put my thoughts about this here: (hope linking my blog is fine, otherwise, I will remove it, mostly I want to discuss what I am thinking) 

[https://letters.empiresec.co/p/monero-is-cash](https://letters.empiresec.co/p/monero-is-cash)

-------------------------

HubertusVIE | 2024-12-09 22:12:18 UTC | #2

This is a reply from an economics point of view, contra Monero.

1. Bitcoin can become cash (currency for the real economy).
2. It is PRIMARILY held back by an economic issue, not a technical one. Monero has the same problem. Let's call it volatility but there is more to it. 
3. I think the best way for bitcoin to become cash is a bitcoin based Chaumian ecash layer. Long story.
4. Lightning is needed for the total system but is not 'cash' per se.
5. Bitcoin-based ecash has blinded signatures, this gives great fungibility.
6. No CTV needed (for this cash goal). If just for that, Core can just stay as is.

I am currently writing up the economic problem and a proposed solution which is already being developed. If interested, here are the first two chapters: https://blog.bitcr.org/p/the-bitcoin-dilemma-store-of-value-or-medium-of-exchange.

-------------------------

moonsettler | 2024-12-30 15:15:55 UTC | #3

I think it's worth pointing out that nothing we currently do is post-quantum. CTV would be quantum resistant as far as SHA256 is, but the scaling related constructs take extensive use of Taproot Schnorr sig aggregation. The current blind Schnorr ecash is not post-quantum either. Someone with access to a quantum computer could easily print infinite amount of "cash" for himself.

Monero does have a good enough path to mitigate it's privacy issues as well as providing forward secrecy for a post-quantum era with the FCMP++ and subsequent updates. Afaik they are on track to deliver these updates within 3 years.

Now ofc many dismiss the quantum threat as FUD, but within 3 to 10 years as far as we "know", there is a significant chance that this becomes an issue. Bitcoin development especially regarding post-quantum scaling and privacy is simply not on track to deal with this in any plausible manner.

**edit:** inb4 people are working on PQC for bitcoin! That's great, however to my knowledge the whole body of work is massively anti-scaling and there has been no realistic sounding migration plan proposed.

PS: shouldn't this thread be moved to Philosophy from Protocol Design?

-------------------------

