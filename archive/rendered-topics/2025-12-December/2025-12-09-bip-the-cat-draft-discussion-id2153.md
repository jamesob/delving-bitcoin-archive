# BIP The Cat - Draft discussion

Claire | 2025-12-09 23:52:00 UTC | #1

Hello all,

I would like to humbly request review of my proposal “The Cat”:

https://github.com/ostromcode/The-Cat/tree/main

At a high level, it proposes a new approach to curbing non-monetary uses of Bitcoin (for example inscriptions and Stamps-style protocols) that, according to the measurements in the repository, currently account for over 40% of the UTXO set on commonly used implementations. The draft is thorough; before commenting, please see the Rationale and FAQ for common concerns.

The actions suggested by the BIP are, to my knowledge, unprecedented, so I expect them to be highly controversial. I ask that you take the time to consider my motivations in good faith, as I see this as a pragmatic and necessary step to address a real issue we face today. If this is not the best approach, I hope to come closer to finding a better solution through discussion here.

This is still a draft, but I’ve put a lot of work into it. So far it has had the benefit of review from a few close friends and members of the technical community, and I’d now love to hear opinions from the wider Bitcoin development community.

I think discussion is valuable. If it’s a bad idea, I trust the development community will make that clear through open technical review and debate on this forum, in a way that will still benefit all of us.

I have worked hard to keep it simple and to consider edge cases that could impact its viability, though I’m hopeful exposure to this community will reveal issues that have not been anticipated yet.

For the avoidance of doubt: I am not an AI, and this proposal is not the result of an AI. I only mention this because there have been accusations to that effect in the bips repo. I reached out to dozens of people while preparing this draft, many of whom have been very open about their disagreements with the methods it considers. I think they would attest to my humanity and to the care I have put into drafting the proposal.

Thank you for your time and consideration.

-------------------------

billymcbip | 2025-12-10 13:34:20 UTC | #2

Concept NACK. This sets a precedent for consensus-level confiscation of UTXOs.

-------------------------

Claire | 2025-12-15 18:22:37 UTC | #3

