# Is it time to increase the blocksize cap?

myles | 2024-06-03 17:21:52 UTC | #1

Bitcoin needs to scale in order to achieve the vision of freedom money. Here are some thoughts that just don't make sense on the current trajectory we are on right now:

- Lightning channels require at least 2 on-chain transactions to set up. If fees even stay at their CURRENT prices (which, I predict they will increase as the demand for block space increases), then that is widely unaffordable for most people in the world. Sure, you could interact with the network through a third party, but that defeats the purpose of Bitcoin. How can a $5 per week family in Ethiopia self-custody Bitcoin in its current state?

- Many people make the argument that the base layer should be reserved for governments, banks, and major companies and that regular people will be exposed to Bitcoin through these entities. What ever happened to not your keys not your coins?

- When did this notion that everyone needs to run their own full node become sensible? Did SPV nodes disappear? If our storage and bandwidth technology has been upgraded massively since 2009, why are we still at 1MB blocks?

- Why are a handful of developers allowed to decide what's right and wrong for Bitcoin? Shouldn't block size be determined by the free market like everything else in Bitcoin? When did we move away from our Austrian ideals and into autocracy of Bitcoin Core developers?

I may be wrong about some, or all, of what I just said but from my 12+ years of experience in Bitcoin, these are just some of the things that have not made sense to me personally. I understand the appeal of molding Bitcoin to look favorable to banks and governments, but I really believe in Bitcoin as a tool for human rights and individual freedom and see it more than just a way to "get rich quick". I hope that we can, as peers, move it back on the right track.

-------------------------

CubicEarth | 2024-06-03 19:40:34 UTC | #2

Yes, it is.

The problem is Bitcoin's philosophy today is heavily influenced by the fact that there was a "blocksize war", and many of the advocates for increasing on-chain capacity left bitcoin development or were driven out. It's a form of political polarization that we are left with.

The tiny-block arguments are full of contradictions.

• That the cost of running a node is all that matters, while ignoring the cost to transact

• That small blocks are critical for decentralization (which only matters here for purposes of censorship resistance), while acknowledging high fees mean Bitcoin will primarily be used by large institutions and governments (who ARE the censors)

• Thinking that IBD time must be quick (even though regular users are priced out of transacting). Institutions aren't going to not run node because syncing takes 4 days instead of 1 day.

•  Arguing that smaller blocks maximize fee revenue for miners. That is just wrong. There is an optimum block size that would maximize fee revenue, which would be neither too big nor too small.

• Arguing that smaller blocks are more inclusive, because they make it easy for people with bad internet to run a node. The fact is tiny blocks put an absolute cap on access to the base chain, excluding 98% of humans, whereas making running a node only feasible for people who have better than circa-year-2000  3.0 Mbps internet only excludes the 20% of people on the planet who aren't well connected.

And bad-faith arguments too:

• Acting like anyone not in favor of tiny-blocks is a "big-blocker" and should use BSV or some garbage.

• Arguing that there isn't a point to increasing block sizes because for any reasonable increase, there will still be unmet demand. (Which BTW, is why fee revenue would go up with marginally larger blocks (to a point)).


And this misguided ones:

• Somehow pretending that because other layers are and will be needed, that there is no benefit to increasing layer-1 capacity.



I think to have a more concrete discussion around this topic, it would help to pick a couple of few block sizes and analyze their potential real-world impacts. What would they actually do to IBD time? And bandwidth and storage requirements? How many new Lightning users could be accommodated? What kind of fee-modeling can be done?

IMO 8 MB, 16 MB and 32 MB block weights would be the reasonable numbers to study. Assuming that we are at ~4MB right now.

Edit: 

Blocks should be as large as they can safely be.

-------------------------

MattCorallo | 2024-06-03 19:53:06 UTC | #3

[quote="myles, post:1, topic:941"]
If fees even stay at their CURRENT prices (which, I predict they will increase as the demand for block space increases)
[/quote]

