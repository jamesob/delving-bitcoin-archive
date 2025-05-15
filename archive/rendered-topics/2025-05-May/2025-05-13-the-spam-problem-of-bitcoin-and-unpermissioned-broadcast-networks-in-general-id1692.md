# The Spam problem of Bitcoin and Unpermissioned Broadcast Networks in general

moonsettler | 2025-05-14 10:25:42 UTC | #1

Has the spam problem of bitcoin ever been formally defined in quasi-mathematical terms?

My intuition is that p2p value transfers inherently are at a motivation disadvantage compared to spammers who wish to reach millions or billions of people with their content. Broadcast networks simply do not provide the same value bye-per-byte to someone that wishes to transmit information person to person, probably not even in the same magnitude. In the past many people may have made the mistake of thinking about spam as self gratifying "graffiti" that nobody will see anyhow. I don't think that is true anymore. Nor do I think that "pricing out" spammers is going to work as well as people have assumed in the past. The more likely outcome seems to be that p2p value transfers will largely abandon bitcoin before spammers do.

Other currently unpermissioned networks like nostr that work with sort of broadcast principles have similar problems. Imposing cost on a per post basis is not going to be a successful deterrent, if the value derived from spam (for the spammer) is higher than the value derived form sharing thoughts in a social media setting. While nostr or a future social media network can easily adopt a different strategy for propagation than dumb broadcast, bitcoin is for better or worse stuck with it on layer-1.

Generally the internet has largely chosen to identify the users (impose a cost on identity) and impose sanctions on misbehaving instead of imposing cost on user actions to deal with spam. However this path is contrary to the principles and ethos of bitcoin. It has also led to catastrophic consequences to privacy via the extension of the surveillance state through big-tech.

-------------------------

gmaxwell | 2025-05-15 01:51:04 UTC | #2

I don't think spam is usually a useful word in the context of Bitcoin.

The normal usage, spam email, is an unsolicited message sent to you that you don't want.  You are successful in defeating it if you don't see it. The sender paid nothing or infinitesimally close to nothing to send it.  It doesn't matter if your computer, or your ISPs computer or your ISPs ISPs computer, or the sender's ISPs.... whatever sees it, stores it, etc.  If your juicy meat optics don't see it, you win.   You still win if you spent more computing power not seeing it than the spammer spent sending it so long as your cost is still infinitesimal.  It doesn't matter to you if *other* people see the spam, you still win if you don't see it.

Most of the time in Bitcoin the thing people calling spam is activity from a consenting party, to a consenting party, mined into the blockchain by a consenting miner (who got paid handsomely by the sender for their help).  It was fairly expensive for the sender to send.   The thing that distinguishes it from any other usage is that they're using Bitcoin for some collateral purpose. They want to achieve some side effect other than sending bitcoin from one person to another subject to conditions. The anti-spammer loses unless they keep this transaction out of the blockchain, and there is no mechanism to prevent a miner from adding a transaction short of it being invalid according to the consensus rules.

I think there is some cultural inertia from years ago when it was incredibly *cheap* to dump data into Bitcoin, practically free, and when it literally could be less expensive to "store" data that way rather than in a commercial service.  In that case, there were obvious incentives towards abuse. But today the Bitcoin price and competition for space is such that it's something like a hundred thousand times more expensive to store data in Bitcoin then a model price for "amazon s3 forever".  Since storage is very cheap it doesn't take completely outrageous fees to drive that usage off. So then if it expensive why is there any collateral use at all?  

For the remaining activity that people are complaining about it appears to be that space in Bitcoin is a scarce commodity that they can turn into a non-bitcoin "valuable" asset through the magic of seigniorage.  It should be obvious, but this particular use much less or even *negative* response to increasing costs or otherwise restricting resources.   But it's also utterly unlike and kind of spam something like nostr would git.

In nostr I suspect the problem you have is that most messages have at most infinitesimal value, so you can't impose more than infinitesimal costs. And so pricing out unwanted traffic doesn't work there-- but it does work in Bitcoin, it just doesn't price out seigniorage because it doesn't care or even likes the constraints.

One solution to seigniorage is to explode its scarcity. Early in Bitcoin there was a wave of people copying the bitcoin code to create altcoins and then behaving kind of abusively towards the Bitcoin community to promote their altcoin.  One community member just created a website where anyone could push a button and create an altcoin... eventually it was bought for 10 BTC by an altcoin producer only to shut it down, but its work was done and altcoins that were *just* a copy of bitcoin were less of an issue.  

The same kind of thing might work for the seigniorage but that depends on how much the demand for the tokens is real vs the whole thing just being a cover for money laundering (clean account issues/obtaines tokens, dirty money account buys them at crazy prices).

There does exist an analog of email spam in Bitcoin,  "dusting" ... but it's seldom discussed.  And there are obvious countermeasures possible to reduce it (e.g .wallets should hide really tiny payments, wallets should avoid showing any kind of "sending address", etc).

-------------------------

garlonicon | 2025-05-15 05:58:29 UTC | #3

[quote]While nostr or a future social media network can easily adopt a different strategy for propagation than dumb broadcast, bitcoin is for better or worse stuck with it on layer-1.[/quote]
Have you ever thought about Layer Zero? It is possible to not only go downwards, and make networks like sidechains, which are definitely Layer Two, or Lightning Network, which is something like "Layer One and a Half", but it is also possible to go upwards as well, and make a network, where Bitcoin mainchain will be a subnetwork of.

Which means, that if you have for example Lightning Network, then you have to process the mainchain, and your LN transactions. Which means, that resource-wise, you see and process more transactions than usual, because the cost of handling it, is the accumulated cost of running a Bitcoin node, and some LN node.

But: it is technically possible to do it differently. You can have a network on Layer Zero, which will contain less transactions, than the Bitcoin mainchain, and where Layer One transactions will commit to. And then, you can look at the current Bitcoin Core code as a combination of Layer Zero node, and some Layer One additional code. Which means, that you can process less data, and stay in the same network.

So, to sum up, the answer to your question "how to deal with spam" is "just compress it, just simplify it". Then, spammers will still stay on Layer One, when they will still push their data, as much as they want, but you, as a user, can jump to Layer Zero, and enjoy a more useful network in practice. I think the main reason, why Layer Zero was not yet made, is that people are lazy, and nobody was really forced to do so in practice.

-------------------------