Hi Forum, I wanted to take this opportunity to respond to @gmaxwell on this topic. He responded to me on the [Bitcoin Development Mailing List (link to discussion here)](https://groups.google.com/g/bitcoindev/c/Q6ulQb13okg/m/khU9PL1RCwAJ) and for some reason several other replies have been published but my response to him has not. I don’t know if that venue is continuing to censor me but it sure feels like it (I responded on December 11th, since then Greg has had 3 replies approved).

I am thankful that this forum exists as perhaps a more fair place for discussions about contentious issues. 

My reply follows:

Hi Greg, thank you for the response.

When you say I misunderstood your quote, are you referring to the notion that you thought we should try to shape behavior? You also said your reason for caring about motivation in the first place was because you didn't want to give the wrong impression about what was "kosher". Both you and Sipa (who also stated in that thread "The problem is that people may see this as a legitimization of the storage in the first place, and encourage doing so even more." ) seemed to be primarily motivated by the social signals that would be sent by allowing even a small amount of data to be used arbitrarily in the protocol, but felt (from my reading) that it may be worth the trade-off for, as you said, shaping users to more conservative needs.

I certainly appreciate your larger point here, where you state that current use has nullified previously effective means of curbing this behavior, and as you say, sometimes even further increasing it. This is precisely why my proposal exists as it does. It acknowledges that these attempts to "stop" the behavior are not effective against strong financial incentives to overcome them. They would also eventually lead to more harmful abuse; typically this filter discussion lands at the threat of fake pubkey use, which has generally been seen as the most harmful way to include data in the system.

My proposal attempts to destroy trust in those markets, to reduce the desire to spam in the first place.

"Were such a proposal seriously advanced it would likely cause a new flood of transactions both to move to evade it directly and as a result of NFT indexer changes to just "wormhole" the tokens to new outputs after the fact (and a new marketing opportunity for the NFT gifters). NFTs are just an imaginary parallel world that don't depend on the network to validate their activity, so they don't really care about the network's rules, and as such the network's rules have pretty limited effect. "

I understand why you feel that way, but this is not the only way to look at it, and I'm glad we're able to have this discussion. I am not so naive about the behavior of these communities and have some experience with them in the past. You're right: every crypto asset is essentially a narrative.

Where I feel you don't give me enough credit is when you dismiss that the buyers of those narratives are apathetic towards them. One of the largest debates in the ordinal world was the value that the actual numbering system should have at all. The entire point of ordinal theory is that they number and value specific satoshi as non-fungible assets themselves. To suggest that they would be content to assign a new satoshi with their key, I think, underestimates how sentimental they are about their rules. It may seem like a silly delusion, but they obviously value them or they wouldn't spend so much on them.

"The proposed gain is some negligible one time reduction in utxo disk space. You motivated it by claiming this is a 'memory usage' reduction, but it's not-- it's just full node storage in particular as the txouts in question are normally sized and largely quiescent already-- so the savings is pretty insignificant. "

I have a lot to learn, because reading this is a significant challenge to my worldview. I have been under the impression that bloat to the live state (chainstate) is one of the most harmful aspects of things like the fakepubkey threat. This becomes more nuanced when we think of future implementations that don't have a separate chainstate at all, but ignoring those, one of the central reasons stated for removing the recent OP_RETURN limits was to reduce potential UTXO bloat from applications like Citrea. There are several proposals now to address this bloat, and this is the first time I'm hearing someone consider 40% of the current state to be a negligible amount.

"And moreover the proposal would intentionally and knowingly confiscate millions of dollars in funds. This is outright theft, and I believe it makes the idea a total non-starter."

Millions of dollars certainly seems like a lot of money, but these UTXOs are less than $1 on average. It is only due to the massive size of the stated problem that they collectively represent such a significant amount of funds. It's my contention that the users of these satoshi are not intending to use them for their satoshi value whatsoever. They are simply using these as data points tracked in a database to facilitate a gambling trend.

I don't say that to judge the behavior for moral reasons, but to highlight an abuse of the incentives in the Bitcoin network. The defense against dust spam in Bitcoin is the use of a fee market. This is a very effective incentive that makes it expensive and usually pointless to create a lot of smaller UTXOs rather than a few large ones. The problem with the abuse we see from schemes like ord is that they ignore this dynamic by adding an external incentive that pays much more than the satoshi themselves are worth, sidestepping the filter and creating a dynamic that was never imagined by the design.

As an open system, this kind of abuse is hard to stop once it starts. My proposal stands as a way to address it and add new dynamics that aim to reduce the demand for such behavior by understanding it and addressing it in a way specific to its creation, which is to destroy trust in the markets that cause them. I presume you would not even be okay with the potential burning of a single satoshi even if it meant clearing up something people felt was a material problem to the network, so of course you'll certainly take issue with this. I am thankful for you and those like who. It is this uncompromising culture that has continued to keep Bitcoin from becoming like so many inferior projects of the past. 

"As an aside, since the idea fails so easily on basic principled grounds it's a disrespectful waste of participants time to advance a proposal at such length and technobabbly depth which could have been floated with a single sentence "How would people feel about a proposal to discourage NFTs by identifying a snapshot of existing dust-value NFT outputs and making spending them consensus invalid?" or similar. Much of the opposition to LLM use in the field of BIP discussion has been due to widespread misuse of LLMs to create thickets of dense language and obscure non-starter, trivial, or ill considered proposals making them much more costly to deal with and discouraging serious and thoughtful review of *all* ongoing discussion."

This is a very fair point; in hindsight I feel my proposal was much too detailed for this stage of discussion. To be honest, I felt intimidated proposing to this list and wanted to be sure that I had put in enough work to demonstrate the seriousness of my idea. I think I could have done it with a summary that linked to additional details instead.

I did not use an LLM to create my proposal, though I do use LLMs as a tool for research and occasional formatting. I do not think LLMs should be used in place of human ideas. My proposal was written by me (and the friends who helped me) in every part, and only edited for style or grammar by a machine.

-------------------------

