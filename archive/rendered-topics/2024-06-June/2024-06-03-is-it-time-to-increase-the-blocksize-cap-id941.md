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

shocknet_justin | 2024-06-04 15:33:21 UTC | #8

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

-------------------------