If there were a way to be incredibly confident in this, then this kind of conversation would probably make at least some sense, but I see no reason to be confident in this at all. Much of the current blockspace demand is for things that have proven to be incredibly short-lived in other blockchain ecosystems, so suggesting that it is long-term-sustainable on Bitcoin seems naive to me.

-------------------------

myles | 2024-06-03 20:02:00 UTC | #4

Well there are 2 scenarios:

1 - More people join Bitcoin as time goes on and more people begin to transact on it

2 - Bitcoin fails

If the goal of Bitcoin is to become the world currency, then there exists a state in the future in which billions of people are using it for transactions every day. Let's be EXTREMELY conservative and say there are 10 million transactions worldwide every day. If BTC can handle only 700,000 per day, then that means every day, an additional ~9 million transactions are unable to be confirmed. It's very likely in this scenario, big banks would bid up their own transactions to hundreds, if not thousands, of dollars in order to continue normal operations. This would then mean all UTXOs under $1000 of BTC would become dust.

As to the 2nd scenario, if Bitcoin is not the world currency, it has, by definition (and by vision) failed or has yet to reach its potential.

-------------------------

cguida | 2024-06-03 21:33:46 UTC | #5

NACK.

Anyone advocating for raising the block size is either ignoring the existence of Lightning or has never run a Lightning node on low-resource hardware.

We need to lower the block size instead.

See https://www.youtube.com/watch?v=CqNEQS80-h4 for a strong argument as to why lowering the block size would be beneficial.

-------------------------

ProofOfKeags | 2024-06-03 22:30:05 UTC | #6

I tend to agree with @MattCorallo here in that while we are working hard to increase demand for block space across a variety of efforts, we don't really have reason to believe that block space demand is a foregone conclusion. Changing the block size limit has very non-linear effects on fees too so it's very *very* hard to predict how a change to the block size would impact the fee market.

That said, I do think we should begin to normalize the discussion of expanding block size capacity. I think that in the '15-'17 era it was good for us to take a hard line stance on the issue but I think the wider community has a very reactionary response to even suggesting it and I think we can unwind some of that by talking very openly and honestly about what it might solve and at what expense.

The main uncertainty I'd like to make everyone aware of is that doubling the block size won't necessarily cut fees in half. All market prices develop as a clearing some supply against some demand. Bitcoin's block space supply schedule is *perfectly inelastic*... no amount of increased prices will invite any sustainable increased supply (modulo consensus forks that... raise the block size 🙃). On the other side of it, the user demand elasticity is fairly immeasurable. Especially in the presence of more sophisticated RBF procedures (like the one we shipped in LND 0.18), the visible fee market isn't necessarily representative of the overall willingness to pay fees.

You can imagine a situation where there is *exactly* 4MWu of transactions every ten minutes that has a willingness to pay 1ksat/vb for their transaction, but has the sophistication to start with 1sat/vb and RBF their way up to ensure they are in the next block. If you had a block space budget of 4MWu + $e$ per block, then the fees would be 1sat/vb, and if you had a budget of 4MWu -- $e$ you would end up with fees at 1000sat/vb as the participants RBF their way into winning that bidding war. This example is admittedly contrived, but the point I'm trying to convey here is that a miniscule increase to the block size can wipe out the entire fee market in certain conditions.

As such, using observed fee levels to try and decide on a new block size limit parameter is likely not a good approach overall. If we were to try and fuck with the block space supply schedule I'd like to see something that changes the *elasticity* of the block supply as opposed to something that kept *perfect inelasticity* and just changed the available supply.

I think this is an important conversation to normalize, though. I don't think that in 2024 we should mindlessly reject conversations on the subject like it's still 2017.

-------------------------

CubicEarth | 2024-06-04 03:23:58 UTC | #7

I agree with the premise of [myles](/u/myles)'s reply. We need to plan for Bitcoin's success. Planning for its failure, especially if that means limiting its reach, isn't the way to go here.

**Editing** (posted early by mistake)

With regards to total fees and block space:

