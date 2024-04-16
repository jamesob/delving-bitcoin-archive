# Can Game Theory Secure Scaling?

hynek | 2024-04-13 17:18:42 UTC | #1

Hi all,

I'm a bit frustrated with the trend of using custodial lightning wallets. It’s simply cheaper to use custodians these days... and I believe that it isn’t necessary. The crucial part of sovereignty is the ability to sign a message, which doesn’t incur any costs. Therefore, I believe there is a way to make self-custody both fast and cheap.

There have been a lot of new proposals in the last year, so I kind of doubt you're interested in thinking about another one... Still, I'd like to try to get some feedback on my proposal, which I'm calling "Last Mile."

The basic idea is to use classic bitcoin transactions without broadcasting them unless there's a dispute. Honest behavior is encouraged through calibrated incentives, in simple terms, making theft uneconomical.

And here is my current way of thinking:

https://github.com/hynek-jina/Hynek/blob/0301c90015f9d66fb22e347f21924dc9d04f0bc8/blog/Last%20mile%20transactions.md

*This approach is somewhat naive, but maybe there is some potential. Perhaps it will motivate you to help me improve it into something even better.*

I would love to read your feedback!

-------------------------

garlonicon | 2024-04-15 05:14:32 UTC | #2

[quote]*This approach is somewhat naive, but maybe there is some potential.*[/quote]
It may seem naive, but when I thought about sidechains, my conclusions were similar. Definitely, if you want to make a test network, then it is sufficient to just prepare all transactions off-chain, without moving any coins at all. Because then, a simple test network can be deployed: it starts with zero coins, and they appear on the network if you sign them, and disappear, when you move them on-chain in any way.

And obviously, that model is not "the final state of things", because if Alice knows all conditions behind a given UTXO, then she can always move them on-chain, no matter what was signed previously, and broadcasted to any parties. However, if you note that you can hide N-of-N multisig, behind a single public key, then you can see some potential in this idea.

So: my conclusion was simple: if you want to build any network in this way, it should have general rules like that: "sign on-chain coins to make a peg-in, and move on-chain coins to create a peg-out". And that rule should not be restricted any further, when it comes to checking, if the history is valid. However, to prevent misuse, you can then restrict it further by introducing standardness rules. Which means, that peg-ins and peg-outs of the whole network, should work even with OP_TRUE. But: the client you interact with, can for example say: "you have to show me some on-chain UTXO, where there is P2TR, with 2-of-2 multisig, and one of those keys are owned by me".

[quote]Alice will need to disclose information that a given UTXO is partially pledged to a particular bitcoin address. This information can be shared with nodes that would have the assigned counter addresses in addition to a list of valid UTXOs. Or, for example, Alice can share this information via nostr on generally know relay (or more of them).[/quote]
Guess what: this is the perfect task for the sidechain network. Imagine that some of the current Bitcoin nodes would have additional features, like "storing penalty transactions". And then, those nodes could act as a global watchtower. But: to complete the whole picture, you need to note, that if all transactions are public, then someone may just broadcast all of that unconditionally, even if there is no dispute. So, to mitigate that, you should also encrypt penalty transactions, so they will be decrypted and broadcasted, only when nodes encounter the previous transaction.

[quote]How to prevent the long-term collaboration with miners?[/quote]
If they will run their full nodes with those new features, then it would be sufficient. Of course, there is always a risk, that some miners may alter the default settings, but well: any second layer can be always attacked, if miners are malicious. So, it is all about ensuring, that the majority is honest, and share your software with that majority.

-------------------------

ProofOfKeags | 2024-04-15 19:58:05 UTC | #3

I think it's worth noting that there are two main reasons that running lightning and similar off-chain deferred settlement protocols. First is the issue where you need to monitor the chain for breaches of contract and respond in kind. Secondly it requires interactivity in order for a transfer to complete. Both of these things require the persistent liveness of an agent operating on your behalf. This is what makes running these off-chain protocols harder, not the signing itself.

Doing *more* deferral as in your proposal (I admit I've only skimmed it so feel free to correct me if I misunderstand) doesn't do anything to alleviate this problem and in fact exacerbates it to a (modest) degree.

If you are interested in removing the need for custodial lightning, you need one of two things. Either an extremely simple way to spin up an agent that works on your behalf in this way (this is what products like [Start9](https://marketplace.start9.com/lnd) offer), OR you need protocols that eliminate the liveness requirement for receiving money and responding to potential breaches.

Indeed it is already the case that the game theory of lightning that dishonesty is disadvantageous. Broadcasting a prior state opens you up to a complete loss of all channel funds. However, to guarantee this in practice, you must be able to respond to breaches. Extending this assumption to the funding transaction doesn't do anything to actually alleviate the pressures that nudge people towards custodianship.

-------------------------

hynek | 2024-04-16 19:50:43 UTC | #4

[quote="garlonicon, post:2, topic:797"]
Alice knows all conditions behind a given UTXO, then she can always move them on-chain, no matter what was signed previously
[/quote]

Sure, she can. But it's not beneficial to her.

[quote="garlonicon, post:2, topic:797"]
However, if you note that you can hide N-of-N multisig, behind a single public key, then you can see some potential in this idea.
[/quote]

Could be interesting.. but here the initial idea is that you don't need any on-chain preparation.

[quote="garlonicon, post:2, topic:797"]
And then, those nodes could act as a global watchtower.
[/quote]

It would be cool.. but the idea here is to keep this information for peers only because I don't know how to prevent unwanted broadcasting.

[quote="ProofOfKeags, post:3, topic:797"]
If you are interested in removing the need for custodial lightning, you need one of two things. Either an extremely simple way to spin up an agent that works on your behalf in this way (this is what products like [Start9](https://marketplace.start9.com/lnd) offer), OR you need protocols that eliminate the liveness requirement for receiving money and responding to potential breaches.
[/quote]

I admit that these two params are slightly worse at Last mile compared to LN. However that isn't what I'm trying to improve primary. My main ambition is to extend the number of sovereign users without polluting the time-chain. Making it accessible to more people.

-------------------------

