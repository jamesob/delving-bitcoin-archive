# Response to Pieter Wuille's StackExchange Answer Re: Nuking the Opreturn Filter

cguida | 2025-09-17 19:13:07 UTC | #1

(Also posted on twitter: [https://x.com/cguida6/status/1968117953871720463](https://x.com/cguida6/status/1968117953871720463))

(Also posted on nostr: https://njump.me/nevent1qqsffyx0zm37748cskqjka634vmslywt0qf67hhd8e02cj49xvksylcpz4mhxue69uhhyetvv9ujuerpd46hxtnfduhsygpgjhpnps3l8qceds80nzx7dk5rhqa5tqldt7wpakc22kwwe50eqqpsgqqqqqqsj759xr )

Several core supporters have pointed me to Pieter Wuille’s recent StackExchange post\[0\] explaining core’s rationale for raising the default opreturn limit 1250x from 80 bytes to 100,000 bytes in the upcoming bitcoin core v30. I don’t know where else to publish this response, because I don’t know what apps Pieter uses, and StackExchange does not seem like the right place for a constructive back-and-forth.

(Edit for Delving: Thanks to @moonsettler for reminding me that @sipa uses Delving. I think Delving is probably the right place for this discussion to take place, if indeed it is a discussion and not just a single one-way post with no followups, as I have experienced in the past on here. I did not post this response to StackExchange as it is designed for exactly the kind of 1-question-1-answer dynamic I am trying to avoid).

My goal here is to challenge Pieter’s explanations with claims and questions of my own, in order to:

1. find any pieces of Pieter’s rationale that I am misunderstanding or lacking,
2. inform the public as to what I believe to be the most compelling arguments against removing the opreturn limit, and
3. ultimately, to arrive at the real truth of the matter and help bitcoin follow an optimal path

So I’ve taken Pieter’s post and responded inline here (paragraphs with double arrows are the original SE question asker, those with single arrows are Pieter, and those without arrows are me):

> Given the debate around the issue I think this is probably an opinion-based question, but I’d still like to try to clear up some misconceptions.
> Disclaimer: I’m a Bitcoin Core contributor, argued in favor of dropping the OP_RETURN restriction, and co-authored a document on the matter that was signed by several other contributors and got published on the [http://bitcoincore.org](https://bitcoincore.org) website (which I recommend you read). While I’m only going to give my own view below, I think I have a good idea on at least the proponents’ opinion on the matter.

Yep, you definitely seem to be the most authoritative voice on the “pro-32406” side of the debate. You are also obviously one of the most important devs in bitcoin history, and your claim to being “a Bitcoin Core contributor” is a massive understatement.

> > I agree that Bitcoin is money, not a storing database, and strictly only monetary data should be on the blockchain.
>
> I agree with this, and have argued strongly in the past against developing solutions that relied on using the chain purely as a publishing mechanism for non-financial data. Blockchain space can be expensive, slow, unreliable, and costly (in terms of bandwidth/processing by nodes). It should be used for what it’s good for: proving that censorship-resistant payments are valid to the world; a use case for which no better alternative exists, as opposed to many data-publishing use cases that have no such requirements.

Yes, I’m glad that we agree on this.

> I believe that in the long term, such (in my view) sub-optimal use will be driven to alternative solutions anyway as on-chain space grows more valuable, and that deviations to this, like we see now, are temporary convenience solutions driven by hype.

I think this is where we disagree. Greg Maxwell argued\[1\] in 2015 that “Demand for cheap highly-replicated perpetual storage is unbounded”. I agree with Greg. If we do not impose any additional costs on people trying to use bitcoin as a general-purpose database, such use will \*absolutely\* drown out payment use cases, especially merchant Lightning nodes which, at the moment, are bitcoin’s most important payment use case.

I think filters make it \*look like\* there is no demand for large opreturns, just as the fence around your house makes it \*look like\* there is no demand for trespassing on your property. Deterrents are often misunderstood, because you can’t see all the attempts that never happened, \*precisely because of the deterrent\*.

> It’s worth warning people that they likely won’t persist with changing economics, and provide some discouragement for it - but not when that discouragement becomes harmful on itself (see further).

I’m not sure what you mean here by “they likely won’t persist with changing economics”… if your assertion is that altcoin Ponzis will magically stop happening \*just\* because of on-chain fees… I don’t think that’s likely true at all; one look at the “crypto industry” outside bitcoin should be sufficient to explain why. But maybe that’s not what you were saying?

> However, as developers and community of node runners, we also do not really get to decide what people use the chain for, beyond agreeing on consensus rules(\*).

This is another point of disagreement. The noderunners \*absolutely must\* enforce bitcoin’s usage as a payment network and reject attempts to use bitcoin as data storage since, as I already mentioned, demand for perpetual, highly-replicated storage is infinite, and therefore there is no way the consensus rules \*themselves\* can prevent data use cases from drowning out payment use cases. Even with a fork to disallow almost all script types, there would still be people making crazy schemes on top of fake pubkeys or whatever in order to facilitate their token Ponzis. Ponzi participants are not particularly sensitive to fees, if the promise of getting rich quick is compelling enough.

Put more simply: bitcoin’s identity as money cannot be enforced at the consensus layer. Therefore, the only way to ensure that bitcoin stays money is if we enforce its usage as money at the social/mempool layer. The only reason bitcoin has stayed money thus far is because of its protective “bitcoin maximalist” culture, which ensures that its #1 priority is always to be the best money, and that other use cases are severely discouraged.

In the absence of strong mempool filtering (and possibly other social-layer mechanisms), I don’t see a way for data use cases \*not\* to gain such a dominant role that bitcoin ceases to be money entirely. It just seems inevitable to me.

The reason filters are effective at deterring unwanted data use cases is that the filter serves as a warning that is backed up by the noderunner community’s will. The warning says: “we don’t like transactions that store more than 80 bytes of arbitrary data, so if users attempt to work around this, we will take action to mitigate such usage. If our mitigations don’t work, we will continue escalating until the spammers leave.” This is exactly what we did in 2014\[2\], and is exactly what will work today.

> The contents of blocks is decided by miners, who are - by design - driven by economic incentives.

If you are implying that fees are the only “economic incentives” miners pay attention to, this can be easily disproven by considering the result of the block size war; larger blocks promised \*much\* larger fee revenues for miners than small blocks (and presumably number-go-up in the short-term because of perceived “scaling”), and yet the small blockers were victorious. Miners were forced to concede that a bitcoin with smaller blocks and segwit would be much more economically valuable in the free market than a bitcoin with large blocks and no Lightning Network.

(Similarly, a bitcoin that is easy for merchants to accept directly because their Lightning channel opens and closes aren’t being overrun by altcoin speculative manias is much more valuable in the free market than a bitcoin that is filled with spam.)

That is: miners must also pay attention to long-term incentives in addition to short-term incentives. Sometimes it takes some nudging from the user community in order to help them decide, but eventually the market will always converge on whichever version of bitcoin is the most valuable, and miners must pay close attention in order to make the right choice.

> And if people are willing to - hopefully temporarily - pay more for “dumb” data storage on chain than real payment activity, there is little that can be done about this, for a multitude of reasons:

> Data can always be disguised as payments,

Yes, this is precisely why the consensus rules are useless for making sure bitcoin stays money, and why we must react at the policy level rather than the consensus level. Only the most popular formats, which by definition are easily-identifiable, need to be filtered.

> making attempts to stop or even discourage it a cat-and-mouse game, which at best pushes those who pay for these use cases to slightly less efficient techniques (like encoding the data in addresses, amounts, signatures, …).

No, at best the spammers \*leave bitcoin entirely\* and go to other chains, as they did in 2014 (again, see \[2\]). We won the cat-and-mouse game last time; why assume we would lose this time? This assertion has no evidence.

> Given enough economic demand for data storage, nothing prevents submitting transactions directly to willing miners (which fundamentally can’t be prevented(\*)), bypassing non-mining nodes, the software they choose to run, and thus ultimately, their influence entirely.

Of course, but the goal of spam filtration is not \*the complete elimination of spam\*; the goal of spam filtration is just to make the system usable for its intended purpose. On bitcoin, this means making sure bitcoin stays money by making sure that the majority of activity on the network is payments, not data. The opreturn filter is performing this role admirably, as can be seen by the 99% reduction in large opreturns\[3\]\[4\] it causes.

> Philosophically, would you want a Bitcoin in which some (significant, but far from universal) subset of node runners get to decide which transactions are allowed or not, given that that, if effective, the same mechanism could be used to block “undesirable” payments, while providing a censorship-resistant payment system was the very thing Bitcoin was designed to enable?

This argument doesn’t make sense to me. You just argued that users can always submit transactions directly to a miner. So bitcoin is still censorship resistant, even if the relay network is entirely hostile nodes, entirely blocksonly nodes, or even if the relay network doesn’t exist at all. So I don’t see how a supermajority of noderunners running filters does anything to change bitcoin’s censorship resistance. Bitcoin is censorship resistant, period. So spam filtration is an entirely different matter from censorship.

Second, if a supermajority of noderunners is running a filter, \*this is the only mechanism by which the user community can enforce bitcoin’s moneyness\* since, as previously pointed out, bitcoin’s moneyness cannot be enforced at the consensus level. Thus, if a filter is being run by a supermajority of nodes (assuming no long-term sybil attacks), this must be considered the bitcoin network’s will, and going around it should be considered abusive. I very much doubt that a supermajority of noderunners could be convinced to filter OFAC transactions, for example. But again, if that does somehow happen, there’s always direct-to-miner submission APIs (and other more decentralized solutions such as gudnuf’s hackathon project, memcooler\[5\]).

> Of course, ultimately the agreement of all Bitcoin users (the consensus rules, see below(\*)) determines what transactions are allowed, but this being hard to change is a feature, not a bug.

Again, if we require consensus changes in order to prevent spam, we will obviously lose to the spammers. There is no way consensus rules can keep up with rapidly evolving threats from Ponzi scammers. But modular filters\[6\] can easily keep up with such threats.

> > I still find it hard to understand the argument on favor to this change. As far as I understand the OP_RETURN will be relaxed from 80 Bytes to 100k bytes, allowing the storage of arbitrary data, like images and even small videos. Why ?
> > I hear often “Arbitrary data is already been placed on the blockchain”, but then why the complacent attitude towards it ? and not fight the spam instead ?

> To be clear, Bitcoin Core v30 lets node runners configure the OP_RETURN size limit (the -datacarriersize config option). The only change is that its default value is increased (to effectively unlimited).

The change of the default is the important feature of this discussion. The fact that the control was left in is not significant in comparison. Up until recently, core had >95% share of the nodes, and this is more than sufficient for the default to be very sticky. Nodes that dissent from the default don’t have much effect on the actual incidence of transactions that get confirmed.

> It is marked deprecated, which means the option is expected to be removed in a future version, but given the current political dispute around it, I do not expect that will happen any time soon.

It is remarkable that it was even considered to remove the filter entirely. This would have meant that any miner wanting to filter large opreturns would need to either stay on core v29 forever or switch to knots. As it is, any miner wanting to disallow transactions with \*multiple\* opreturn outputs will have only these options once v30 is released. As is, the change is remarkably heavy-handed, but it would have been even more heavy-handed to remove datacarriersize entirely.

> The reason for changing the default isn’t just that OP_RETURN is already used, and data storage through other means is already possible (including through ways that are cheaper than through OP_RETURN); it is that attempting to address this at the relay level is not more than a mild discouragement

According to the data, nonstandard opreturns are 99% less popular than standard ones. Calling this “mild discouragement” might be the understatement of the century.

> and given enough incentives to bypass it (as we’ve seen exist)

We’ve seen a few nonstandard transactions slip through here and there, but brc20 and runes were the worst recent spam attacks and both were built on standard transactions, not nonstandard ones. It is very doubtful that a token Ponzi launching on top of a nonstandard transaction type would be very popular.

> is harmful on itself, both for the ecosystem at large, and for individual node runners, in addition to being ineffective.

We’ve already established that they are effective for their purpose, which is severe ratelimiting, but let’s hear your reasoning for why they are harmful.

> The reason it is harmful for the ecosystem at large is mining centralization. Given enough demand for transactions that the network of nodes shuns (for whatever reason), large miners are incentivized to develop means for users and companies to submit these shunned transactions directly to them (this is already happening to some extent).

This is true only if mining nodes largely disagree with the rest of the network about which transaction formats are harmful. The only time the miners should win this disagreement is if the node network is being sybiled by an entity attempting to censor. The rest of the time, miners should heed the will of the supermajority of users, and miners should be considered hostile entities if they continue mining large volumes of transactions the rest of the network has deemed abusive.

> If these “out-of-band translation relay rails” end up becoming economically relevant enough (through the out-of-band fees they involve) that miners not using them become uncompetitive, the ecosystem has a huge problem, far bigger than some temporary JPEG hype drivel: the inability for new small miners to enter the ecosystem (because who would bother sending transactions to a tiny miner)

This seems to be a point of contention between the core and knots sides of this debate; personally I am skeptical that this is as big of a problem as you seem to, but I assume you have data/reasoning to back this up, and I would be interested in seeing it.

The question is: how big of a mining centralization risk is opreturn being left at 80 bytes, and why wasn’t the opreturn filter a mining centralization risk the entire decade it’s been in existence? Why suddenly now is there a huge mining centralization risk? Are there other filters that are mining centralization risks? Perhaps limits on future script versions or transaction size? If opreturn can suddenly become a mining centralization risk after a decade of minding its own business and doing exactly what it’s supposed to be doing, what other filters might do the same thing?

You seem to think that the risk is bad enough to risk bitcoin ceasing to be money by arbitrary data making bitcoin difficult or impossible to use for payments, so it must indeed be a large risk. I have repeatedly asked core devs and other supporters what problem/risk PR 32406 is attempting to solve\[7\], and no one seems to have an answer other than “Citrea is going to put 64 bytes into the utxoset”, which is of course incoherent.

> especially anonymously (the best protection the ecosystem has long term against miner censorship).

This can be done by identifying miners by their bitcoin addresses; see memcooler\[5\] and Ocean mining’s user account model, plus tracking which blocks were previously mined to which addresses, for how this could be done.

> In addition, it incentives the development of one or a few centralized “payment submission network” companies that all big transaction creators and miners contract with (something we’ve seen in other blockchains), imposing a large censorship risk on this company, and making the entire goal of decentralization a shim of what it was supposed to be.

I’m curious what would stop this from happening if the government decided to sybil the bitcoin mempool relay network in order to attempt to censor certain individuals from transacting. You seem to suggest that direct miner submission schemes are \*always\* a bad thing, but it seems to me they could come in handy if there are ever any \*real\* censorship attempts.

> The reason it is harmful for individual node runners is that the entire point of participating in transaction relay is having a reliable view of what transactions will be mined in the future

You appear to be forgetting that mempool relay’s \*primary\* purpose is to relay payments to miners. Most noderunners I know are proud of providing this valuable service to the bitcoin network. This is why the suggestion by some core devs to “just run blocksonly” is confusing.

> for at least these reasons:
> For performing decentralized fee estimation (avoiding the need for centralized services for this).

Decentralized fee estimation works just fine without looking at the mempool. It is only necessary to look at the mempool if your goal is to get into the \*very next block\*. Since nobody has this need now that we have Lightning, I think it is perfectly acceptable to point users to centralized fee estimation services in the rare case where this is needed, at least until we have a way to retain transactions just for fee estimation (as we do for compact block reconstruction).

> For accelerating relay of blocks. When a node already has (most, or ideally all) transactions that are included in a block downloaded and/or validated, it does not need to redo those things when transactions actually appear in a block. The BIP152 compact blocks protocol relies on this principle. The faster the node network can propagate blocks through the network, the more it reduces the benefit of large miners over small miners (this is related to selfish mining), contributing to decentralization by not hurting new small miners entering the market.

Doesn’t the risk of having a spam-filled block orphaned because of slow propagation cancel out the risk that the miner will \[accidentally\] perform a selfish mining attack? This is another place where I’d love to see some actual numbers.

Also, blockreconstructionextratxn helps here.

> When miners accept transactions that a node shunned, the node is still required to download, process, validate, and store its result

Yes of course, but as the opreturn filter shows, if enough nodes are filtering, there is a 99% reduction in confirmed spam, so eventually the tradeoff is worth it

> forgoing any benefits related to moral stance or resource usage the shunning might have had in the first place.

That is a bold claim; again, I would love to see numbers that show how much an individual node runner is hurting himself by not relaying certain transactions… my hunch is that it doesn’t hurt much at all. I’ve been running Lightning on Knots for a long time and haven’t noticed any issues.

> Of course, nodes are in no way required to participate in transaction relay. They can use whatever policy they like, including not relaying transactions at all (the -blockonly setting), if they are uninterested in having a reliable view of future transactions.

Yeah, and if they are also uninterested in helping bitcoin function as a payment network…

> > I also hear the argument “Blocks are empty” so Miners need generate revenue. To which I also respond; Bitcoin mining is a free market and the difficulty adjustment helps miners set their output accordingly to balance their electricity bill. They could partially set their output off/on.
>
> Miners have huge capital investments in their hardware that needs to be paid off, the actual electricity costs of running the miners are only part of their bill. In the long term you’re right of course that miners can and will shut down if they are unprofitable, and this will result in the difficulty compensating for it, making costs lower for the remaining miners. But even then, those remaining miners in the new equilibrium will prefer earning more fees over less fees!

Yes, and if everyone is filtering the same set of transactions, then no miner has an advantage here…

> And if people are willing to pay those, for whatever reasons, including ones you and I dislike, they’ll find ways to do so.

Yes, and miners who include these transactions are hostile and as such will have additional costs, and there will be 99% fewer abusive transactions if we filter aggressively enough (again, see opreturn data)

> > So is this changed being pushed by people who want to put a storage business model on top of bitcoin, like the altcoins do ? What does will affect Node runners (like me) who don’t want to store dumb JPGS and possible illicit material on their hard drives.
>
> Speaking for myself, I hope you believe me when I say that is not the motivation at all. I think these use cases are temporary hype cycles, and not rational use of blockchain space, but the market can stay irrational for a long time.

Agreed, they are temporary hype cycles, but they never stop. There is always another cycle, and our hostility to them is the primary deterrent that reduces their frequency and intensity.

> However, I believe that attempts to discourage these use cases through node relay policy, in the presence of widespread evidence that miners accept these transactions anyway, are ineffective, akin to making nodes bury their heads in the sand, and ultimately harmful to the decentralization of the system at large.

I disagree with all of these claims. See above discussion.

> (\*) One small note however, the ecosystem does have an actual means of blocking transactions: consensus rule changes. If there is widespread agreement throughout the ecosystem that a certain class of transactions is harmful, the possibility exists in theory to adopt a consensus rule change that makes these transactions outright invalid, forcing miners to reject them (because otherwise they’d be producing invalid blocks, forgoing all revenue). This process is slow, should be used very carefully not to undermine fundamental properties (like censorship!) and is unsuitable for anything resembling a cat-and-mouse game like this, in my view.

Yes, this is the reason we should play the cat-and-mouse game at the policy level, and not at the consensus level. There is an army of thousands of noderunners who are eager to play the part of the cat. I honestly don’t see how we are going to lose against these invaders.

> However, I do want to bring this up to make it clear that in the presence of widespread, fatally-harmful behavior, node runners are not entirely powerless in this matter.

A consensus change would obviously be a last resort, as it is very easy to store arbitrary data in the blockchain, even with extreme measures like p2sh^2. The mempool is where we should focus our efforts.

Sincerely,

Chris Guida

\[0\]: [https://bitcoin.stackexchange.com/a/127903/34855](https://bitcoin.stackexchange.com/a/127903/34855)

\[1\]: [https://gnusha.org/pi/bitcoindev/CAAS2fgQyVs1fAEj+vqp8E2=FRnqsgs7VUKqALNBHNxRMDsHdVg@mail.gmail.com/](https://gnusha.org/pi/bitcoindev/CAAS2fgQyVs1fAEj+vqp8E2=FRnqsgs7VUKqALNBHNxRMDsHdVg@mail.gmail.com/)

\[2\]: [https://blog.bitmex.com/dapps-or-only-bitcoin-transactions-the-2014-debate/](https://blog.bitmex.com/dapps-or-only-bitcoin-transactions-the-2014-debate/)

\[3\]: [https://x.com/GrassFedBitcoin/status/1931854732831977845](https://x.com/GrassFedBitcoin/status/1931854732831977845)

\[4\]: [https://x.com/oomahq/status/1931856202511888465](https://x.com/oomahq/status/1931856202511888465)

\[5\]: [https://github.com/gudnuf/memcooler](https://github.com/gudnuf/memcooler)

\[6\]: [https://github.com/bitcoinknots/bitcoin/pull/119](https://github.com/bitcoinknots/bitcoin/pull/119)

\[7\]: [https://x.com/cguida6/status/1965525597683060979](https://x.com/cguida6/status/1965525597683060979)

cc murchandamus sr_gi pwuille adam3us

-------------------------

mechanic | 2025-09-17 17:50:38 UTC | #2

Thanks for taking the time to write this. As I mentioned on Nostr, I think concerns around fee estimation have been misrepresented. You can never know what a miner will include in their next block and trying to make any aspect of Bitcoin function in a way that relies on the premise that you can is a fool’s errand.

Even with near-complete mempool homogeneity, the next block can just take 30 minutes instead of 10 minutes at which point your didn’t pay a high enough fee.

Further, some miners run non-Core filters. If you want to go mempool-only with your fee estimation that means you’re competing with imaginary transactions that don’t exist to some miners rather than looking at what’s actually making it into the chain.

In the former scenario you’re overpaying for no reason which is unfixable. If filtering results in accidental *underpayment* then you can just RBF. It makes no sense to invoke fee estimation as a reason to try to imagine you can accurately predict what Antpool’s next block is going to contain. You must use a network-wide approximation that comes from far more subtle heuristics that assume mining can become decentralized along with relay policy.

The push to make all mempools as identical as possible along with trying to anticipate miner-behaviour is resignation to a status quo that ultimately leaves Bitcoin censorship prone due to extreme centralization.

Bitcoiners must get comfortable with the fact that mempools will contain what their user allows, not what Core developers “force” upon them by virtue of crazy defaults.

-------------------------

ismaelsadeeq | 2025-09-17 18:43:39 UTC | #3

I think your assumption about Bitcoin Core fee estimation is not accurate. We do not rely solely on mempool transaction data for fee estimation, as you imply.

We use transactions that your node has seen in its mempool that have been confirmed. If your node’s policy rules diverge from the majority of the network and prevent it from seeing a significant number of transactions, this can negatively affect the `estimatesmartfee` algorithm.

Even proposals for mempool-based fee estimation do not use mempool data blindly; they apply metrics and verify that a node’s policy aligns with the majority of the network’s hashrate before estimating fees using unconfirmed mempool transactions.

If only a small fraction of miners filter differently than your node, the effect is negligible: your node will still see most transactions and should estimate fees accurately.

The problem arises when a large percentage of the hashrate is filtering. In that case, Wuille has proposed https://github.com/bitcoin/bitcoin/issues/27995 ignoring transactions that a node expects to confirm that are not actually confirming when making a fee-rate estimate. This prevents overpaying when filtered transactions pay higher fees and mitigates adversarial scenarios where a miner controlling a majority of hashrate broadcasts high-fee transactions without confirming them to try to trick the network into paying more.

We are not in that situation now. I think it is in the best interest of your node’s mempool to reflect the policy of the majority of the network’s hashrate; if it does not, it will harm the node’s ability to predict competitive transaction fees for its transaction.

-------------------------

AdamISZ | 2025-09-21 13:42:09 UTC | #4

Reading both yours and @mechanic ‘s points here (though forgive me I have not read *all* of your post yet!) is helpful to me, thanks.

[quote="cguida, post:1, topic:1991"]
Put more simply: bitcoin’s identity as money cannot be enforced at the consensus layer. Therefore, the only way to ensure that bitcoin stays money is if we enforce its usage as money at the social/mempool layer. The only reason bitcoin has stayed money thus far is because of its protective “bitcoin maximalist” culture, which ensures that its #1 priority is always to be the best money, and that other use cases are severely discouraged.

[/quote]


> Yes, this is precisely why the consensus rules are useless for making sure bitcoin stays money, and why we must react at the policy level rather than the consensus level. Only the most popular formats, which by definition are easily-identifiable, need to be filtered.

This helps clarify it a bit better for me (and I see you expand on the same point elsewhere, too). So you are saying that something that is not definable with a fixed static ruleset needs to essentially be “policed” by nodes, with the filters acting as a warning mechanism. Perhaps the biggest difference of opinion we have is that I see this as not only impractical but actually dangerous to Bitcoin as money, more so than spam itself. I’ll explain first abstractly and then with a couple of examples:

* the core essential property of bitcoin is that others cannot tell me what transactions I am and am not allowed to make (yeah, I know, heard it all before :slight_smile: )
* Concretely, let’s start with this trivial example: breach transactions in Lightning publish hashes on chain. That is data, it is not a payment (not a destination, not an amount). Same with submarine/coin swaps, often. Stupid example? Sure, but …
* What if it was like Lightning as a general offchain payment mechanism, but required 5 hash preimages onchain, not 1? What if it required 200, spread across 20 transactions?
* The above is made up, of course, but as I’m sure you know, there are proposals for fraud proof based systems that *tend* to use quite a lot of onchain transactions (I think flavors of BitVM though that field is moving way too fast to say anything specific)
* There are probably people who think that “souped up” L2 systems (e.g. “rollups”) are either not going to happen, or for some reason are a bad idea, but if you could do properly trustless exchanges with near-infinite bandwidth, 100x cheaper and 100x better privacy than onchain, are you advocating against that if the data it puts on chain is not “financial”?
* You might reasonably answer all the above with “of course I’m not against L2s and as long as the data they put onchain is reasonable; that is effectively financial, even if not literally; that isn’t part of what I’m advocating policing with our nodes”, but what about an L2 that both offers fast cheap private payments, and also lets people mint jpegs? And because it’s private, you don’t know from the random 32 byte strings onchains, whether the activity was “financial/payments” or not?
* Notice I didn’t even appeal to the obvious (and historical, and practical) example of data embedded in pubkeys, nor to the real world of today with data embedded in scripts. 

But why such a strong statement as “dangerous to Bitcoin as money” though? Because it isn’t just wrong-headed to try to police transactions at the network layer, it’s anathema to Bitcoin’s central purpose. If Bitcoin’s p2p network is actively attacking Bitcoin (the protocol) itself (certainly not the case now because it’s not homogeneous, and long may that last! Including, yes, I think it’s great that Knots, and others, exist as a plausible alternative) because it becomes a police force on what transactions are allowed, it may fail under that weight, because if it still existed at all, it would do so with small groups of miners talking to each other secretly.

Another way to look at it is: if the p2p network becomes properly homogeneous in that imagined scenario, **and could fully agree on a specific type of transaction to be banned**, then they could and should simply soft fork in the restriction. Like ban, or limit, OP_RETURN for example.

But I think your response to that is: “you’re missing the point! the bad actors shift around and so a simple static rule can’t ban them.” Well, that means it’s also completely impractical to police them. Are node runners going to get together every week and review the patterns of “bad” transactions that have been happening? The whole point of Bitcoin is that is not a *community* based money, such things completely fail to scale, and also the point of Bitcoin is not to take power away from a central group and give it to the people, it’s to take power away from the central group and give it to **no one at all**. Having a committee decide what kinds of transactions to filter out this week is, I’m sure you agree, not realistic.

Obviously the picture I painted there of “police gone wild” is hyperbolic - I don’t think that will happen! I’m just saying nodes policing the network to enforce rules that are not in the software is somewhere between incoherent and actively dangerous because of the centralization pressure it creates.

(This is more personal opinion, but I think Bitcoin’s long term success will depend on more wide usage, which in turn will depend on finding the right way to do what Peter Todd likes to call client side validation with ZKP techniques, because the usability barriers won’t be overcome-able otherwise at scale; notice how client-side-validation as a paradigm really doesn’t fit with what you’re saying. But this is parenthesized because it’s not a necessary part of the discussion).

-------------------------

cguida | 2025-09-25 22:55:08 UTC | #5

Hi Adam, I appreciate your thoughtful response, you brought up several great questions :slight_smile:

\>Concretely, let’s start with this trivial example: breach transactions in Lightning publish hashes on chain. That is data, it is not a payment (not a destination, not an amount). Same with submarine/coin swaps, often. Stupid example? Sure, but …

There are 3 reasons why this is acceptable:

1. **It doesn’t use excessive data**. The size of the data fits into 40 bytes, so it is considered standard by \~all nodes

2. **It doesn’t cause high fees or bloat the utxoset**. Shitcoin metaprotocols (such as brc20 and runes), in contrast, are guilty of damaging bitcoin in these ways.

3. **Lightning is a bitcoin L2 and not an app**, so we should treat it as more vital to bitcoin than an app. HTLC resolution transactions are required for the proper operation of Lightning, a unilateral-exit L2 that bitcoin cannot succeed without. We did 3 forks in order to enable Lightning; it deserves to be treated as a vital part of bitcoin at this point and not some sideshow. There are lots of other potential unilateral-exit L2s, like Ark, and these should also be considered bitcoin as such, and not applications, since they do not require trust in order to exit to L1.

Citrea uses excessive data (144 bytes for its watchtower challenge txs and apparently a lot of inscription data as well) and is not an L2 because it does not have unilateral exit (having opted instead for a 1-of-n trust model, so it should be considered an app, and not bitcoin as such). However, I don’t expect Citrea watchtower challenge transactions to cause utxoset bloat or high fees (though the other data Citrea plans to post on-chain may do so).

\>are you advocating against that if the data it puts on chain is not “financial”?

No, I’m saying that we should not be making extreme changes to bitcoin’s social contract (that it should be used primarily for money; lifting the opreturn limit makes this much harder to achieve) to accommodate a non-unilateral-exit app. I don’t care if people build these things as long as they fit within the arb data constraints and don’t cause excessive fees or utxoset bloat.

\>And because it’s private, you don’t know from the random 32 byte strings onchains, whether the activity was “financial/payments” or not?

Well no, we don’t need to know what the actual use case is; we only need to know whether that specific format is causing high-volume spam that drives high fees and/or utxoset bloat. That information is sufficient to both identify such transaction formats as spam and to detect and refuse to relay them.

\>But why such a strong statement as “dangerous to Bitcoin as money” though? Because it isn’t just wrong-headed to try to police transactions at the network layer, it’s anathema to Bitcoin’s central purpose.

I disagree; bitcoin’s central purpose is to be permissionless money. If we pivot to data storage, there is no way to avoid data storage becoming so dominant that it destroys the payment use case.

\>If Bitcoin’s p2p network is actively attacking Bitcoin (the protocol) itself (certainly not the case now because it’s not homogeneous, and long may that last!

I’m not quite understanding this statement

\>Including, yes, I think it’s great that Knots, and others, exist as a plausible alternative) because it becomes a police force on what transactions are allowed

I don’t think it does. You are conflating spam filtration with censorship here. Censoring bitcoin is not possible because any miner with enough PoW to mine a block can confirm your transaction. You don’t need the relay network to help you with this; you can simply submit your tx directly to a miner. This is the real mechanism that keeps bitcoin censorship resistant; the relay network has nothing to do with it.

\>Well, that means it’s also completely impractical to police them.

It’s completely practical. I am happy with the 99% reduction in large opreturns the opreturn filter currently causes. A *100%* reduction is not practical; I completely agree. But that is not necessary.

\>Are node runners going to get together every week and review the patterns of “bad” transactions that have been happening?

[I don’t see why not](https://github.com/bitcoinknots/bitcoin/pull/119), if that’s what it takes to make sure bitcoin stays money and doesn’t become a general-purpose database that is impossible to use for money. Of course modular filters can be developed, vetted, and distributed in a decentralized way, and obviously noderunners are free to run them or not.

\>The whole point of Bitcoin is that is not a *community* based money, such things completely fail to scale, and also the point of Bitcoin is not to take power away from a central group and give it to the people, it’s to take power away from the central group and give it to **no one at all**

I agree; I don’t see how allowing noderunners to decide which transactions to relay changes this in any way.

\>Having a committee decide what kinds of transactions to filter out this week is, I’m sure you agree, not realistic.

Of course not, and again, I don’t see why that would be necessary. Just let anyone write, audit, distribute, and run whatever filters they want. You’re never going to be able to force people *not* to run filters, and I don’t know why anyone would want this.

\>Obviously the picture I painted there of “police gone wild” is hyperbolic - I don’t think that will happen!

Me neither! But even if it does, again, we have direct miner submission so it makes little difference.

\>I’m just saying nodes policing the network to enforce rules that are not in the software is somewhere between incoherent and actively dangerous because of the centralization pressure it creates.

And I’m saying that some amount of this is inevitable, due to the inability of the consensus rules *by themselves* to guarantee that bitcoin stays money if we don’t bias payments above data somehow.

\>notice how client-side-validation as a paradigm really doesn’t fit with what you’re saying

Can you elaborate on this point? I’m not immediately understanding.

-------------------------

AdamISZ | 2025-09-24 12:46:00 UTC | #6

[quote="cguida, post:5, topic:1991"]
There are 3 reasons why this is acceptable:

[/quote]

(on your counters to my toy examples): so you have I guess 4 rules here, size of data, high fees caused, utxo bloat, bitcoin vs non-bitcoin. On size someone has a to make a rule about how big is/is not OK. I think the difficulty with that that makes consensus difficult is, as we’ve all seen, data just tends to move around to wherever it can’t be ejected easily, and is cheapest. On high fees I have no comment, it’s kind of just related to the others. On utxo bloat I guess I agree, except for the points earlier made by others, ad nauseam, that it’s incredibly difficult to ban data from utxos, albeit it will never be the cheapest option for the spammer. \[1\]

I think your point 3 about “bitcoin vs not bitcoin” is the one I’d like to delve into the most, but see below for my points around “client side validation”.

[quote="cguida, post:5, topic:1991"]
You are conflating spam filtration with censorship here.

[/quote]

I am indeed! That’s probably my central point. “Filtering” has no purpose if it is not with the intention of censoring (you are trying to stop the activity), and “spam” is not something you will ever be able to objectively define at the level of bitcoin transactions (the purpose of my somewhat awkward examples. Btw I thought of another, though it’s kind of off the wall - coinjoins. Coinjoins do not transfer money from one party to another (thinking basic old style equal-output) and *objectively* take up a lot of space on the blockchain for non-financial transactions. They are really spam, in that sense; as silly as this example might seem, it relates to something in the next point). 

[quote="cguida, post:5, topic:1991"]
> notice how client-side-validation as a paradigm really doesn’t fit with what you’re saying

Can you elaborate on this point? I’m not immediately understanding.

[/quote]

Sure. It’s a bit difficult to find clear explanations of the general “client side validation” idea (though, I just found [this](https://docs.rgb.info/distributed-computing-concepts/client-side-validation) which is pretty good!). I’m guessing you already have a (perhaps vague, like mine) understanding, but for anyone who doesn’t: the core idea is that the individual users are validating, offchain, a history defined in terms other than L1, while using L1 as anti-double-spend through committing to some kind of “summary” of the history (think, merkle tree roots e.g.). Essentially it’s just a fancier and grander vision of commonly understood L2. Rollups in Ethereum land (and planned in various experimental ways on Bitcoin) are another example, notwithstanding there’s sleight of hand centralization happening there with quorums and sequencers and whatnot.

So I’m really just making the same point again, that you can’t easily, or sometimes at all, distinguish monetary uses, and at the highest level, it’s fundamentally impossible because the whole *point* of using an L2 is to provide privacy and scalability *by removing the per-transaction semantics from the base blockchain*. So I think your position that we can police usage is only fully compatible with fixed format, public and unscalable payments on the base layer (and tbh, imo, not even compatible with that!).

[quote="cguida, post:5, topic:1991"]
And I’m saying that some amount of this is inevitable, due to the inability of the consensus rules *by themselves* to guarantee that bitcoin stays money if we don’t bias payments above data somehow.

[/quote]

Our inability to bias is a feature, not a bug. In fact it might be the main reason Bitcoin still exists 16+ years after inception.

The thing is, I do (I think) understand your basically opposite position: that on the contrary, Bitcoin is threatened with not existing by the ability to fill it with non-monetary data. I disagree. Perhaps I’ll expand elsewhere, this will get too long otherwise.

[quote="cguida, post:5, topic:1991"]
You’re never going to be able to force people *not* to run filters, and I don’t know why anyone would want this.

[/quote]

Totally agree with the first half, for the second half it’s nuanced to me: I certainly don’t want to be able to *force* people not to run filters, but that’s not quite the same as not *wanting* people to not run filters - there is a reason to want it not to happen, namely the centralization pressure on miners that Pieter and others have gone into. But hey, obviously it’s nuanced.

\[1\] To nerd out for a moment, signing on pubkeys in utxos is the only idea I know to stop utxo spam, and it’s clearly a scalability nightmare. It’s also extremely unpleasant security wise having a signature onchain immediately with every new coin, although I’m speaking theoretically there; I know of no practical security impact (and I’m not considering the quantum threat practical!). An interesting technical question is whether there might still be a way to embed data if scriptpubkeys are (P, (R, s)). I remember looking into it but not finding one, at least not if we speak specifically of Schnorr pubkey-prefixed signatures, and specifically that the (implicit) message signed includes the pubkey P. Of course you can always grind some bytes into a validly generated pubkey, but that’s not really worth considering (vanity etc.).

-------------------------

cguida | 2025-09-24 19:32:19 UTC | #7

\>so you have I guess 4 rules here, size of data, high fees caused, utxo bloat, bitcoin vs non-bitcoin

Yes. Each of these things is a somewhat blurry line, but I think a use case that objectively violates all four (eg brc-20) can be said to be objectively harmful, whereas things that violate none can be said to be objectively harmless. Again, there is a large gray zone, but there are use cases that are objectively harmful and it makes zero sense for a payment network to prioritize these over, eg, Lightning channel opens and closes.

\>On size someone has a to make a rule about how big is/is not OK

Yes, 80 bytes has been the limit for over a decade now. If people want to reopen this discussion that’s fine, but it should be a discussion, not a unilateral decision.

\>I think the difficulty with that that makes consensus difficult is, as we’ve all seen, data just tends to move around to wherever it can’t be ejected easily, and is cheapest

Yes, and whenever harmful use cases emerge, we should attempt to restrict them.

\>On utxo bloat I guess I agree, except for the points earlier made by others, ad nauseam, that it’s incredibly difficult to ban data from utxos

It’s not actually. Knots has filters for inscriptions (which [directly caused \~7GB of new utxoset growth](https://statoshi.info/d/000000009/unspent-transaction-output-set?orgId=1&refresh=10m&viewPanel=8&from=1588309200000&to=now), which is over half the current utxoset). Citrea’s watchtower challenge transactions are [also trivially filterable](https://youtu.be/J9bRVIXOhm0?t=12555).

Data transactions tend to be harmful in direct proportion to how easy to detect they are. So we don’t need to try to eliminate *all* data transactions; just restricting the most harmful formats is more than sufficient.

\>albeit it will never be the cheapest option for the spammer.

Spammers don’t seem to be particularly fee-sensitive. BRC-20 is a case-in-point. Again, data use cases have infinitely more demand for block space than payment use cases, so if we don’t bias payments at all, then data use cases will inevitably dominate.

\>“Filtering” has no purpose if it is not with the intention of censoring (you are trying to stop the activity), and “spam” is not something you will ever be able to objectively define at the level of bitcoin transactions (the purpose of my somewhat awkward examples

I think “makes bitcoin worse money” is a good starting point for a definition of spam. Obviously there is an infinite amount of negotiation over what precisely this means; but I think most of us can probably agree that BRC-20 fits this definition.

I would love to understand how the anti-filter crowd defines “censorship” as this seems to be a persistent point of misunderstanding in these interactions. It sounds like you consider all spam filtration and all attempts at spam filtration to be censorship; is that correct? For example, do you consider email spam filters to be “censorship”?

\>Coinjoins

I think it’s pretty clear that coinjoins don’t meet the definition of spam (as long as they don’t use excessive data, bloat the utxoset, or cause excessive high fees).

Coinjoins’ purpose is to use bitcoin as permissionless money and they obviously increase its utility as such. I don’t think anyone would ever argue that they are spam. Of course noderunners are free to refuse to relay them if they wish, but I find it very dubious that a significant portion of the network would decide to do so.

\>client-side validation

Yes, all L2s and apps do some degree of this. Again, I’m fine with such use cases as long as they stay within 40 bytes (and 80 bytes is the quasi-consensus limit). Opreturn was specifically introduced for such things, and the limit was set to 40 bytes to accommodate a hash and an 8-byte identifier. If people need more space than this, then they should reopen the conversation and get bitcoiners’ consent rather than just unilaterally shoving data into the blockchain. The former is peaceful; the latter is hostile.

\>So I’m really just making the same point again, that you can’t easily, or sometimes at all, distinguish monetary uses, and at the highest level, it’s fundamentally impossible because the whole *point* of using an L2 is to provide privacy and scalability *by removing the per-transaction semantics from the base blockchain*

I understand and appreciate your point that “spam” is not something that can be formally specified in such a way as to be reliable for the rest of time. I had to come to terms with this recently myself when trying to imagine a set of consensus rules that would always be able to reject data spam as invalid while always allowing payment use cases as valid. We agree 100% here. This is precisely the reason I am not advocating for a consensus change; since it is [not even theoretically possible](https://blog.bitmex.com/the-unstoppable-jpg-in-private-keys/) to detect and filter all data use cases, attempting to formalize such things is a fool’s errand.

The fact that spam is not something that can be formally defined is precisely the reason why such things can only be countered at the social/mempool layer rather than the consensus layer, and it is precisely the reason why the protective bitcoin maximalist culture \[and the filters its proponents run\] will always be vital to bitcoin’s survival and success.

*However*, it is *not* the case that, since spam cannot be formally specified, we necessarily need to give up all efforts to bias payments over data. Yes, there is a large gray area as I’ve already acknowledged; but there are also transaction formats that are *only* abused for data. BRC-20 is the worst offender here, and, again, it is trivially easy to identify. It *objectively, permanently* harmed bitcoin and thus is *objectively spam and should be discouraged*.

There is no *general* formula for spam, but we know *100% without a doubt* that BRC-20 *specifically* is spam, and it is very likely that all inscriptions are spam, especially those that violate the datacarriersize setting (80 bytes on a majority of nodes).

\>So I think your position that we can police usage is only fully compatible with fixed format, public and unscalable payments on the base layer (and tbh, imo, not even compatible with that!).

I am not following your train of reasoning to this conclusion. I am not saying that we can make a general filter that has no false positives; I am saying we can filter specific formats we \*know\* are data spam. Mempool filters are the ideal mechanism for this as they can adapt quickly to counter evolving spam threats.

\>Our inability to bias is a feature, not a bug.

No such “inability” exists. See again the [99% decrease in large opreturns](https://x.com/oomahq/status/1931856202511888465) the opreturn filter causes. We have been biasing bitcoin towards payments since the days of Satoshi, and we should continue doing so for as long as we can.

\>In fact it might be the main reason Bitcoin still exists 16+ years after inception.

I disagree. Yes, censorship resistance is a primary reason bitcoin has been successful, but plenty of shitcoins are “censorship resistant” too, and yet they are not money. Censorship resistance is necessary, but not sufficient, for bitcoin to be successful in the long run. We must also make sure that data use cases are de-prioritized as much as we can. It is bitcoin’s maximalist culture and its hostility to scammy/spammy use cases (as well as its censorship-resistant architecture) that has led to bitcoin’s success.

Again, Ethereum is just as “censorship resistant” as bitcoin, and yet it is not as successful and cannot make as strong a claim to being “money” as bitcoin. The difference between Ethereum and bitcoin is this “bias” towards good money, and this bias is a *cultural* (rather than *consensus*) phenomenon because *it has to be*, for the reasons outlined above.

If we break with 16 years of precedent and remove this bias, there is *nothing* stopping bitcoin from becoming \[something very much like\] Ethereum, full stop. (You can easily see this effect on chains like BSV and BCH, which have devolved into shitcoin casinos, having given up on being permissionless money long ago.)

\>The thing is, I do (I think) understand your basically opposite position: that on the contrary, Bitcoin is threatened with not existing by the ability to fill it with non-monetary data. I disagree. Perhaps I’ll expand elsewhere, this will get too long otherwise.

I would love to hear your full rationale, and I can’t really think of a better venue for such discussion than here on Delving.

\>Totally agree with the first half, for the second half it’s nuanced to me: I certainly don’t want to be able to *force* people not to run filters, but that’s not quite the same as not *wanting* people to not run filters - there is a reason to want it not to happen, namely the centralization pressure on miners that Pieter and others have gone into. But hey, obviously it’s nuanced.

Yes, it’s definitely nuanced. I think this conflict is likely never to resolve to anyone’s satisfaction, because the flipside of attempting to prevent users from submitting certain transactions is attempting to prevent users from deciding which transactions they are able to refuse to announce to their peers, which is arguably a matter of free speech and the right to remain silent. I’m sure you would agree that forcing someone to say something against their will is bad; it could be considered compelled speech. Forcing nodes to relay transactions they don’t want to relay is the same thing.

\[1\] To nerd out for a moment, signing on pubkeys in utxos is the only idea I know to stop utxo spam, and it’s clearly a scalability nightmare. It’s also extremely unpleasant security wise having a signature onchain immediately with every new coin, although I’m speaking theoretically there; I know of no practical security impact (and I’m not considering the quantum threat practical!). An interesting technical question is whether there might still be a way to embed data if scriptpubkeys are (P, (R, s)). I remember looking into it but not finding one, at least not if we speak specifically of Schnorr pubkey-prefixed signatures, and specifically that the (implicit) message signed includes the pubkey P. Of course you can always grind some bytes into a validly generated pubkey, but that’s not really worth considering (vanity etc.).

Completely agreed, which is why, again, consensus is not the proper place to fight spam. Mempool policy is much better suited to this purpose.

It’s likely that key grinding (or hiding data in privkeys as the BitMex article shows) would defeat pretty much any consensus-level filter. It is much more cost-effective to filter specific spam tx formats at the mempool level than to try to predict all possible future spam formats and enshrine those in consensus (which would obviously stifle innovation).

-------------------------

AdamISZ | 2025-09-25 01:14:21 UTC | #8

[quote="cguida, post:7, topic:1991"]
It’s likely that key grinding (or hiding data in privkeys as the BitMex article shows) would defeat pretty much any consensus-level filter. It is much more cost-effective to filter specific spam tx formats at the mempool level than to try to predict all possible future spam formats and enshrine those in consensus (which would obviously stifle innovation).

[/quote]

Oh I hadn’t seen that recent [article](https://blog.bitmex.com/the-unstoppable-jpg-in-private-keys/) by Bitmex Research, very interesting that they came up with the same idea I had a while back in May on the mailing list, [here](https://groups.google.com/g/bitcoindev/c/d6ZO7gXGYbQ/m/Y8BfxMVxAAAJ)  ; quote:

> The question is, what is the arbitrary data channel that you refer to, that remains, when doing this? The R-value is ofc arbitrary but it's still a "image" not "preimage" (x-coord for the nonce secret \* G). As I write this, one answer occurs to me, that if you used the same R value twice you leak the nonce and the secret key, which in this weird setup means you are "broadcasting" 2 32 byte values that are random, in 2 outputs, which I guess is the same embedding ratio? A horrible idea in practice given you lose control of the outputs; I know that at least some schemes that embed data in utxos deliberately do so to keep them in the utxo set permanently. So I somehow feel that that's not what you meant ...

I dismissed this partly on the grounds that it has a 50% embedding rate (in non-witness ofc), but also as noted, there is a funnier objection to the idea: we are specifically worried about utxo bloat, but specifically because these outputs are globally spendable, they don’t have to stay in the utxo set! Perhaps we have a need for a janitorial team even if not a police force.

-------------------------

cguida | 2025-09-25 23:01:27 UTC | #9

Yes, it’s a cool mechanism for sure!

\>we are specifically worried about utxo bloat, but specifically because these outputs are globally spendable, they don’t have to stay in the utxo set! Perhaps we have a need for a janitorial team even if not a police force

Unfortunately the utxoset bloat from brc20s is also technically “spendable”, but my understanding is that because of the incentives of their Ponzi protocol, very few of these outputs will ever be spent.

-------------------------

AdamISZ | 2025-09-26 11:45:11 UTC | #10

[quote="cguida, post:9, topic:1991"]
Unfortunately the utxoset bloat from brc20s is also technically “spendable”, but my understanding is that because of the incentives of their Ponzi protocol, very few of these outputs will ever be spent.

[/quote]

I see. But specifically the point i was raising about the private key broadcast case is that it’s globally spendable (hence “janitors”, anyone can be the cleaner there). But overall I’m guessing there’s a can of worms here: depending on the data-embedding protocol itself, you might have dusty output sizes or otherwise uneconomical-to-spend outputs, which interacts with the janitor idea (which *only* applies in those cases where the output is freely spendable by anyone) in that the janitor may have a \~ zero salary …

-------------------------

sipa | 2025-10-07 18:17:50 UTC | #11

[quote="cguida, post:1, topic:1991"]
I did not post this response to StackExchange as it is designed for exactly the kind of 1-question-1-answer dynamic I am trying to avoid).
[/quote]

Thanks for posting this here; I agree this is a better place for discussion.

[quote="cguida, post:1, topic:1991"]
I think this is where we disagree. Greg Maxwell argued[1] in 2015 that “Demand for cheap highly-replicated perpetual storage is unbounded”. I agree with Greg.
[/quote]

Yes, I agree demand for **cheap** highly-replicated perpetual storage is unbounded. As in, as the price per byte goes to zero, the demand goes to infinity (I can't speak for 2015 Greg, but I believe that's what he was trying to say here). If it costs nothing, why wouldn't people just put their backups there? This is the reason for having block weight limits: so that unbounded demand can't impose unbounded costs on node runners.

But that's not what we're talking about here. As a Bitcoin network user trying to get a transaction confirmed, you're not competing with all hypothetical demand for block space, only with demand that is willing to pay a higher fee per weight than yourself- and at any given feerate value, that demand is finite.

[quote="cguida, post:1, topic:1991"]
If we do not impose any additional costs on people trying to use bitcoin as a general-purpose database, such use will *absolutely* drown out payment use cases, especially merchant Lightning nodes which, at the moment, are bitcoin’s most important payment use case.
[/quote]

I don't know if this is what you believe, but if it is the case that demand for the data-storage use case will always be **willing to pay a higher price per byte** than the payment use case, then I believe Bitcoin simply never had a chance, and a very fundamentally different design is needed. However, I don't think this is the case: there is no rational reason why someone would want their data replicated *precisely* to all Bitcoin nodes and nothing else if it's not data relevant for bitcoin, if it comes at the cost of competing with those who do actually need that - bitcoin payment data.

As for imposing additional costs - I do think that's possible in theory, through consensus rule changes, but only up to a small constant factor, as any data publishing can be encoded in ways that are indistinguishable from normal payment data. Assuming you're talking about policy, I do not believe that it's possible for policy to impose extra *cost*. It can add a burden to deployment, but not in operating cost. There is no reason why, given enough demand, direct submission to miners ought to be any more expensive than having the node network relay it for you - centralized systems are inherently more efficient after all, so it might even be cheaper. The best we can hope for is make it so the advantages of direct relay are sufficiently negligible to not warrant the cost of deploying them at scale (they exist already of course, but only provide a tiny amount of mining income right now).

[quote="cguida, post:1, topic:1991"]
I think filters make it *look like* there is no demand for large opreturns, just as the fence around your house makes it *look like* there is no demand for trespassing on your property. Deterrents are often misunderstood, because you can’t see all the attempts that never happened, *precisely because of the deterrent*.
[/quote]

Of course, deterrents can work. I think it's pretty clear that historically, policy-based discouragement of large OP_RETURNs has worked. But as stated, deterrents like this are just adding a burden to development, not a long-term cost increase. I think of it as similar to the [smart cow problem](https://en.wikipedia.org/wiki/Smart_cow_problem), where the fence stops being effective as soon as a single person does the work of figuring out how to bypass it. And I think it's clear this has happened; just look at the [flood of sub-1-sat/vb transactions](https://mainnet.observer/charts/fees-sub-1-sat-vbyte-transactions/) that are making it into blocks, long before *any* client (even LibreRelay!) relayed these by default.

I don't think there is a reason to expect demand for very large data-storage OP_RETURNs, as using witness data is cheaper and equally useful for this purpose. I do expect demand for medium sized ones (above 80 bytes), for publication purposes. This is a distinct use case, and likely much more monetary in nature. For example, Lightning HTLCs require the publishing of a preimage on chain without which a transaction cannot go through. The goal here isn't replicated data storage, it's guaranteeing that the other side learns the preimage atomically, together with the funds being taken. Lightning puts this inside a script, but alternative systems may want to put it elsewhere. And inscription-style witness data is undesirable here, as it needs a second transaction, losing the atomicity, or adding significant complication.

This all ignores the fact that the reasons for discouragement have changed. I believe that historically, in the fledgling Bitcoin economy, before blocks were consistently full, there was much more potential harm to the ecosystem from data storage, as it risked accelerating bandwidth usage, chain growth (it predated pruning), and block propagation (it predated compact blocks). With regular organically-full blocks and software developments, there just isn't as much impact on node operators, with or without data-storage transactions (and especially `OP_RETURN`-style data storage has about *the lowest* resource impact on nodes). To be clear, even then, there was the possibility for out-of-band relay to emerge if there had been sufficient demand. That didn't happen back then, but is clearly happening now.

[quote="cguida, post:1, topic:1991"]
I’m not sure what you mean here by “they likely won’t persist with changing economics”… if your assertion is that altcoin Ponzis will magically stop happening *just* because of on-chain fees… I don’t think that’s likely true at all; one look at the “crypto industry” outside bitcoin should be sufficient to explain why. But maybe that’s not what you were saying?
[/quote]

I don't believe that wildly speculative behavior like that will go away entirely no. It almost seems human nature that causes appetite for it.

My belief is that, over time - and it may be a long time - if Bitcoin is successful, on-chain fee pressure *on Bitcoin* will price out all or most inappropriate uses of the technology, causing them to move elsewhere. Either to other chains, or ideally, to other technology entirely. That includes things like silly token minting, JPEG storage, and various other things, which just inherently don't require replication specifically to all Bitcoin nodes. I consider the desire to use the Bitcoin blockchain for these purposes to be dumb and irrational, but of course the market can stay irrational for a long time.

To be clear, I don't consider the presence of these changing temporary irrational demand hype cycles to be a threat to Bitcoin anymore, even if they never really go away.

[quote="cguida, post:1, topic:1991"]
Put more simply: bitcoin’s identity as money cannot be enforced at the consensus layer. Therefore, the only way to ensure that bitcoin stays money is if we enforce its usage as money at the social/mempool layer.
[/quote]

Ok, I very fundamentally disagree here. I believe Bitcoin's identity as money is a result of it being technology that is primarily useful for that. If somehow demand for competing use cases is inherently stronger, then no mempool/relay design is going to stop that: the demand will result in the development of an alternative relay mechanism, centralized or not, that bypasses those node runners entirely.

Moreover, if Bitcoin's survival relies on node runners on a regular basis coming to agreement on what transactions are acceptable, based on the transactions' intent, then it just seems entirely uninteresting technology to me. 

I liked how @AdamISZ put it:

[quote="AdamISZ, post:4, topic:1991"]
The whole point of Bitcoin is that is not a *community* based money, such things completely fail to scale, and also the point of Bitcoin is not to take power away from a central group and give it to the people, it’s to take power away from the central group and give it to **no one at all**.
[/quote]

I saw your response that your goal here isn't censorship but discouragement to curb "abuse". I don't think that changes much: it remains the case that your hope is that node runners have a power to decide which transactions are good, and which are bad, and take action that incentivizes or disincentivizes creators of such transactions.

This isn't to say that data-storage is an intended, and much less desirable, goal of the design. Bitcoin is clearly designed for supporting the bitcoin currency. But it also has programmability, and flexibility for building systems on top whose design wasn't originally known. Data storage is a byproduct of that flexibility.

[quote="cguida, post:1, topic:1991"]
If our mitigations don’t work, we will continue escalating until the spammers leave.”
[/quote]

Do you mean including consensus rule changes? Because apart from that, this is the equivalent of writing an angry letter. Nobody has to care.

[quote="cguida, post:1, topic:1991"]
If you are implying that fees are the only “economic incentives” miners pay attention to, this can be easily disproven by considering the result of the block size war; larger blocks promised *much* larger fee revenues for miners than small blocks (and presumably number-go-up in the short-term because of perceived “scaling”), and yet the small blockers were victorious.
[/quote]

No, miners care about (1) creating valid blocks (incl. the subsidy that gains them) and (2) transaction fees (and probably (1.5) not being shut down by regulators e.g. if they're identifiable). Node runners (at least economically relevant ones) have - through their ability to jointly set consensus rules - power over (1), which is what mattered in the block size war. It doesn't matter what fees a miner might have hoped to have gotten from larger blocks, if their blocks would be considered invalid by the ecosystem. This is also why I say that in the presence of transactions the ecosystem actually considers damaging, consensus rule changes are really the only power node runners have.

But node runners' impact on (2) is very limited, because wallets and miners are not required to use the node runner network to relay transactions.

[quote="cguida, post:1, topic:1991"]
Miners were forced to concede that a bitcoin with smaller blocks and segwit would be much more economically valuable in the free market than a bitcoin with large blocks and no Lightning Network.
[/quote]

I'm not sure I agree here. Miners might as well have been in total agreement that a bitcoin with larger blocks would be more valuable for everyone, they still had no choice but adopting the rules that the ecosystem demanded of them. Miners are "contractors" to the network. The network sets the rules of what blocks they'll accept, and miners order transactions into those blocks, and in return, the network pays them (through subsidy, and the ability to charge fees) for that service. If the network decides to change the rules of the game, miners have to follow or they will stop being paid.

[quote="cguida, post:1, topic:1991"]
No, at best the spammers *leave bitcoin entirely* and go to other chains, as they did in 2014 (again, see [2]). We won the cat-and-mouse game last time; why assume we would lose this time? 
[/quote]

I don't need to assume: I can already see that a large amount of transactions make it into the chain despite not being relayed by the most common node implementations. It's baffling to me that you keep asserting relay-based discouragement works and will continue to work, despite blatant evidence that at least in some cases, it is trivially bypassed.

[quote="cguida, post:1, topic:1991"]
The opreturn filter is performing this role admirably, as can be seen by the 99% reduction in large opreturns[3][4] it causes.
[/quote]

How do you explain that this appears to be the case for OP_RETURNs, and does not appear to be the case for sub-1sat/vb transactions? I think a more likely explanation is that either (1) those with demand for larger OP_RETURNs haven't yet developed the infrastructure to bypass common node implementations yet or (2) there just isn't that much demand for them because there are cheaper ways to do data storage.

To be clear, I don't think it really matters to what extent that is the case or why, as I don't think today large OP_RETURNs, or other data storage demand is in itself harmful to nodes or Bitcoin at large. I think for data-storage use cases this is misguided, dumb, silly, and/or irrational, but me holding that opinion isn't going to change anyone's mind. The market, over time, may.

[quote="cguida, post:1, topic:1991"]
This argument doesn’t make sense to me. You just argued that users can always submit transactions directly to a miner. So bitcoin is still censorship resistant, even if the relay network is entirely hostile nodes, entirely blocksonly nodes, or even if the relay network doesn’t exist at all. So I don’t see how a supermajority of noderunners running filters does anything to change bitcoin’s censorship resistance. Bitcoin is censorship resistant, period. So spam filtration is an entirely different matter from censorship.
[/quote]

Fair enough; this needs more nuance.

Direct submission to miners works for **transactions that both sides consent to** (transaction creators and miners). The P2P node network can assist or not, but there is little it can do to intervene - all it needs is for the transaction data to make it to the right place, with or without their help.

But what makes it so that miners will generally accept all transactions that pay a competitive fee - the property we actually want for censorship resistance? It's not too hard to imagine a world where there are just a few large mining pools, which are heavily regulated, and thus not beyond influence from those who might actually want to block or stifle certain transactions. The answer is that anyone can join the set of miners without asking for permission. **This is the entire reason why proof-of-work exists.** If we would be okay with an immutable set of miners, we could give them some signing keys each, say that every block needs to be signed by a majority, and get rid of the wastefulness of proof of work. We don't do that because the ability for anyone to enter the mining market, for economic reasons, or more importantly because they don't like what existing miners are doing, is what fundamentally provides censorship resistance. Miners don't censor fee-paying transactions, because if they did, someone else could pop up to mine those transactions anyway. Sure, they need hardware, and electricity, and network connectivity, but beyond that, they don't need to ask, or even tell anyone. They could do so anonymously, from anywhere in the world. The fact that today's miners choose to largely identity themselves seems weird to me, but apparently the publicity is worth it to them. In any case, they're not required to keep doing that.

However, for this to work, it must also be possible for miners to get competitive access to fee-paying transactions anonymously. If direct-to-miners payment rails to miners develop and get used to the point that they form a substantial portion of mining income, this stops being the case, because nobody will bother submitting to a small-scale newcomer miner with a tiny percentage of the hashrate. Companies might pop up that manage pools of unconfirmed transactions and turn them into block templates for miners. An anonymous miner may well need to do without access to these, and their fee income might be restricted to just the ideological transactions others refuse (or are coerced not to) mine - needing to mine at a loss otherwise.

**This is what I see as the largest threat to Bitcoin in the short to medium term:** the marketplace for block space moving out of the public P2P network, and into centralized channels where they are ripe for censorship. This isn't exactly a [theoretical concern](https://arxiv.org/abs/2509.16052v1).

[quote="cguida, post:1, topic:1991"]
Again, if we require consensus changes in order to prevent spam, we will obviously lose to the spammers. There is no way consensus rules can keep up with rapidly evolving threats from Ponzi scammers. But modular filters[6] can easily keep up with such threats.
[/quote]

You're saying node runners will get together on a regular basis, and come to an agreement as to what types of transactions are going to get the seal of approval for P2P relay? And you think this will get the 90%+ agreement and adoption rate needed for transaction relay to become unreliable? And that those who disagree won't run their own nodes with different policies, or just develop alternative systems that bypass the P2P network? This seems ludicrous to me, and also doesn't result in what I'd find interesting.

[quote="cguida, post:1, topic:1991"]
The change of the default is the important feature of this discussion. The fact that the control was left in is not significant in comparison.
[/quote]

I agree, but I still felt it was worth correcting the misconception that the option is removed.

[quote="cguida, post:1, topic:1991"]
Doesn’t the risk of having a spam-filled block orphaned because of slow propagation cancel out the risk that the miner will [accidentally] perform a selfish mining attack? This is another place where I’d love to see some actual numbers.
[/quote]

Larger miners benefit from having their blocks being slow to propagate, smaller miners suffer. This is easy to confirm with simulations. It is also intuitive I think: larger miners have a larger probability of finding two blocks in a row, and don't suffer a propagation delay between them.

[quote="cguida, post:1, topic:1991"]
Also, blockreconstructionextratxn helps here.
[/quote]

Only to nodes which have peers whose relay policy matches that of miners. It seems highly hypocritical to me to not want to relay certain transactions yourself, but still rely on having peers that do relay those transactions to you so your reconstruction speed is fast. Put otherwise, the extra pool only helps with policy divergence in situations which more or less correspond to the filtered transactions propagating well anyway.

-------------------------

RandyMcMillan | 2025-10-08 00:10:48 UTC | #12

He will be here next week saying nobody is addressing his questions or concerns.

-------------------------

1440000bytes | 2025-10-08 15:49:40 UTC | #13

[quote="sipa, post:11, topic:1991"]
[quote="cguida, post:1, topic:1991"]
Also, blockreconstructionextratxn helps here.
[/quote]

Only to nodes which have peers whose relay policy matches that of miners. It seems highly hypocritical to me to not want to relay certain transactions yourself, but still rely on having peers that do relay those transactions to you so your reconstruction speed is fast. Put otherwise, the extra pool only helps with policy divergence in situations which more or less correspond to the filtered transactions propagating well anyway.
[/quote]

It is a solution (w/ some trade-offs) without introducing another [p2p message](https://delvingbitcoin.org/t/sharing-block-templates/1906) or expecting every node to use the same policy. Every node has `125` `maxconnections` by default and it is reasonable to expect some divergence in mempool polices of a decentralized network.

-------------------------

sipa | 2025-10-08 16:22:33 UTC | #14

[quote="1440000bytes, post:13, topic:1991"]
It is a solution (w/ some trade-offs) without introducing another [p2p message](https://delvingbitcoin.org/t/sharing-block-templates/1906) or expecting every node to use the same policy. Every node has `125` `maxconnections` by default and it is reasonable to expect some divergence in mempool polices of a decentralized network.
[/quote]

I am not saying the extra pool for block reconstruction is a bad idea, or that it doesn't help in the presence of mempool variance (whether that's caused by policy divergence, or other reasons). It is great.

I am saying it is hypocritical to bring it up in this discussion, because it only helps with reconstruction rates when mined transactions already propagate well. If your goal is making a certain class of transactions relay so badly across the P2P network that it becomes unreliable as OP seems to want, then obviously you can't expect your node to hear about those transactions anymore, and the extra pool won't help you.

-------------------------

jal | 2025-10-20 02:44:53 UTC | #15

> Put more simply: bitcoin’s identity as money cannot be enforced at the consensus layer.

Nothing can be enforced and bitcoin has no such identify.

> Therefore, the only way to ensure that bitcoin stays money is if we enforce its usage as money at the social/mempool layer.

That is an absurd and baseless statement and the “therefore” is non sequitur. 

> The only reason bitcoin has stayed money…

The definition of money is EXTREMELY ambiguous and historically ephemeral. And “the only reason” wtf are you talking about? 

It would be generous to call you super disingenuous. You OBVIOUSLY have hidden motives and you don’t understand money or the role of bitcoin’s consensus layer thats clear. 

I don’t know why you get the time of day from these people.

-------------------------

cguida | 2025-10-20 19:00:34 UTC | #16

Hi Pieter - thanks for the thorough response, I really appreciate it.

>Yes, I agree demand for **cheap** highly-replicated perpetual storage is unbounded. As in, as the price per byte goes to zero, the demand goes to infinity (I can’t speak for 2015 Greg, but I believe that’s what he was trying to say here). If it costs nothing, why wouldn’t people just put their backups there? This is the reason for having block weight limits: so that unbounded demand can’t impose unbounded costs on node runners.

You are probably correct that this is what Greg was specifically referring to, but I think the statement is still applicable now, because [the cost per month goes to zero as the time the data is hosted goes to infinity](https://groups.google.com/g/bitcoindev/c/G10fR8-mnIA/m/A3_O_YTlAAAJ):

>Data spam pays an upfront fee, then enjoys bulletproof integrity and availability guarantees for the rest of eternity. A finite quantity (the upfront fee) divided by an infinite quantity (the amount of time the data is hosted) is zero. This is why payment txs can never fairly compete with data txs.

Data storage may not be as cheap as it was in 2015, but it’s actually pretty close now, since on-chain traffic is low and we just lowered the default minrelaytxfee.

>But that’s not what we’re talking about here. As a Bitcoin network user trying to get a transaction confirmed, you’re not competing with all hypothetical demand for block space, only with demand that is willing to pay a higher fee per weight than yourself- and at any given feerate value, that demand is finite.

Again, even with a very high upfront fee, the utility to the user of (for example) 1kB of block space is much greater than the utility of the same amount of block space to someone trying to open a Lightning channel; say, for example, a merchant trying to set up Lightning payments to accept bitcoin in a shop. As many have often explained, notably [Andreas Antonopoulos](https://x.com/cguida6/status/1977015055331078538), it's very important to bitcoin's success that bitcoin be conducive to direct p2p commerce, otherwise it will be impossible for the bitcoin standard to replace the fiat standard.

Shitcoin metaprotocol marketers explicitly use the fact that bitcoin has credibly censorship-resistant, perpetual data storage to trick their victims into thinking these fungible token schemes have value, which causes them to throw much more money away on fees than a rational actor otherwise would.

It's a bit odd to me that core devs' response to this situation is "just use bitcoin more", when it's largely because of bitcoin core being difficult to use for non-technical merchants that bitcoiners tend to avoid spending their bitcoin.

(For details on merchants' struggles, see this [email I sent to the bitcoindev ML](https://x.com/cguida6/status/1976003362723336410), which was inexplicably censored by the ML mods.)

But there is also a general cultural reluctance to spend bitcoin directly, which I agree is harmful.

> I don’t know if this is what you believe, but if it is the case that demand for the data-storage use case will always be **willing to pay a higher price per byte** than the payment use case, then I believe Bitcoin simply never had a chance, and a very fundamentally different design is needed.

Yes, I really do believe this, and no, I don't think bitcoin needs to be redesigned.

It is not possible to define bitcoin as money in the consensus rules; this is merely a [very strong] social contract, only enforceable at the social/cultural/policy layer. This means that no matter what changes we make to the consensus rules, there will always be [little hacks and workarounds](https://blog.bitmex.com/the-unstoppable-jpg-in-private-keys/) to store data in the blockchain.

However, as long as bitcoin's social layer is committed to always making bitcoin better money, and we demonstrate that we are committed to going tit-for-tat with the spammers whenever they discover workarounds, they will eventually [give up and flee to other chains](https://x.com/cguida6/status/1970171927831683095), which have much more spam-friendly social layers.

Yes, policy is leaky, but together with consensus changes and the political will to use both, we have more than enough firepower to repel the spammers indefinitely.

As I and others have stated repeatedly, it's not necessary to prevent *all* spam; it's only necessary to target specific use cases that are objectively known to cause harm, such as shitcoin metaprotocols like brc20 and runes.

> However, I don’t think this is the case: there is no rational reason why someone would want their data replicated *precisely* to all Bitcoin nodes and nothing else if it’s not data relevant for bitcoin, if it comes at the cost of competing with those who do actually need that - bitcoin payment data.

This just strikes me as magical thinking. It's like saying "people will only use the park for good things because people who use the park for bad things have to compete with the people who use it for good things".

No. The people who use the park for bad things mess it up for everyone (including themselves!), and the same is true of bitcoin.

>As for imposing additional costs - I do think that’s possible in theory, through consensus rule changes, but only up to a small constant factor, as any data publishing can be encoded in ways that are indistinguishable from normal payment data.

Again, the easily detectable spam (shitcoin metaprotocols, unencrypted contiguous CSAM, etc) is the most harmful kind of spam. We can eliminate 90% of the harm by targeting only 10% of use cases. (For the record, a consensus rule change is probably best to counter contiguous CSAM).

Imposing costs on spammers via policy is not only possible; it's been [wildly successful historically](https://x.com/OrangeSurfBTC/status/1963339476257718764).

> Assuming you’re talking about policy, I do not believe that it’s possible for policy to impose extra *cost*. It can add a burden to deployment, but not in operating cost.

I think it self-evidently increases costs, especially on the most harmful kinds of spam. Otherwise, why would we see a 3-4-order-of-magnitude decrease in large opreturns because of the filter as the data linked above show?

The "cost" is not necessarily direct; most of the cost for shitcoin metaprotocols comes in the form of risk. If noderunners are talking about filtering (or already filtering) a transaction format you're thinking of using for your shitcoin metaprotocol, you're probably not going to raise as much money from shitcoin VCs as if you design your protocol on top of a standard format.

The biggest risk, of course, is that you raise a ton of money thinking you're in the clear and then noderunners *suddenly* start filtering your protocol, at which point everyone loses their investment (or your team has to scramble to redesign the protocol to use a different format, which will probably just get filtered too.. and *then* you lose your investment).

>There is no reason why, given enough demand, direct submission to miners ought to be any more expensive than having the node network relay it for you

For high-volume spam, any tx format that is not accepted by 100% of hashpower has a higher average cost/vB than standard txs (due to a lower supply of available block space), and any tx format that is accepted by a *majority* of hashpower can be considered effectively standard (if it's not harmful), or we need to fork in order to disable said transaction format (if it *is* harmful).

So nonstandard txs will always have a higher *average* cost than standard ones (meaning standardness rules work), and if the cost delta is not sufficient, *then* we can tighten consensus rules. Presumably miners will understand that the standardness rule is a boundary they should not cross if they want bitcoiners to consider them honest rather than hostile.

Also, in practice, we've seen that, even with direct submission, the opreturn filter still reduces the frequency of large opreturns by at least 3 orders of magnitude, so it must be doing something to drastically increase costs, or reduce benefits (otherwise we would not see a reduction at all). I think it is fairly clear what the increased costs are: not wanting to harm bitcoin, not having the knowledge or motivation to seek out alternative submission methods, not wanting to wait several blocks before the mining pool you chose to hit a block, etc.

But again, I think the most obvious costs are on shitcoin metaprotocol *creators*, and not on the *users* of said metaprotocols. There are strong disincentives to creating shitcoin metaprotocols on top of a nonstandard tx format, and I think this is the primary reason large opreturn usage has stayed low for over a decade. (Of course, removing the opreturn filter eliminates these disincentives.)

>The best we can hope for is make it so the advantages of direct relay are sufficiently negligible to not warrant the cost of deploying them at scale (they exist already of course, but only provide a tiny amount of mining income right now).

I don't think that's necessarily true. We could boycott miners who abuse direct submission APIs to solicit harmful transactions (perhaps using something like [memcooler](https://github.com/gudnuf/memcooler), perhaps with a web of trust to disincentivize cheaters). I imagine miners such as Mara and F2Pool would think twice about confirming large quantities of objectively harmful tx formats like brc20 if they knew it would drive fees from normal payment txs to their competitors.

>Of course, deterrents can work. I think it’s pretty clear that historically, policy-based discouragement of large OP_RETURNs has worked.

Fantastic! You are one of the few core devs I've talked to who is intellectually honest enough to admit that filters can work under some circumstances. I think it's been harmful to the discourse that almost all core supporters say "filters don't work", when the historical data trivially refute this claim.

Here is a [chart of opreturn data from September](https://x.com/OrangeSurfBTC/status/1977582697506897961) (capped at 100, so data for large opreturns is visible):

![photo_2025-10-15_14-08-06|690x388](upload://7UOO8uuZJxABlmmVxjRDZlBE6Il.jpeg)

"Filters work in some circumstances and not in others" should be an obvious point of agreement between the two "sides" in this debate. The fact that very few core supporters can admit this is contributing to the perception that the core side is being dishonest. So again, I appreciate the honesty, and I think the more interesting part of the conversation, which is *when* filters work and *how* we can make that happen, can now take place.

>But as stated, deterrents like this are just adding a burden to development, not a long-term cost increase

But again, we are seeing (at least) a 3-order-of-magnitude reduction of large opreturns, *because of the opreturn filter*. So even though the filter does not block *all* large opreturns, it functions well for its intended purpose, which is to *severely rate limit* spam.

> I think of it as similar to the [smart cow problem](https://en.wikipedia.org/wiki/Smart_cow_problem), where the fence stops being effective as soon as a single person does the work of figuring out how to bypass it

An excellent analogy! I agree that filters are somewhat like a gate with a latch that can be opened by a "smart cow". The question then becomes: how can we close the gate and make it harder to open again next time? The farmer does not just sit there and watch while a giant herd of cows invades his land.

> And I think it’s clear this has happened; just look at the [flood of sub-1-sat/vb transactions](https://mainnet.observer/charts/fees-sub-1-sat-vbyte-transactions/) that are making it into blocks, long before *any* client (even LibreRelay!) relayed these by default.

I think it's clear that this happened *for subsat txs*, not for filters *in general*. I suspect that the "subsat summer" was a coordinated effort to create a data point specifically for this conversation. I think someone looked at all the filters and thought to themselves "which of these would be the easiest and cheapest to subvert?". Filling a block with 0.1sat/vB txs costs 100,000 sats, which is about US$112 currently (assuming my math is correct). Together with the same Libre Relay subnet activists that brought you standard mempoolfullrbf, this was clearly the easiest way to "prove" that "filters don't work". (These activists are obviously well aware of how to change their relay policy manually.)

But obviously the opreturn filter is a different beast from the 0.1-1sat/vB filter. The benefits of large opreturns are much less certain than the benefits of cheap txs, while the costs are much larger. Also, large opreturns are just much more expensive fee-wise, so generating a large volume of them in order to incentivize miners to drop the filter is much more costly (especially because it is more likely to fail).

Why not let the Libre Relay subnet attempt to subvert the opreturn filter, same as before? Why do their job for them by *pre-emptively* removing the filter in core's defaults? It makes no sense.

>I don’t think there is a reason to expect demand for very large data-storage OP_RETURNs, as using witness data is cheaper and equally useful for this purpose.

I don't know why anyone who is aware of the brc20 attack during 2023-4 would possibly think this. It is clear that if we give shitcoin spammers a novel way to embed their data, they will abuse it to the maximum extent possible.

>I do expect demand for medium sized ones (above 80 bytes), for publication purposes. This is a distinct use case, and likely much more monetary in nature. For example, Lightning HTLCs require the publishing of a preimage on chain without which a transaction cannot go through. The goal here isn’t replicated data storage, it’s guaranteeing that the other side learns the preimage atomically, together with the funds being taken. Lightning puts this inside a script, but alternative systems may want to put it elsewhere. And inscription-style witness data is undesirable here, as it needs a second transaction, losing the atomicity, or adding significant complication.

Yes, @AdamISZ brought up HTLCs and I responded above. I agree that such use can be legitimate; however, I think that Citrea should have either made their data fit into 80 bytes, which is a long-established norm for datacarrying transactions, or they should have asked consent from the bitcoin network to violate this norm. Just charging ahead using a tx format everyone is well aware is abusive is the act of a hostile entity, and appeasing such behavior will only encourage more of the same.

Obviously if PR#32406 had raised the limit to 145 or so, that would have been much less controversial than simply removing the filter entirely.

>This all ignores the fact that the reasons for discouragement have changed. I believe that historically, in the fledgling Bitcoin economy, before blocks were consistently full, there was much more potential harm to the ecosystem from data storage, as it risked accelerating bandwidth usage, chain growth (it predated pruning), and block propagation (it predated compact blocks).

This is a good point; it is certainly true that spam is less harmful now than it was a decade ago. But as I said above, no matter how much we improve bitcoin's tech, the threat from [network-level DoS from shitcoin spam](https://x.com/AnitaPosch/status/1732722398359638414) will always exist, and the most efficient way to fight such rapidly-evolving threats is in policy (and *then* consensus, if policy is insufficient). It makes little sense to artificially suppress filtering, because then we can't even collect data as to which miners and what percentage of hashpower would flout the filters and are thus openly hostile.

> With regular organically-full blocks and software developments, there just isn’t as much impact on node operators, with or without data-storage transactions (and especially `OP_RETURN`-style data storage has about *the lowest* resource impact on nodes).

Well no, the thing that has caused the biggest impact to node operators is the [more than doubling of the utxoset by the brc20 attack](https://statoshi.info/d/000000009/unspent-transaction-output-set?orgId=1&refresh=10m&viewPanel=8&from=1588309200000&to=now). This attack occurred despite the arb data for brc20 being stored in the witness, which does not go in the utxoset. So "large opreturns cannot cause utxoset bloat" is not a valid claim. Degens can keep designing inefficient, harmful protocols much longer than we can stay solvent, unless we discourage them somehow.

>To be clear, even then, there was the possibility for out-of-band relay to emerge if there had been sufficient demand. That didn’t happen back then, but is clearly happening now.

Okay, but the simple fact that it's *possible* to submit txs out of band to miners does not mean that filtering is always bad and that we have no choice but to allow whatever spam pays a high enough fee. That is an incredibly dubious jump in reasoning. The reasonable reaction is not "well, slipstream exists, so I guess the spammers win." The reasonable reaction is:

- Okay, they are going around the filters. What can be done about this, if indeed anything needs to be done?
- At first glance, the fact that direct APIs exists does not seem to be weakening the opreturn filter much. It is still extremely effective (it has achieved more than a 3 order-of-magnitude reduction in large opreturns), so maybe the tradeoff is worth it.
- Can we provide social pressure on hostile mining pools to get them to reconsider their abusive behavior? (This has worked many times in the past).
- Can we provide financial pressure by boycotting them? (See memcooler for a way to accomplish this).

...etc. We should not just throw up our hands and say "okay, hostile mining pools, you win"
. That is clearly a terrible precedent.

>I don’t believe that wildly speculative behavior like that will go away entirely no. It almost seems human nature that causes appetite for it.

Yes, and the fiat money printer encourages a lot more of it than would be normal.

>My belief is that, over time - and it may be a long time - if Bitcoin is successful, on-chain fee pressure *on Bitcoin* will price out all or most inappropriate uses of the technology, causing them to move elsewhere.

What evidence do you have for this belief? How do we expect people to increasingly adopt trustless utxo-sharing schemes like Lightning and Ark (I assume this is what you mean by "on-chain fee pressure [from payments] will price out [data spam]") if there are huge spam attacks that prevent merchants from opening and closing Lightning channels for months at a time?

This belief seems to be based purely on faith. We have to make it happen, or it won't happen.

>I consider the desire to use the Bitcoin blockchain for these purposes to be dumb and irrational, but of course the market can stay irrational for a long time.

As explained above, the value proposition of storing data on-chain is obvious. It is *especially* effective for shitcoin metaprotocols, which rely on bitcoin's bulletproof persistence/censorship resistance guarantees to trick users into thinking the tokens built on such schemes have value, and this will never stop being the case unless we do something to make the guarantees less bulletproof for this type of abuse.

>To be clear, I don’t consider the presence of these changing temporary irrational demand hype cycles to be a threat to Bitcoin anymore, even if they never really go away.

Right, I guess this is where we disagree. It seems obvious to me that if we stop being hostile to spam, we are inviting the entire shitcoin industry to set up shop on bitcoin, and then there will be no important differentiators between bitcoin and the rest of crypto. Our hostility to the spam is the deterrent. Once the deterrent is gone, the demand for spammy metaprotocols will go to infinity, as demand for harmful use cases is only held in check by our hostility toward it.

>I believe Bitcoin’s identity as money is a result of it being technology that is primarily useful for that

It's primarily useful for money because [*bitcoiners are hostile to non-monetary use cases*](https://blog.bitmex.com/dapps-or-only-bitcoin-transactions-the-2014-debate/). If we stop being hostile to non-monetary use cases, it will no longer be the case that data storage has lower demand than payments. See literally any altcoin for examples of what happens to a coin that otherwise looks like bitcoin technically but does not have a monetary maximalist culture.

I have provided evidence that refutes your claim. What evidence do you have that supports it?

>If somehow demand for competing use cases is inherently stronger, then no mempool/relay design is going to stop that: the demand will result in the development of an alternative relay mechanism, centralized or not, that bypasses those node runners entirely.

Again, we only need to do what worked in 2014, which was to *convince the spammers that we intend to escalate until they go away*. It seems silly not to at least try something that was demonstrably very effective every other time it was tried. If for some reason it doesn't work this time, we can simply escalate by changing the consensus rules.

If demand for block space for data is inherently higher than demand for block space for payments (I don't see any evidence that it's not), and we do nothing to correct this bias, then bitcoin dies. If your belief is "well then bitcoin isn't interesting/was never going to work anyway", then I think you are being too utopian.

Bitcoin as a decentralized money with a decentralized maximalist culture that defends its moneyness can potentially improve the incentive structure of the world to such a degree that we could ignite a new Renaissance. I don't think there are any methods of human organization that just "run themselves", indefinitely, with no regard to the honesty of the participants. Anyway bitcoin itself, whose consensus mechanism relies on 51% honest hashpower, is certainly *not* such a system. Let's not let perfect be the enemy of good.

>Moreover, if Bitcoin’s survival relies on node runners on a regular basis coming to agreement on what transactions are acceptable, based on the transactions’ intent, then it just seems entirely uninteresting technology to me.

1) No one is suggesting that nodes "come to agreement on a regular basis on which transactions are acceptable based on the transactions' intent". I'm just suggesting that we keep doing what we are doing with large opreturns, which causes the aforementioned 3-order-of-magnitude drop. (Again, no censorship, just ratelimiting. As it has been for over a decade.)
2) There are transactions that are *objectively harmful*, based on their format, not their intent. brc20 is one such format. We don't need a committee to decide not to relay these. We just need noderunners to decide for themselves which transactions not to relay, and the ones that are the most harmful will have the least nodes relaying them, and will thus be the most difficult to get confirmed. As it should be.

>I don’t think that changes much: it remains the case that your hope is that node runners have a power to decide which transactions are good, and which are bad, and take action that incentivizes or disincentivizes creators of such transactions.

There is nothing we can do to take this power away from nodes, so trying to make it so that noderunners cannot refuse to relay certain tx formats is "fighting the market" just as much as filtering itself. So your desire to limit the types of filters noderunners can run is just as ill-advised as filtering itself, by your own standards.

>This isn’t to say that data-storage is an intended, and much less desirable, goal of the design. Bitcoin is clearly designed for supporting the bitcoin currency. But it also has programmability, and flexibility for building systems on top whose design wasn’t originally known. Data storage is a byproduct of that flexibility.

Sure, but to the extent possible, we should make sure that such flexibility does not undermine important properties of the system. For example, I doubt that you yourself would advocate for getting rid of the dust filter, or limits on transaction sizes, etc.

>Do you mean including consensus rule changes? Because apart from that, this is the equivalent of writing an angry letter. Nobody has to care.

Yes of course! But it's very irrational to suggest that we should just fork, without trying filtering first. Again, filtering has worked literally every time we've tried it (for tx formats that are objectively harmful). It makes no sense to escalate directly from "all spam is fine" to "FORK!" This is like doing nothing and pretending not to notice as someone repeatedly does something that bothers you, then suddenly punching them in the face with no warning. This is the strategy of a bad actor. A good actor says "hey, please stop that", then escalates proportionally until the request is complied with.

If we default to filtering a certain tx format and miners *still* can't take a hint, *then* it makes sense to fork. But the fork is always there as a threat, which is a significant reason why filtering alone can work: because miners know there are harsher fallback methods.

>No, miners care about (1) creating valid blocks (incl. the subsidy that gains them)

Yes, but miners also have influence over which blocks are valid. My point is that merchant nodes have *more* influence over this, and the block size war proves this. When the rest of the ecosystem says "hey miners, you're making bitcoin worth less", miners are forced to listen, even if they normally ignore long-term concerns in favor of short-term profits.

In the case of spam, we can achieve this by filtering (and implicitly threatening a fork), or by forking. Ultimately the miners are forced to listen to the capital/merchant nodes, because they want to end up on the "correct" chain. This is very much an "economic incentive".

>It doesn’t matter what fees a miner might have hoped to have gotten from larger blocks, if their blocks would be considered invalid by the ecosystem. This is also why I say that in the presence of transactions the ecosystem actually considers damaging, consensus rule changes are really the only power node runners have.

This is like saying that words have no power but punching has power. The words have power *because* the punching has power, and thus we can usually settle our differences using only words. The same is true of filtering.

>But node runners’ impact on (2) is very limited, because wallets and miners are not required to use the node runner network to relay transactions.

There are plenty of ways to disincentivize miners from mining objectively harmful tx formats. I have already suggested several. Filtering by default in bitcoin core (which carries the threat of a fork in the event the filters are ignored) is the most efficient way. Boycotting hostile miners would also be straightforward to implement.

>Miners might as well have been in total agreement that a bitcoin with larger blocks would be more valuable for everyone, they still had no choice but adopting the rules that the ecosystem demanded of them

Exactly; it doesn't matter what miners want; it matters what the market determines is the best version of bitcoin. It's obvious to me that a spam-resistant bitcoin is much more valuable in the free market than a bitcoin filled with spam (just see any altcoin for proof).

>Miners are “contractors” to the network. The network sets the rules of what blocks they’ll accept, and miners order transactions into those blocks, and in return, the network pays them (through subsidy, and the ability to charge fees) for that service. If the network decides to change the rules of the game, miners have to follow or they will stop being paid.

Exactly; we just need to make our wishes known to the miners; it's obvious which fork would win in a fork war between a spam-filled fork vs a spam-resistant one, so miners may as well just stop mining spam now. This is the "economic incentive" I was referring to.

>I don’t need to assume: I can already see that a large amount of transactions make it into the chain despite not being relayed by the most common node implementations.

1) Yes, because core has suddenly decided to submit to the mining industry rather than trying to advise it, so miners have started seeing themselves as the arbiters of which tx formats are harmful, rather than core.
2) Again, for the thousandth time, the goal of spam filtration is not to prevent *every single* spam tx. The goal is to drastically rate-limit spam, which the opreturn filter demonstrably does.

>It’s baffling to me that you keep asserting relay-based discouragement works and will continue to work, despite blatant evidence that at least in some cases, it is trivially bypassed.

I really wish you would show me what evidence you're looking at. Again, the evidence I'm looking at says that filters reduce spam by at least 3 orders of magnitude. And again, the fact that a few spam txs slip through here and there is completely normal and fine. You wouldn't eliminate your spam filter just because a few spam emails slip into your inbox. This line of reasoning is quite absurd, yet I keep hearing it from the core side.

>How do you explain that this appears to be the case for OP_RETURNs, and does not appear to be the case for sub-1sat/vb transactions?

1) ">1-sat/vB transactions" are still much more popular than subsat txs. So the filter is still working. Not as well as the opreturn filter is, but
2) cheap txs are not inherently harmful. minrelaytxfee was probably due for a reduction anyway
3) the 0.1-sat/vB-1-sat/vB filter is the cheapest filter to intentionally use as a demonstration that "filters don't work", and I suspect this "subsat summer" thing was done intentionally to act as a rhetorical device in this conversation. It is really nothing more than that, though. The situation with the opreturn filter is very different, which is why the same attack would not have worked if attempted against that filter, and the filter instead needed to be *pre-emptively removed* in the core repo.

>I think a more likely explanation is that either (1) those with demand for larger OP_RETURNs haven’t yet developed the infrastructure to bypass common node implementations yet

Opreturns are just a lot more expensive; that's sufficient to explain the difference. Though, notably, now that core has eliminated filter for txs 0.1-1sat/vB, attacking the opreturn filter will be much cheaper.

The other notable difference, of course, is that there would be backlash against miners for confirming high-volume (or illegal-content) opreturn spam, whereas cheap txs are not inherently harmful (though there would be a backlash against abuse of subsat txs as well).

>or (2) there just isn’t that much demand for them because there are cheaper ways to do data storage

There isn't much demand for large opreturns because *there is a very popular filter that filters them*, so *any shitcoin metaprotocol designer would have to be insane to use large opreturns as the tx format*. As soon as this filter is eliminated, demand will inevitably skyrocket.

>To be clear, I don’t think it really matters to what extent that is the case or why, as I don’t think today large OP_RETURNs, or other data storage demand is in itself harmful to nodes or Bitcoin at large.

Are you not concerned that IBD was made measurably more expensive during the 2023-4 spam attacks, and that utxoset growth was only 3.5GB/year during that attack, while the limit is ~52GB/year? It seems to me that all that is necessary for bitcoin nodes to continue to become more and more expensive to run is for more and more spam attacks like brc20, which seems to be exactly what removing the opreturn filter will do.

And even if we fix utxoset scaling (libbitcoin is promising here), there will always be the threat of high-fee data spam drowning out payments. It's very concerning to me that core devs do not seem to acknowledge the danger of either of these obvious attack vectors.

>I think for data-storage use cases this is misguided, dumb, silly, and/or irrational, but me holding that opinion isn’t going to change anyone’s mind.

I don't think it's irrational, especially while the world is on a fiat standard; again see above for my rationale.

>The market, over time, may.

I think this is purely wishful thinking.

>Fair enough; this needs more nuance. Direct submission to miners works for **transactions that both sides consent to** (transaction creators and miners). The P2P node network can assist or not, but there is little it can do to intervene - all it needs is for the transaction data to make it to the right place, with or without their help.

Sure, but as stated above, we have mechanisms for exerting pressure on hostile miners who repeatedly confirm transactions the rest of the network considers objectively harmful. You seem to be ignoring the concept of negative externalities here, or denying that they exist on bitcoin, which they very much do.

And there's no incentive for me to relay garbage through my node, so even if others don't run the same filters as me, I still don't think there's a convincing argument that I must relay garbage (or run blocksonly, which I'm confused as to why core devs keep recommending, since it surely has the same effects as filtering, but to an even greater degree).

> But what makes it so that miners will generally accept all transactions that pay a competitive fee - the property we actually want for censorship resistance? It’s not too hard to imagine a world where there are just a few large mining pools, which are heavily regulated, and thus not beyond influence from those who might actually want to block or stifle certain transactions. The answer is that anyone can join the set of miners without asking for permission. **This is the entire reason why proof-of-work exists.**

Yes of course, we agree. This is the entire reason spam filters cannot be used for censorship.

>If we would be okay with an immutable set of miners, we could give them some signing keys each, say that every block needs to be signed by a majority, and get rid of the wastefulness of proof of work. We don’t do that because the ability for anyone to enter the mining market, for economic reasons, or more importantly because they don’t like what existing miners are doing, is what fundamentally provides censorship resistance. Miners don’t censor fee-paying transactions, because if they did, someone else could pop up to mine those transactions anyway. Sure, they need hardware, and electricity, and network connectivity, but beyond that, they don’t need to ask, or even tell anyone. They could do so anonymously, from anywhere in the world. The fact that today’s miners choose to largely identity themselves seems weird to me, but apparently the publicity is worth it to them. In any case, they’re not required to keep doing that.

Yep, again, no disagreement from me here.

>However, for this to work, it must also be possible for miners to get competitive access to fee-paying transactions anonymously.

Yep, memcooler can do this, by sending the tx only to the owner of a particular bitcoin address, or many miners who own many bitcoin addresses.

>If direct-to-miners payment rails to miners develop and get used to the point that they form a substantial portion of mining income, this stops being the case, because nobody will bother submitting to a small-scale newcomer miner with a tiny percentage of the hashrate

Again, unless you can submit txs to lots of miners at once. It would be possible to use a web-of-trust both to punish miners that cheat, and to be reasonably confident new miners won't cheat.

There are solutions to all of these things if we just think outside of the box. Even just discussing these ideas could be sufficient to convince miners to knock it off, and prevent their needing to be implemented. But this attitude of "well there's nothing we can do" will certainly do nothing to dissuade miners from misbehaving.

>Companies might pop up that manage pools of unconfirmed transactions and turn them into block templates for miners. An anonymous miner may well need to do without access to these, and their fee income might be restricted to just the ideological transactions others refuse (or are coerced not to) mine - needing to mine at a loss otherwise.

>**This is what I see as the largest threat to Bitcoin in the short to medium term:** the marketplace for block space moving out of the public P2P network, and into centralized channels where they are ripe for censorship. This isn’t exactly a [theoretical concern](https://arxiv.org/abs/2509.16052v1).

Okay, thanks for the link, I will try to read it soon. Thank you for finally stating a rationale for [PR 32406](https://github.com/bitcoin/bitcoin/pull/32406) that sounds plausible; no other core dev I've spoken to has raised concern for any "disease" that justifies this "cure". "Bitcoin could turn into something like Ethereum" is precisely my concern here, so we definitely agree that this is a valid concern that should be avoided at all costs.

Are you stating for the record that avoiding the centralization we see on Ethereum is your primary justification for supporting the removal of the opreturn limit?

If so, why do you expect Citrea's 144-byte-arb-data transactions to cause this centralization of template construction (they are never expected to occur)? (If not, then we're back to square one.)

And why not advocate raising the limit to 160 instead of 100,000? You have already stated that you don't think there's any demand for large opreturns, so why remove the filter entirely and create demand that otherwise would not exist?

>You’re saying node runners will get together on a regular basis, and come to an agreement as to what types of transactions are going to get the seal of approval for P2P relay?

I'm not saying that, no. I'm just advocating what we already do, which is recommend that noderunners and miners run certain defaults, which has worked splendidly for all filters except for a couple of overly restrictive examples (fullrbf and subsat).

>And you think this will get the 90%+ agreement and adoption rate needed for transaction relay to become unreliable?

Yes? This is already what's happening *right now* with the opreturn filter, as stated repeatedly.

>And that those who disagree won’t run their own nodes with different policies, or just develop alternative systems that bypass the P2P network? This seems ludicrous to me, and also doesn’t result in what I’d find interesting.

No, I never said that. If ~10-20% of the network is relaying a tx format, that tx format will be reliably mined. But if 20% agree to relay a particular format, said format is probably not that harmful.

Alternative submission methods can all be countered. Libre Relay can be countered by large numbers of [Garbageman](https://groups.google.com/g/bitcoindev/c/bmV1QwYEN4k/m/k203AfdWAwAJ) nodes, and hostile miners with direct submission APIs can be countered by boycotting these miners.

All of this will only work for the *most objectively harmful* tx formats, like brc20. I don't see any reason to expect miners to continue accepting brc20s once the entire network is filtering them, and demonstrating a willingness to continue escalating until they knock it off.

>Larger miners benefit from having their blocks being slow to propagate, smaller miners suffer. This is easy to confirm with simulations. It is also intuitive I think: larger miners have a larger probability of finding two blocks in a row, and don’t suffer a propagation delay between them.

I don't find this claim at all intuitive, but I am open to looking at data or hearing rationales that convince me otherwise. To me it seems like large miners always have some nonzero incentive to selfish-mine, but I'm not immediately seeing how slow block propagation from filtering makes this incentive significantly larger, especially because such an attack can backfire, whether accidental or intentional.

But if it's true that propagation delays are always centralizing, and that the opreturn filter is causing this centralizing force to be significantly worse than otherwise, then of course I'm open to revising my position. But I'm just not seeing it, especially since you yourself claim that large opreturns have no demand.

>>Also, blockreconstructionextratxn helps here.

>Only to nodes which have peers whose relay policy matches that of miners. It seems highly hypocritical to me to not want to relay certain transactions yourself, but still rely on having peers that do relay those transactions to you so your reconstruction speed is fast. Put otherwise, the extra pool only helps with policy divergence in situations which more or less correspond to the filtered transactions propagating well anyway.

Yes, but if I don't have *any* peers that propagate a particular transaction, then isn't the probability of that tx being mined much lower anyway?

-------------------------

coinjoinkillua | 2025-12-15 12:53:07 UTC | #17

> I very much doubt that a supermajority of noderunners could be convinced to filter OFAC transactions, for example. But again, if that does somehow happen, there’s always direct-to-miner submission APIs (and other more decentralized solutions such as gudnuf’s hackathon project, memcooler\[5\]).

I agree 💯

I believe in this so strongly that I think it will be trivial for you to convince a supermajority to filter "spam".

The core client should not have arbitrary filters enabled by default and It should not reinforce the social stratum or any subjective aspect of the protocol, Because this sets a dangerous precedent, while users cannot be forced to accept OFAC rules, developers do not have that same privilege, especially if they create a precedent. 

I think that solves everything, you're right, you're just looking for a shortcut and using the wrong means to achieve an end.

-------------------------

