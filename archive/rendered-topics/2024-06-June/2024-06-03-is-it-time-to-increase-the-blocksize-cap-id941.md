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

