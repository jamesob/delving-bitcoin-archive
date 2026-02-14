# Silent Payments notifications via Nostr

setavenger | 2026-01-15 23:09:56 UTC | #1

This write-up has the goal to discuss the approach of sending notifications for incoming Silent Payments transactions via Nostr.
The design is an extension of the discussions around [“Stealth addresses using nostr”](https://delvingbitcoin.org/t/stealth-addresses-using-nostr/1816) which had a Nostr-only approach.

The design allows for notifications being sent outside of Nostr so for a clearer structure there is a Silent Payments and a Nostr part.
The Silent Payments outlines why the schema is suggested as it is and the Nostr part discusses the using nostr as the communication layer for the notifications.

*Note: this was posted as Gist before the comments received there are summarised below as an “appendix”*

## Silent Payments

Silent Payments offer a novel way for receiving payments without interactivity.
But the way SP is designed it requires us to look for the needle(s) in the haystack(s).

The sender (Alice) knows which needles she placed in the haystacks.
The receiver (Bob) needs to check everything to know what belongs to him!

The BIP already proposed out-of-band communication to somewhat alleviate this burden for the receiver.
If the sender lets the receiver know that she made a payment to Bob in tx1 then Bob does not necessarily have to check tx2,3,…,n in a given block.
If Bob thinks Carol or an unknown person sent him a payment without a notification he can always fall back to full scanning.

### So how can we structure such a notification?

The minimum requirement for notifications to make sense is the txid.
Then Bob can theoretically find all the relevant information to prove to himself that the notification is legitimate.

But ideally Alice sends more information.
In order to fully compute the outputs “from scratch”, Bob needs the previous scriptpubkeys.
Those are not within the block data and in most cases are not trivial to retrieve.
So Alice should also provide the tweak for the tx.
This adds no burden to Alice as she had to compute tweak anyways to compute the output for Bob.

A confirmed blockhash/-height would be very useful for Bob to find the txid faster.
This should only be optional though.
Providing Bob with the confirming block is going to add a new burden on Alice.
Alice then has to monitor the transaction and can only send the notification once she knows the confirming block.

*Note: One could also add the relevant outputs to this as well.
I’m not certain where I stand on this.
The notification and the transaction should be verified.
In that process the outputs would be most likely touched anyways.
If Merkle proofs were used the confirming height/hash would be required.*

### Schema

The final content will look something like this:

```json
{
    "txid": "5a45ff552ec2193faa2a964f7bbf99574786045f38248ea4a5ca1ff1166a1736",
    "tweak": "03464a0fdc066dc95f09ef85794ac86982de71875e513c758188b3f01c09e546fb",
    "blockhash": "94e561958b0270a6a0496fa8313712787dcacf91b3d546493aea0e7efce0fc45" // optional
}
```

\*Note that the blockhash is optional and can be omitted by Alice.
In that case Bob needs to check what the status of the transaction is.

### Receiving the notification

> This write-up tries to be as unopinionated as possible regarding wallet implementations.
> Ideally any source from full-node to light client + indexer should work with the design outlined.

One way how this could look:
Upon receiving the notification Bob checks for the txid against his chain backend.
Bob pulls the transaction and does the receiver-scan computations for the transaction.
He already has the tweak so he needs no further external information.
He computes the ECDH secret and finds out which outputs in the transaction belong to him.
Then Bob’s wallet can proceed as it would with any other found output.

## Nostr

Using Nostr as a means of communication for the notification messages would be an option.
Which Nostr messaging logic and which keys are used is quite important.
The wrong approach could leak metadata and degrade privacy of Bob and possibly Alice.
Many of the intricacies have been discussed in the post and comments of [“Stealth addresses using nostr”](https://delvingbitcoin.org/t/stealth-addresses-using-nostr/1816) on Delving Bitcoin.
I have outlined the worst case below:

> A bad design would be using NIP-04 direct messages. Using Alice’s npub and Bob’s scan key. NIP-04 leaks which npubs have had direct communication. The whole world would know that Alice has had interaction with Bob’s scan key.

NIP-17 would conceal the sender but still show which npub has received some form of information.
These notes could be anything from privately stored information to a message or notification.
As there the sender is not linked these notes can be anything.

### Which key does the receiver use?

Now that Alice’s key does not matter anymore we will look at the keys for Bob.
Several Options exist:

1. Use the scan key
2. Use a dedicated new key for the address
3. Bob’s existing npub that he uses for normal nostr activity, or any key with additional non-SP activity

(1) Using the scan key would be simple as it does not require additional handling of an npub for outside activity which is not related to the SP address.
An immediate downside is that any Nostr activity for that npub would be a strong “payment received” signal for outside observers.

(2) A key which was created with the sole purpose of using it for receiving notification messages would have the same issues as (1).

(3) The additional activity would help conceal the strong “payment received” signal.
The only issue would be if Bob wants to separate his Nostr identity and his SP address.
But in that case it’s probably better to fall back to full scanning for now.

It should be noted that one could design wallets in a way that Alice could copy-paste the above JSON to send via any messenger. Bob then imports the info into his wallet and the process would continue in the same way.

### Relays

To briefly touch upon the relay part of Nostr.
As there is no global state in Nostr Alice and Bob must have at least one relay in common.
Otherwise Bob will never see the notification sent out by Alice.
Therefore Bob should also advertise a list of recommended relays to avoid notifications ending up in the “void”.

## Final Design

Summarising the ideas outlined in this write-up.

### Advertising

Bob advertises his Silent Payment address as a BIP321 URI string as he now already does.
He attaches the optional tags for his npub and the relay list.

`bitocin:?sp1qsilentpayment=&npub=npubbobsprofile&relays=wss://relay1.example.com,ws://relay2.example.com`

### The Content

Using the JSON defined in the Schema section above as the base.

```json
{
    "txid": "5a45ff552ec2193faa2a964f7bbf99574786045f38248ea4a5ca1ff1166a1736",
    "tweak": "03464a0fdc066dc95f09ef85794ac86982de71875e513c758188b3f01c09e546fb",
    "blockhash": "94e561958b0270a6a0496fa8313712787dcacf91b3d546493aea0e7efce0fc45" // optional
}
```

### Communication

Ideally wallet implementations would allow for an import of the above JSON data so notifications can be sent via any messaging layer.
With the Nostr route Alice would send a NIP-17 Nostr DM to Bobs npub via at least one of the specified relays listed in the URI.

### Checking the notification

How any wallet implementation verifies and uses the notification is not scope of this write-up.
Several different designs for Silent Payments wallets exist.
They handle the scanning and finding of UTXOs differently.
Different data sources are used as each wallet makes its own privacy and performance trade-offs.
Therefore every wallet should handle notifications in its own way to keep the trade-offs it wants.

## Additional thoughts

The notification data could be attached in messaging clients and embedded similarly as it’s currently done with Cashu tokens.
Based on this a wallet could do automatic labelling of UTXOs in the wallet.
This is also relatively safe.
Only Alice knows which txid belongs to Bob.
Faking this notification is basically impossible without Alice or Bob leaking info to third parties to begin with.

## Closing remarks

The design is influenced by [“Stealth addresses using nostr”](https://delvingbitcoin.org/t/stealth-addresses-using-nostr/1816).
The desired UX (reduce scanning time) improvements of Stealth addresses are basically identical to the above design.
But this design brings it back to Silent Payments where we have a global state - the blockhain - to which one can always fall back to.
This was my main concern reading Stealth addresses.

## Appendix (Gist summary)

This write-up was posted as a Gist and received a couple of comments.
The write-up was kept the same but I will do my best to give a brief summary of the points which were raised and comment on them in the process.
For details please refer to the [Gist](https://gist.github.com/setavenger/a0cd7e71b47ded9fca9c99085130cf2a)

### Spam Prevention

Spam prevention will be tricky this is undeniably so.
In my opinion any sufficiently motivated bad actor will try spam to Bob with fake or computationally expensive messages.
But once Bob notices that he is being attacked he can simply fallback to on-chain scanning.

→ The best case is a lot better and the worst case is pretty much the same as without this notification spec. Therefore relying on standard Nostr practices with rate limiting relays etc. should suffice.

Some ideas though:
* merkle proofs could limit the damage but the asymmetry is on the side of the attacker here
* Nip-13 PoW on notes.
  Asymmetry is more in favour of receiver but too much of this and we then should rather rely on Bitcoin’s PoW.

=> The spec described here is more on the optimistic side of things.
If things are fine we get to scan very cheaply.
If they go bad everything is just as without notifications.

### Bad implementations sending incorrect tweaks

The following scenario is of concern:

1. Alice sends Bob a notification and includes the tweak.
2. The tweak is valid for an output in the transaction.
3. The tweak was **not** derived according to the Silent Payments spec

Now Bob has two options:

1. Immediately Spend the UTXO / move to a protocol compliant address
2. Make sure to store the tweak and map it to the coin.

The coin has become unrecoverable without the tweak information from the notification message.

-------------------------

RubenSomsen | 2026-01-17 01:00:32 UTC | #2

[quote="setavenger, post:1, topic:2203"]
Bad implementations sending incorrect tweaks

$$
…
$$

Now Bob has two options:

[/quote]

That’s all accurate, but I think you might still be overlooking something important.  For every payment Bob will need to determine whether the tweak data etc. was correct, which requires more data (especially if the exact block isn’t known) and is not trivial.

And if you’re going to be second guessing all the data Alice is sending you, this calls into question whether she should even send it at all. This is why you may want to go with the “absolutely minimal” message (or go the opposite route and “maximize DoS resistance”) as described in my [previous reply](https://gist.github.com/setavenger/a0cd7e71b47ded9fca9c99085130cf2a?permalink_comment_id=5931734#gistcomment-5931734).

-------------------------

setavenger | 2026-01-18 17:01:15 UTC | #3

Thank you for clarifying. The minimal approach would be to only send a txid (if Alice does not wait for confirmation) or a with a block hash + merkle proof for DoS protection.

Thinking out loud here but I think this can be seen through different lenses. Keeping in mind that we separate the Silent Payments layer and the Nostr layer. Moreover, not all wallets will be designed in the same way when it comes to their data backend and scanning logic.

We can establish that if Bob receives a notification from an unknown and un-trusted source that he should not trust any information in that message. He has to make sure he can fail as early as possible with the least amount of computation wasted in the process.

I think the only reason the tweak field is still interesting here is if Bob receives the tweak from his friend Alice. Or even in general someone in his Web of Trust where the DoS probability is generally low. Then Bob only needs to pull the tx from anywhere he trusts. He does not need to be served the scripts of the inputs as he already has the tweak. He then does the typical last steps of a light client who has a tweak and outputs to match against. The kind of additional data Bob needs to find is a lot more standardised. It's just transaction data.

Having the tweak can make a good scenario a lot easier and brings more flexibility to client design. I don't think all light clients will look the same in the end. I alone have tried several approaches with BlindBit and I'm sure others have tried many other approaches as well. Bob can always discard/ignore the tweak if his client does not need or trust it. But with a tweak available it is enough for Bob to get the simple txhex and do the last mile computation to find a match.

I think if we keep it a bit agnostic with regards to how the wallets might handle data serving (some might not even serve data - think Frigate style external scanning) and the involved communication channels the tweak should be allowed in. Maybe only in an optional capacity where it can be discarded if Bob or his wallet don't trust it. The txid would be mandatory as the "absolute minimal" component.

Let me add one question that can steer the further design as well: **Do we expect Alice to wait for the transaction to confirm before she sends the notification message?** If she waits she has a lot more targeted information available that she can send to Bob. Light clients which only rely on filters + pulling blocks will have a difficult time looking up a specific transaction only by its txid (at least without risking privacy degradation). Other wallet designs (e.g. Electrum based) will probably have an easier time making due with only a tweak and a txid. They could benefit of a tweak field in the message.

-------------------------

RubenSomsen | 2026-01-21 10:41:07 UTC | #4

[quote="setavenger, post:3, topic:2203"]
The minimal approach would be to only send a txid

[/quote]

[quote="setavenger, post:3, topic:2203"]
Light clients which only rely on filters + pulling blocks will have a difficult time looking up a specific transaction only by its txid

[/quote]

Your latter sentence is why I suggested the taproot output for the minimal approach rather than the txid. This gives you something to scan for using regular block filters. Though txid filters are theoretically possible too, of course.

[quote="setavenger, post:3, topic:2203"]
Web of Trust where the DoS probability is generally low. Then Bob only needs to pull the tx from anywhere he trusts. He does not need to be served the scripts of the inputs as he already has the tweak

[/quote]

I’m not convinced there is a safe enough middle ground where we only partially check stuff. Either we trust the sender or we don’t. If we trust the sender then we don’t have to fetch any data from other sources at all. The sender can just provide everything that is needed to spend the output (see the last paragraph of [my post](https://gist.github.com/setavenger/a0cd7e71b47ded9fca9c99085130cf2a?permalink_comment_id=5931734#gistcomment-5931734)).

[quote="setavenger, post:3, topic:2203"]
brings more flexibility to client design

[/quote]

Perhaps, but I suspect it may be a little early to start considering where to be flexible. In order to come up with a good protocol we’ll inevitably have to first form a reasonably well-informed opinion about what the client design might look like. Or at least that’s the only way to stay reasonably lean.

Opinions can differ, but I’m not very optimistic about Web of Trust reliance, for instance. If we had a well-functioning and reliable WoT to slot this all into then great, but today we do not.

[quote="setavenger, post:3, topic:2203"]
Do we expect Alice to wait for the transaction to confirm before she sends the notification message?

[/quote]

The easy part of this question is that not waiting is clearly preferable to Alice. How I’d approach the rest of the question is by figuring out what protocol we end up with when we answer this with both a yes and a no and seeing how practical the end result is for Bob.

-------------------------

setavenger | 2026-02-14 15:45:15 UTC | #5

[quote="RubenSomsen, post:4, topic:2203"]
Your latter sentence is why I suggested the taproot output for the minimal approach rather than the txid. This gives you something to scan for using regular block filters. Though txid filters are theoretically possible too, of course.
[/quote]

How would that look in practice? If Bob only receives an output, he has to go through all block filters until he finds the output - every time. A heuristic to speed up this process would be to go backwards from chain tip. But this still seems impractical to me. A timestamp as a hint could make this more feasible to limit the probable range.

[quote="RubenSomsen, post:4, topic:2203"]
I’m not convinced there is a safe enough middle ground where we only partially check stuff. Either we trust the sender or we don’t. If we trust the sender then we don’t have to fetch any data from other sources at all. The sender can just provide everything that is needed to spend the output (see the last paragraph of [my post](https://gist.github.com/setavenger/a0cd7e71b47ded9fca9c99085130cf2a?permalink_comment_id=5931734#gistcomment-5931734)).
[/quote]

Why would a wallet need to go to either extreme? There are two aspects we are discussing. One being DoS resistance and the other being validation of the message. The latter is generally required in my opinion. A wallet cannot just take a notification and blindly trust that this output is spendable, or even exists. The most basic check is: "Does this output still exist and did I already receive this notification". Most or all wallets are most likely able to run the scanning logic for that transaction at the same time. I don't see a good reason, yet, to skip this validation. My assumption here is that during any basic validation the wallet would use it's default backend which would be "other sources".

Bob, despite trusting Alice, blindly adding the notification's output as an input in a new transaction sounds like a UX nightmare to me. One just needs to consider an RBF-Scenario where the transaction's inputs changed, consequently changing all SP outputs. Alice did not defraud Bob and can be trusted. But two notifications exist of which the first one does not point to valid transaction or output. I guess this is somewhat what you were eluding to in your previous reply to the Gist.

[quote="RubenSomsen, post:4, topic:2203"]
Perhaps, but I suspect it may be a little early to start considering where to be flexible. In order to come up with a good protocol we’ll inevitably have to first form a reasonably well-informed opinion about what the client design might look like. Or at least that’s the only way to stay reasonably lean.
[/quote]

I directionally agree with this. I would say, though, that for very standard data, like the tweak, I don't see a problem at all. I'd be more critical if we were talking about fringe or edge case data. The tweak is standard information which every sender has and every receiver needs. Trust assumptions referenced above.

Also about adding a tweak. I think it mainly becomes an issue if a wallet has a bad trade-off or handles a notification in a way which creates weird UX cases.

[quote="RubenSomsen, post:4, topic:2203"]
Opinions can differ, but I’m not very optimistic about Web of Trust reliance, for instance. If we had a well-functioning and reliable WoT to slot this all into then great, but today we do not.
[/quote]

Let me rephrase/clarify. I meant Web of Trust in the wider sense not the Nostr scoring system. If I receive this notification via something like Signal or E-Mail from somebody with whom I have had a previous relationship, I have to take less protective measures compared to a completely open and unchecked input channel like random Nostr DMs.

-------------------------