First of all we need to acknowledge that no one has any idea what the appropriate security budget of Bitcoin ought to be, and there certainly is no consensus on the answer even if someone knew. So I believe it is inappropriate to try to engineer other variables with the aim of attaining some specific yet unstated and un-agreed-to security budget.

Second, even if we had a number in mind (or even a budget as a percentage of an economic metric), no one can make the claim that a marginal increase in the size of blocks would lead to less total revenue for miners because that is unknown. Personally I would be happy to wager that a marginal increase in block size would yield more revenue for miners.

Third. Relating to "planning for success" - there is the concept of "induced demand". If we provide the capacity, people will find use cases for it that they are willing to pay something for. Aside from positive or negative impacts to fee revenue for miners, there are real-world costs to storing, transmitting, and analyzing chain data. It would be worthwhile to have analysis on how much larger blocks would cost the nodes, collectively, in extra resource consumption.

Fourth. It is worthwhile to reason the other way around as well. What would be an amount that a person at the median income of the world (or top 10% or bottom 90%th) would likely to be happy to pay to make a LN tx, or other "settlement size" TX on Bitcoin? Maybe $1, or $5. Using the naïve and old school 7-tx/s with 1 MB blocks, that would be 4,200 txs/ block. If we had 32 MB blocks, with 134,000 txs @ ~$1.50 / tx, that would be $200,000 per block in fees. Is that enough security budget?

-------------------------

shocknet_justin | 2024-06-04 15:35:40 UTC | #8

Never.

A batch open of 50 channels takes ~2k vbytes

A day has 144k vbytes

Thus we could open 3M channels a day, or 1BN channels **per year**

Nodes willing to wait days for a channel batch may get them **nearly free** as they're subsidized by larger and high time preference ones

Undoubtedly closures will cost more than opens, but that's also optimal from an incentives perspective

> are a handful of developers

Seems you have no idea how any of this works, good luck changing the network with a handful of developers

> Bitcoin as a tool for human rights

Sound money is a human right, everything else is downstream of that.

> a $5 per week family

Bitcoin being better money doesn't help people that **don't have money**, it does however help them earn it permissionlessly.

-------------------------

CubicEarth | 2024-06-04 16:38:28 UTC | #9

[quote="shocknet_justin, post:8, topic:941, full:true"]
Nodes willing to wait days for a channel batch may get them **nearly free** as they're subsidized by larger and high time preference ones
[/quote]

The same can be said of user who want a very cheap node. Syncing will eventually happen, if they are patient and wait (true now, and also with somewhat larger blocks).

-------------------------

myles | 2024-06-04 17:00:31 UTC | #10

[quote="cguida, post:5, topic:941"]
er the block size instead.
[/quote]

So let's just assume that every person on earth will only ever open ONE channel and never will close it. That means it will take 7 YEARS to onboard everyone onto lightning?

-------------------------

myles | 2024-06-04 17:03:50 UTC | #11

[quote="shocknet_justin, post:8, topic:941"]
Bitcoin being better money doesn’t help people that **don’t have money**, it does however help them earn it permissionlessly.
[/quote]

$5 a week IS money. Many people around the world live off of much less than we do in the United States and it's ignorant to kick them off the network because they are "too poor".

-------------------------

shocknet_justin | 2024-06-04 17:25:46 UTC | #12

[quote="myles, post:11, topic:941"]
$5 a week IS money. Many people around the world live off of much less than we do in the United States and it’s ignorant to kick them off the network because they are “too poor”.
[/quote]

No, it isn't, that's simply your fiat unit bias and nothing more. Is .005 US cents money too? Just make up any integer, right? It's a product of a nonsensical contrived example.

It's no coincidence that appeals to the security and sovereignty of the chain for arbitrarily insignificant amounts is the only talking point big blockers have left, a dishonest one. 

Your hypothetical user would receive something no more sovereign or secure than a database entry had you your way regardless, for if Bitcoin were to be so fragile as to shift with the winds of virtue stroking fantasies it would be worthless.

