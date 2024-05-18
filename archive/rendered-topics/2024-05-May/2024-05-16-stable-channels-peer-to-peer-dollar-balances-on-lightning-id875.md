# Stable Channels - peer-to-peer dollar balances on Lightning

tony | 2024-05-16 17:49:21 UTC | #1

**Stable Channels - peer-to-peer dollar balances on Lightning**

**1. Introduction and background**

Good day, folks. I’d like to present an open-source project that I am working on called Stable Channels. The main goal of this project is to bring bitcoin-backed dollar balances to the Lightning network. Another goal is to bring more bitcoin and interest into Lightning. 

Find this project on Github [here](https://github.com/toneloc/stable-channels/ ) and on Telegram [here](https://t.me/+jvZKrdM6XZFjZGMx).

The TL;DR is that Stable Channels matches up BTC shorts (fiat stability seekers) and BTC longs (fiat stability providers) over Lightning Channels. Then, based on the updated BTCUSD price at each checkpoint, we settle sats very frequently so that each side gets what they want. In doing so, we create a bitcoin-backed synthetic dollar balance on the stable side of the channel. The other side is leveraged long. 

Stable Channels addresses a major impediment to bitcoin adoption: price volatility. In contrast to bitcoin, centralized stablecoins give users a stable store of value, and stablecoins are a huge and growing market. Despite this tremendous success, centralized stablecoins face systemic risks. Centralized stablecoins have external dependencies on centralized banks and bond custodians. They run the risk of hacks, bank runs, asset freezes, asset seizures, and forced closures. 

We can and we should offer better stability tools on decentralized bitcoin-only rails. What’s more, we can do this while respecting the bitcoin ethos of self-custody. These decentralized solutions, like Stable Channels, will need to take a much different form from centralized stablecoins if they are to survive and thrive over the long term.

A few things are important to mention before we dig into the mechanics of Stable Channels. First, this idea might sound familiar to you. As far back as 2015, Arthur Hayes [blogged](https://blog.bitmex.com/in-depth-creating-synthetic-usd/) about creating synthetic dollars by matching up longs and shorts on the BitMex exchange. In 2019, Dan Robinson theorized about Rainbow Channels [here](https://www.paradigm.xyz/2019/03/the-rainbow-network-an-off-chain-decentralized-synthetics-exchange) and [here](https://www.youtube.com/watch?v=Dl9eUp28R7k&t=1770s). Rainbow Channels would use “continuous settlement” to let a user “trade, borrow, lend” over payment channels. And in the bitcoin space today, innovative companies like [10101](https://10101.finance/) use DLCs to achieve something similar.

The second thing to mention is this: if we are trying to keep a pot of bitcoin stable in dollar terms, we may be doomed to a fuzzy approximation. Why is this? It’s because the market price of bitcoin moves in seconds and minutes, but bitcoin settles with finality only in minutes and hours. Even with the best of intentions, by the time a settlement transaction is confirmed the price of bitcoin may have changed a lot. Of course, Lightning can help with this. 

Third and finally, this is a complex subject. I won’t cover everything here, but I look forward to starting the conversation and appreciate any feedback. 


**2. Stable Channels: the basic proposal** 

So let’s get to it. What do I propose? As mentioned, the approach with Stable Channels software is to settle sats continuously, minimizing your counterparty risk at any given checkpoint. 

There are two types of users in a Stable Channel: Stable Receivers and Stable Providers. Each of these two types of users run the Stable Channels software, which runs alongside their Lightning node or else can securely interact with it.

1. **Stable Receivers**. **Stable Receivers** want **less** Bitcoin price exposure versus a reference asset, like the dollar. *Example*: Alice has 1 BTC and wants less BTC price volatility / exposure versus the dollar. Alice wants to use her 1 BTC to lock in $62,000 stable USD exposure (let’s assume the current price is $62,000).
2. **Stable Providers**. **Stable Providers** want **more** Bitcoin price exposure versus a reference asset, like the dollar. *Example*: Bob has 1 BTC but wants even more BTC price exposure against the dollar. Bob wants to use his 1 BTC as margin to leverage up his bitcoin exposure. 

We match up these users over a Lightning Channel. Alice and Bob each put in 1 BTC and we open a dual-funded channel. That’s 2 BTC total capacity in a balanced channel. 

Now we are ready to go to the regularly scheduled programming of the Stable Channels software. Every one minute, the program queries the same five different price feeds: BitStamp, CoinGecko, Coindesk, Coinbase, and Blockchain. We take the median price to protect against one or two of them being way off or compromised. 

After we get the latest price, one of three things happens:
1. The BTC price went **down**. Then it takes **more** BTC for $62,000 stability. The Stable Provider pays the Stable Receiver over the channel. 
2. The BTC price went **up**. Then it takes **less** BTC for $62,000 stability. The Stable Receiver pays Stable Provider over the channel.
3. The price **didn’t move**. No payment is required.

This settlement process goes on continuously. Assuming the actors are basically cooperative, then on the Stable Receiver side of the channel we have created a synthetic dollar balance. Each side gets what they want: the Stable Receiver gets stability and the Stable Provider leverages their bitcoin. Of course, each side can lose bitcoin if the price goes against them. And each side can win bitcoin if the price goes their way. Nice.

Note that these are vanilla Lightning channels. For better or worse, this process involves no DLCs, no oracle digital signatures (besides HTTPS to fetch prices from exchanges), no tokens, no modifications to the Lightning contracts / scripts, no fiat, and no banks. Each side remains self-custodial and can leave at any time. As this is an ongoing and voluntary process, there are obvious trade-offs involved, which we shall explore presently. 

**3. Implementation challenges and opportunities**

The program works today for CLN as a Python plugin and for LND as a standalone Python app. The primary functions of [the code](https://github.com/toneloc/stable-channels) are handling the price feeds, handling the core payments and accounting logic, and risk management. 

Experience over the past half year shows that this solution faces similar UX and operational challenges as bitcoin and Lightning today. I will mention these briefly to this well-informed audience: you need to run a Lightning node, the Lightning software is complex, you need to be always online, lots of subtle edge cases, bulky backups, hot private keys, potentially high fees, etc. On the bright side, this solution should benefit from future LN upgrades and updates that may ease these UI/UX pains. 

Next, let’s discuss some more risk factors, attack vectors, and possible mitigation techniques. First, note that even between perfectly cooperative actors, if the price of bitcoin plummets then there will not be enough bitcoin for the stability mechanism to work. For example, if each user put in 1 BTC at $62,000 and the price of bitcoin goes down by more than 50%, then now there is less than $62,000 *total* channel capacity, not enough to keep anyone stable at $62,000. For cooperative actors who want to continue this trade arrangement, perhaps a splice-in could handle this margin-call-like scenario. On the flip side, if the BTCUSD price rises, then there is plenty of bitcoin to keep the Stable Receiver user stable in dollar terms. On a much more mundane note, a cooperative actor’s node could simply go offline; this happens all the time on Lightning. 

What about risks between noncooperative actors? Noncooperative actors could take advantage of this system in several ways. Most predictably, they could wait until the price goes in their favor, then not pay when the price suddenly goes the other way. A few characteristics of Stable Channels countervail this risk. First, the attacker needs to pay to play. An attacker needs to lock money in a channel in the hopes that such a meaningfully profitable opportunity arises. The attacker also needs to pay on-chain transaction fees. Second, in the event of an uncooperative close, the attacker may need to wait days or weeks to get their bitcoin out of the channel. Third, the attacker may need to pay an ongoing service fee or interest rate (more on this later). And fourth and finally, there is always the risk that the attacker himself is attacked.  

Practically speaking and with these failure scenarios in mind, Stable Channels make the most sense between decently cooperative and reputable channel partners. This is basically the case with the Lighting Network today. The Lightning Network uses reputation, identity, and track record as measures of channel counterparty trustworthiness. This may also prove to be the case for Stable Channels, with reputation aiding cooperative behavior and standards arising for trade terms and duration. 

Another approach to mitigate the risk of sudden channel partner misbehavior is to use moving average prices instead of spot prices. This could smooth out the herky-jerky nature of intraday bitcoin price movements and thus lessen the financial damage of missed settlements. Moving averages could also lessen the required  frequency of settlement. A different approach could be to shorten settlement periods even further and further. Yet another simple tactic is to diversify trade / channel counterparties to minimize the impact of any single uncooperative channel partner. Finally, “virtual Stable Channels”—that is, bilateral trade arrangements between Lightning nodes that are not directly connected, perhaps in an opt-in subnetwork—may provide a more fluid solution, provided that transaction fees do not eat too much into the channel balances. 

How big an issue the ongoing “free option” problem proves to be will depend on particular market structure and implementation details. The overall point of Stable Channels is not to eliminate counterparty risk, but to minimize it with the frequent settlement capabilities unlocked by Lightning. The basic idea is to keep it simple such that in the failure mode, you still have your bitcoin. 

One other unanswered question is: “How will Stable Receivers and Stable Providers find each other?" Currently, there is no marketplace matching up Stable Receivers and Stable Providers. These marketplaces could be centralized, like a website or mobile app UI. Or they could be decentralized, using messages like Liquidity Ads on Lightning or nostr notes.  Or all of the above. All of these messaging and orderbook protocols are relevant and various platforms developed in parallel may make sense.

**4. Further analysis and use-cases**

This section gives more context on the finance side of things. This part gets complex; skip it if you wish. 

First, note that the reference asset in which we keep a bitcoin balance stable could be other things besides the US dollar. If an asset has reliable price feeds and plausible liquidity, then it might be a good candidate for a Stable Channel. For example the Stable Receiver may want to be stable in euros, gold, or the S&P 500. Note also that we could offer various bitcoin-focused products besides simple stability versus the spot BTCUSD price. For example, moving averages could offer users some of the upside of bitcoin, while minimizing their downside. Or more of the upside, while also getting more of the downside. We can think of these products as “Bitcoin Lite” on the stable-ish side, or “Bitcoin Heavy” on the slightly leveraged side.

Even in the same channel, we can change or recharacterize channel holdings if both counterparties are up for it, giving us trading use-cases. Let me explain. If the Stable Receiver Alice was originally kept stable at $62,000 ... another way to think about that is that she is kept stable “$62,000 and 0.0 BTC.” She could send a signed message to the Stable Provider to recharacterize or reclassify her holdings of these two (or more) assets. This new message to the Stable Provider Bob would say something like “Please recharacterize my holdings to $31,000 and 0.5BTC."  As “$31,000 and 0.5BTC” still comprises 1BTC total on Alice's side, Bob may be okay with it. He may charge a trade processing fee. If so, the recharacterization processes and— voila—Alice has “bought” 0.5BTC.

Overall, these trading use-cases could bring more liquidity to Lightning. Consider that derivatives trading on the BTCUSD pair is a very large market several multiples higher than the BTCUSD spot market. Most of the time, it seems to me, traders use centralized and custodial derivatives exchanges to put on these trades. These users will inevitably get burned when these custodial exchanges blow up, get hacked, or get shut down. What’s more, these users pay a funding rate to trade on margin with their bitcoin deposited to these exchanges. Normally, 72% or more of the time, longs pay shorts on these exchanges. These interest rates are most often between 10%-20% annualized (there is a nice article on this phenomenon on the [Axiom website](https://www.axiombtc.capital/orange)). 

With Stable Channels and Lightning, we could add in these interest rates payments into the regularly scheduled programming. This interest rate could help modulate the demand for these channels and incentivize use of them. This rate could be split up into small chunks of sats and paid to the Stable Receiver every settlement period. 

Of course, we can also plug in external payments for users. I did this in the latest version of the Core Lighting plugin. The code listens to payments getting routed out of the channel and decreases the amount held stable accordingly. However, note that payments in and out open up arbitrage and attack vectors.

One other interesting integration we have done is with use of the Chaumian eCash system Cashu. Developer Star Builder, others, and I helped build an eCash stable dollar token whose value is secured by a Stable Channel. You can check out a nice UI for this on [Boardwalk Cash](https://boardwalkcash.com/wallet) and check out the eCash mint with data from the Stable Channel on the [Umint mint website](https://umint.cash). 

With eCash, a version of [proofs of liabilities](https://gist.github.com/callebtc/ed5228d1d8cbaade0104db5d1cf63939) can be provided, and a type of proof of reserves can be shown by identifying the on-chain channel funding transaction for the Stable Channel and providing signed attestations to the channel state from both the Stable Provider and Stable Receiver.  Check out early versions of this on the Umint website linked above. 

An eCash system provides many advantages over the all-Lightning approach. Users don't need to run a Lightning node or be online all the time like with Stable Channels. There are scalability and UX advantages, and we can get up and running quickly. However, eCash systems are custodial and at risk of a rugpull or regulatory action. Fedimints could help with the custodial challenges. Finally and obviously, eCash tokens, like Stable Channels, still need to get market buy-in. 

**5. Conclusion** 

Stable Channels is trying to reimagine Lightning capacity as collateral for continuously held derivatives positions. It is streaming finance, with bitcoin as the technological backend, the collateral, and the medium of exchange. Because stablecoins and derivatives trading are much larger markets than Lightning payments alone, adoption of Stable Channels could help expand the capacity and utility of Lightning. 

When I mention this idea, people have often said to me, “Ah, this is a contract for difference.” Or “It’s a futures contract, a perpetual swap, a non-deliverable forward, a whatchamacallit …” Well, Stable Channels may be similar to some of these things, but it's not any of them. Stable Channels are bitcoin-backed and settled on Lightning and these characteristics bring their own peculiarities, which is obvious to anyone who has worked with Lightning. These peculiarities will need to be dealt with in their own ways by bitcoin and Lightning developers. 

This will be a challenge to build out. In my experience, however, the software we’ve built so far has worked well between cooperative actors on reliable servers. This is a testament to years of focused and incremental improvements to Lightning by its developers. Kudos to you. 

*Thank you to several contributors for their input on drafts of this post.*

-------------------------

1440000bytes | 2024-05-16 22:29:28 UTC | #2

[quote]
One other interesting integration we have done is with use of the Chaumian eCash system Cashu. Developer Star Builder, others, and I helped build an eCash stable dollar token whose value is secured by a Stable Channel. You can check out a nice UI for this on [Boardwalk Cash](https://boardwalkcash.com/wallet) and check out the eCash mint with data from the Stable Channel on the [Umint mint website](https://umint.cash).
[/quote]

![image|690x133](upload://uiIUPVjmLjmQwKXFhaihg5rhjND.png)

This warning text on Boardwalk Cash website is misleading as eCash is not self custodial.

[quote]
With eCash, a version of [proofs of liabilities](https://gist.github.com/callebtc/ed5228d1d8cbaade0104db5d1cf63939) can be provided, and a type of proof of reserves can be shown by identifying the on-chain channel funding transaction for the Stable Channel and providing signed attestations to the channel state from both the Stable Provider and Stable Receiver. Check out early versions of this on the Umint website linked above.
[/quote]

- Users still need to trust the mint and stats shared by them publicly
- Not good for privacy

[quote]
An eCash system provides many advantages over the all-Lightning approach. Users don’t need to run a Lightning node or be online all the time like with Stable Channels. There are scalability and UX advantages, and we can get up and running quickly. However, eCash systems are custodial and at risk of a rugpull or regulatory action. Fedimints could help with the custodial challenges. Finally and obviously, eCash tokens, like Stable Channels, still need to get market buy-in.
[/quote]

The drawbacks you mentioned outweigh any benefits custodial eCash offers.

-------------------------

tony | 2024-05-18 16:38:35 UTC | #3

That's good feedback, thank you. 

It will be an ongoing challenge to communicate engineering trade-offs to users. Careful language is important for that. 

The hope is that well-informed users can choose between self-custodial Lightning and custodial systems like ecash in an open market with lots of varieties and competition.

-------------------------