Regardless, I already showed the math above, lightning channels can be near free in perpetuity

-------------------------

myles | 2024-06-04 17:34:28 UTC | #13

[quote="shocknet_justin, post:12, topic:941"]
Your hypothetical user would receive something no more sovereign or secure than a database entry had you your way regardless, for if Bitcoin were to be so fragile as to shift with the winds of virtue stroking fantasies it would be worthless.
[/quote]

What are you actually even talking about? It doesn't matter what the unit of currency is... if you are unable to afford a lightning channel, you are basically kicked off the network because on chain transactions fees are too high. It doesn't matter if I say they can't afford it in Yen, BTC, BSV, slices of bread - lightning is vastly unaffordable for MOST of the world. Seems like you dodged the entire premise of my argument.

-------------------------

shocknet_justin | 2024-06-04 17:36:52 UTC | #14

> Seems like you dodged the entire premise of my argument.

You made no argument... you're using subjective terms 

How much is too much? What is it now? What should we get it to?

I've already showed you channels are effectively free.

-------------------------

cguida | 2024-06-04 17:38:17 UTC | #15

> So let’s just assume that every person on earth will only ever open ONE channel and never will close it. That means it will take 7 YEARS to onboard everyone onto lightning?

Yes. If we double the block size it will still take 3.5 years according to your numbers. Meanwhile the utxoset has [exploded from 5GB to 11GB](https://statoshi.info/d/000000009/unspent-transaction-output-set?orgId=1&refresh=5s&from=now-5y&to=now&viewPanel=8) in just the past year. Utxoset growth is dramatically outpacing Moore's Law, with affordable low-resource hardware becoming more and more difficult to sync. The 1MB non-witness block size limit is the only thing keeping this from expanding even faster.

We need small merchants to be able to run their own nodes, otherwise the notion of "economic nodes" being a check on mining pool power is a complete LARP.

Raising the block size is an unserious solution to scaling. We don't need incremental improvements; we need *fundamental* improvements, such as the new tech covenants enable. Let's focus on real solutions, not these "screw our future selves for a momentary reprieve" ideas.

-------------------------

myles | 2024-06-04 17:42:57 UTC | #16

[quote="shocknet_justin, post:14, topic:941"]
I’ve already showed you channels are effectively free.
[/quote]

No you haven't. You showed a hypothetical, massively unrealistic, situation in which only 3 million edges are added to the Lightning network per day. We have 7 billion people on this planet. If you really want each and every one of them to be able to reap the benefits of ultra sound money, they all need to be able to open and close multiple channels. Assuming they all open just ONE channel (for argument's sake, although one channel for a lifetime is unrealistic), it would take 7 years to onboard the planet.

-------------------------

myles | 2024-06-04 17:44:53 UTC | #17

[quote="cguida, post:15, topic:941"]
The 1MB non-witness block size limit is the only thing keeping this from expanding even faster.

We need small merchants to be able to run their own nodes, otherwise the notion of “economic nodes” being a check on mining pool power is a complete LARP.
[/quote]

How do you know this? Why have developers assumed they know what's right and wrong for Bitcoin? The free market is the best decider of what works for the free market.

-------------------------

MattCorallo | 2024-06-04 17:58:57 UTC | #18

[quote="ProofOfKeags, post:6, topic:941"]
I think that in the '15-'17 era it was good for us to take a hard line stance on the issue
[/quote]

What hard-line stance? There was almost no "keep the block size the same" camp, and ultimately the blocksize was increased.

-------------------------

shocknet_justin | 2024-06-04 18:05:14 UTC | #19

You can't speak about unrealistic hypotheticals when your case is 7B on-chain users

There's about 300M businesses and 1B households... channels scale infinitely more so than cars, housing, or any other fixed appliance

Every economic entity could have a channel in the next year

-------------------------

ariard | 2024-06-05 00:38:14 UTC | #20

Whatever one's philosophical belief about bitcoin can be, i.e as a freedom money or as a human rights and individual freedom tools or as means to be free to transact with who you wish, and whatever the strength of your belief this won't make you free from hard constraints such as physics, economics, mathematics and world geography.

We can certainly hold the goal that allowing a $5 per week family in Ethiopa to self-custody bitcoins as a laudable goal, however zooming out this naively miss that getting scalable bitcoin blockchain _access_ to verify said self-custodied bitcoins isn't a solved problem. There is still no SPV-like Chain-state Delivery Network (see [“On the scalability issues of onboarding millions of LN mobile clients”](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-May/017818.html)) on top of the current network of full-nodes.

While there is a wide range of existent self-custody solutions to allow you to keep your keys and your coins, as soon as you start to have sophisticate spending policy in place, you're running in a conundrum of auxiliary problems (just-in-time fees reserves, 24 / 7 monitoring hosts, backups). Those problems exist independently of the coins holders being a government, a bank, a major company or an individual, and they are quite orthogonals w.r.t block size.

While there is indeed a wide range of considerations in running one or more full-node or not, and those considerations have to be evaluated more carefully if you're running a second-layer node on top of your full-node, in a world where there is a already well-known DoS surface about current block size (a.k.a "The Great Consensus Cleanup"), it sounds more wise to address first those concerns to preserve the robustness of the base-layer network.

Abstraction made of all security issues for once, lightning has a merit has a scaling technology to exist and being deployed. It's very clunky and buggy, and one could say it's worth a [Trabant](https://en.wikipedia.org/wiki/Trabant) as a bitcoin scaling solution. For now, lightning is the best Trabant we have. In the future, if one consider a decades-long timeline, there are known N > 2 counterparties "cypherpunk" scaling solutions like [CoinPool](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-February/019968.html), coming with its own mountain of consensus changes to make it a reality.

As of today, it sounds still realistic to get 50 to 70 millions of channels over the next ~5 years on lightning, this is far from being at capacity and the bottleneck sounds more to be smarter automated liquidity management to keep the average on-chain fees cost predictable by the end-user. True, the $5 per week family in Ethiopa might have to wait a bit longer to get access for "cypherpunk" scalable payments and lightning might have been over-sold by its marketing department.

Lastly, it is a bit easy to blame Bitcoin process decision-making to have degenerated in an autocracy of Bitcoin Core developers. This is certainly true there is an inertia towards entitlement among developers once in place. For some, this entitlement comes to argue whatever technical opinions of the day without consistency claiming some decade-old "domain experts” privilege. Often, it can give the appearance to an external observer it's more justified by complacency or laziness than a thorough first-principles reasoning.

On the other hand, most of Bitcoin Core developers have put the years of work to get the experience and skillset which gives them the "street cred" to see their technical opinions getting echoed among the community. This is correct than asking for people to get scars and wounds for their code and design contributions before to see their technical opinions getting real impact could render the Bitcoin upgrade process utterly conservative. It’s more a matter of equilibrium to maintain a reasonable risk-adverse upgrade process.

To finish on a philosophical note, if one sees Bitcoin as an atemporal store of value, under an aristotelian perspective, network consensus stability is a very valuable property in itself.

-------------------------

azz | 2024-06-18 06:44:43 UTC | #21

NACK

[quote="shocknet_justin, post:19, topic:941"]
Every economic entity could have a channel in the next year
[/quote]

They certainly could, however increasing the L1 block limit probably isn't the way to do this. When Lightning's original paper came out, it was already known that it wouldn't scale to 7 billion occupants. Instead of modifying L1 further, it might be better to design a layer either between the timechain and lightning or on top of lightning. 

Does anyone pushing big blocks run a node themselves? The main issue with, say, 32MB blocks is just that. It's not that most consumer hardware couldn't handle this (though it makes decentralisation over low power, non-traditional IP networks and low bandwidth ones more difficult), but the storage requirements would change enormously. 32MB a block, 144 blocks a day is a **4.608GB per day storage requirement**. My node currently is storing ~658GB (ignoring electrum indexes too). With 32 byte blocks, my hardware could totally handle it, but you'd rival the entire chain size in less than half a year. I can't handle buying 1TB drives every 7.12 months, and I doubt many other operators would be fond of this either. 

Maybe investigate opcodes that don't cause issues to build additional layers, or layers without new opcodes (best, really), but I don't think this is viable. Not on the base layer.

-------------------------

CubicEarth | 2024-06-21 01:39:46 UTC | #22

I have multiple full nodes, some pruned, some unpruned, and I have been running one them since 2013. I don't think we should go straight to 32MB blocks, but we can examine the consequences of them as a worst case scenario.

4 TB SSDs, even with the recent run up in prices, can be had for $200, which is $0.05 / GB. So *if* you feel the need to run an archival node, and *if* you feel the need to serve it from and SSD, you storage costs would be $0.25 / day or $90 / year. If you prune then storage costs don't matter, and if your application allows for a spinning disk hard drive, storage costs would be 1/3 of the SSD, for about $30 a year.

We should keep in mind that while some people do need to run archival nodes, most do not. It is also worth considering that there is every reason to believe that SSD prices will eventually continue the steep downward trend in price they have been on for the past 10 years.

I'll admit that I would not be a fan of 32 MB blocks today, especially since at first they would just be filled with low fee junk.  Considering what we have seen with ordinals it would probably be better to start small, with just a doubling to 8 MB. Then a 4 TB SS would be sufficient for the next 15 years.

Who is the person who would prefer to pay $50 each time they transact on chain rather than a once per year $22 in storage costs?

-------------------------

ZmnSCPxj | 2024-06-29 08:50:16 UTC | #23

As I understand it, persistent storage is not the issue with block size.  The main issue is rather the time it takes to propagate a new block from a miner.

A large miner would strongly prefer to withhold blocks from the smaller miners of the network, only releasing blocks when the smaller miners have found one, in order to orphan the blocks of the smaller miners and effectively remove their hashrate from the network.

The above can be done deliberately, but a well-connected set of mining nodes with low latency with each other can do this accidentally to less-well-connected sets of mining nodes.  And the larger the block size, the more likely such an accident would occur, and the greater the pressure to centralize mining arises.  It all starts with well-connected mining nodes, then colocated mining nodes, then co-owned mining nodes.

Another objection is that every block must be transmitted to every other validator.  This is in fact the first objection in the first ever reply to the bitcoin.pdf paper on the cypherpunks mailinglist: each block needs to be sent to each other participant in the fullnode network.  Yes, you can argue that not everyone has to directly participate in the fullnode network.  Nevertheless, the argument "we should increase block size by N times" implies "we expect N more fullnodes on the network sending N times more data to each other for an N^2 times total global bandwidth consumption" which seems the opposite of "efficiency", and ultimately all costs incurred by the network must by necessity be paid for, including bandwidth, and paid for they shall.  Increasing block size is a massive decrease in efficiency and is the opposite of what you ultimately want.

On the other hand: can we at least ***first*** try to discover ways to scale Bitcoin ***without*** ever increasing block size?  It has worked well enough so far (it has been 8 years since SegWit increased block size) and I can comfortably use Lightning Network today.  I admit I am a nerd and can use Electrum on desktop comfortably for LN (pointing to my electrumx instance on my basement tower server with the fullnode), but with enough effort and iteration I believe this can be made a lot more comfortable to most people, including most people living in my country where 5 USD a week would be an onerous burden.  I am also working on figuring out how to get the last mile more comfortably onboarded without as much onchain footprint, which translates to less resource use and therefore savings that ultimately get passed on to end-users.

Ultimately, we should focus on reducing resource use of non-mining-related resources; the consumption of energy by mining ***is*** the security provided by mining, but this does not apply to the rest of the system, including resources spent on sending mined blocks.  Increasing the block size increases total resource use.  We should be ***restricting*** who can see transactions (the way Lightning Network does, which is why it is a scaling solution, unlike block size increase), not increasing how many publicly-visible transactions can be seen by everybody.

-------------------------

CubicEarth | 2024-07-10 18:38:47 UTC | #24

[quote="ZmnSCPxj, post:23, topic:941, full:true"]
As I understand it, persistent storage is not the issue with block size.  The main issue is rather the time it takes to propagate a new block from a miner.

A large miner would strongly prefer to withhold blocks from the smaller miners of the network, only releasing blocks when the smaller miners have found one, in order to orphan the blocks of the smaller miners and effectively remove their hashrate from the network.

The above can be done deliberately, but a well-connected set of mining nodes with low latency with each other can do this accidentally to less-well-connected sets of mining nodes.  And the larger the block size, the more likely such an accident would occur, and the greater the pressure to centralize mining arises.  It all starts with well-connected mining nodes, then colocated mining nodes, then co-owned mining nodes.
[/quote]

It is true that larger blocks are a centralizing pressure upon miners. But there are several questions at play that must be answered before we could decide if a centralizing force was to be avoided or not. One such question relates to the goals of decentralization itself, and whether slightly more or less centralization of miners has any practical effect on Bitcoin's censorship resistant qualities. All else being equal, less centralization would be better, but we must consider the question a matter of cost-benefit.


Another question relates to the balance and magnitude of other centralizing/decentralizing forces that operate on miners, and how propagation delays would affect the result. For instance, it is a fact that stranded and excess electricity are geographically distributed around the globe and across different political regimes. And more generally, cheap electricity is also well distributed. Losses to small miners from propagation delays would need to be large enough to offset higher electricity costs incurred by co-locating to where propagation losses would be smaller. Aren't the size of these costs currently an order of magnitude or two apart from each other, with electricity costs dwarfing propagation delay losses in miner's location considerations? And to clear up an earlier point - there is only a finite quantity of free or low cost electricity in each location where it is present. This is a very strong force keeping miners geographically distributed.

[quote="ZmnSCPxj, post:23, topic:941, full:true"]
Another objection is that every block must be transmitted to every other validator.  This is in fact the first objection in the first ever reply to the bitcoin.pdf paper on the cypherpunks mailinglist: each block needs to be sent to each other participant in the fullnode network.  Yes, you can argue that not everyone has to directly participate in the fullnode network.  Nevertheless, the argument "we should increase block size by N times" implies "we expect N more fullnodes on the network sending N times more data to each other for an N^2 times total global bandwidth consumption" which seems the opposite of "efficiency", and ultimately all costs incurred by the network must by necessity be paid for, including bandwidth, and paid for they shall.  Increasing block size is a massive decrease in efficiency and is the opposite of what you ultimately want.
[/quote]

Indeed there are N^2 dynamics at play, which can quickly cause problems if we aren't careful. So we need to be careful. 

I disagree that we need to consider the 'efficiency of the network as whole' in the way you are advocating. Instead, we primarily ought to consider current and potential end users, and if the costs and benefits weigh out for them to indeed become or remain a user. And generally, we ought to want Bitcoin to a viable choice for transacting for as many people possible, for both practical and philosophical reasons. It is not a thing that individuals worry about their impact on "total global bandwidth consumption". Not at all. And for the professionals involved in keeping the internet up and running, the push is for more fiber, faster switches and overall better connectivity. Not the rationing of bandwidth. There are parallels to your point of blocksize N^2 scaling with advances in the Megapixel counts for smartphone cameras. Higher pixel counts lead to better photos (taking up more storage), but encourage people to take more photos because of the better results. Indeed, on my smartphone today I have *checks phone* 236 GB of photos and videos. Which is almost embarrassing, but no one argued against better phone cameras for fear of this outcome.

Perhaps it is a reason why phone cameras aren't 200 MP already, because storage does matter and diminishing returns exist. But N^2 consumption of a resource doesn't mean an approach is flawed, or that it increasing N has no benefit - it just means that over some relatively small range of N consumption of available resources goes from trivial to manageable to unworkable.

In terms of viability of the network as a whole we should keep in mind that not all nodes serve blocks, and so the burdens of the N^2 network traffic can be concentrated. However, for this type of question, we ought to use the measure of the cheapest bandwidth available, since there is no reason nodes that block serving nodes couldn't locate in those places. For instance in Switzerland there are 10 Gbit/s fiber to the home plans for $100 usd / month, which could serve 2,500 Terabytes per month of data at a rate of 1 GB / second. Which works out $0.04 / TB transmitted. Bitcoin's chain is currently just under 600GB of data, or ¢2.4 in server transmission costs to do an IBD. It would cost $480 total to provide data to all of the 20,000 currently existent nodes for their IBD. The 17GB / month to keep a node in sync with 4MB blocks @ 20k nodes would total 332 TB, at a cost of $13. $13 to feed to the entire trillion dollar-plus network with data for a month. Multiply this by 10 and _then_ square it and it would still be a completely trivial cost.

I don't disagree on the importance of Lighting, or other scaling layers. And I do agree that the base chain will never have the capacity to settle even a meaningful fraction of global demand for transactions. But neither do either of those truths somehow suggest we shouldn't increase the on chain capacity to its largest practical and safe amount that current conditions allow for. We should. And today that amount is larger than the current blocksize.

[quote="ZmnSCPxj, post:23, topic:941, full:true"]
On the other hand: can we at least ***first*** try to discover ways to scale Bitcoin ***without*** ever increasing block size?  It has worked well enough so far (it has been 8 years since SegWit increased block size) and I can comfortably use Lightning Network today.  I admit I am a nerd and can use Electrum on desktop comfortably for LN (pointing to my electrumx instance on my basement tower server with the fullnode), but with enough effort and iteration I believe this can be made a lot more comfortable to most people, including most people living in my country where 5 USD a week would be an onerous burden.  I am also working on figuring out how to get the last mile more comfortably onboarded without as much onchain footprint, which translates to less resource use and therefore savings that ultimately get passed on to end-users.

Ultimately, we should focus on reducing resource use of non-mining-related resources; the consumption of energy by mining ***is*** the security provided by mining, but this does not apply to the rest of the system, including resources spent on sending mined blocks.  Increasing the block size increases total resource use.  We should be ***restricting*** who can see transactions (the way Lightning Network does, which is why it is a scaling solution, unlike block size increase), not increasing how many publicly-visible transactions can be seen by everybody.
[/quote]

There is not a dichotomy here. It is not one or the other. We can strive to reduce demand for on chain transactions by creating other options that are superior, while at the same time increasing on chain capacity to its safe and viable limits. Bitcoin is an inherently inefficient design, and it has been enormously popular despite that.

-------------------------

Roy | 2024-08-15 09:09:47 UTC | #25

[quote="CubicEarth, post:2, topic:941"]
Arguing that smaller blocks maximize fee revenue for miners. That is just wrong. There is an optimum block size that would maximize fee revenue, which would be neither too big nor too small.
[/quote]

In a network where demand for on-chain volume is fluctuating, then only a dynamical block size limit can be optimal. 

To me the block size debate is all about who pays for the sustainability of the network when the block subsidy becomes negligible.

1. With small block size limit, you have a network with more competition; therefore, transactions paying more. I wouldn't argue that only big institutions could use it. People would just be more encouraged by using a good L2, than L1.  
2. With big block size limit, you don't have as much competition over the long term. (I'd be glad to be proven wrong, if you could give me an optimal block size function.) If you don't have enough competition, people won't pay enough. You'll have to inevitably add tail emission.

As I said, it's all about who pays the price of securing the network. With small blocks, it's people who make on-chain transactions that pay for everyone's coins. With big blocks, you have tail emission, which means everyone pays for the security, even those who don't transact.

-------------------------

qustavo | 2025-03-25 18:54:37 UTC | #26

Given the impossibility to quantify an optimal blockspace demand,  are you aware of proposals suggesting a dynamic blocksize that would work similar to the block difficulty adjustment?

-------------------------

