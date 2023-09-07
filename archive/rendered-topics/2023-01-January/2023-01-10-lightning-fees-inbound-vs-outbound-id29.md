# Lightning Fees - inbound vs outbound

ajtowns | 2023-01-12 07:16:51 UTC | #1

Context: https://twitter.com/renepickhardt/status/1612582052636889088

If a payment going from Alice to Bob has both outbound fees (paid to Alice) and inbound fees (paid to Bob), how do you sensibly negotiate the split between those fees?

Presume first that Alice has complete knowledge about what the demand for the channel is: that is she knows a function $Q(f)$ that perfectly predicts the payment volume that will flow through the channel for a given total fee $f$, so that the total fees will be $f \cdot Q(f)$, split somehow between Alice and Bob. In that case, if we split the fees as $f = a + b$ where Alice receives $a$ and Bob receives $b$, then for any given fee $b$ that Bob quotes, Alice can calculate her optimal fee $a$ by solving $\frac{\partial}{\partial a} a \cdot Q(a+b) = 0$.

If Bob likewise has complete knowledge about the channel demand, he will make the same calculation, resulting in an even split $a=b$. This will usually not maximise total fees collected, instead both sides will raise their fees above the optimal level, as in doing so they get the benefit of increasing their fee but share the losses of decreasing demand with their channel partner.

For example, if $Q(f) = 1000 - 100 \cdot f$ then $f^{opt} = 5$ but $a^{opt} = b^{opt} = 10/3$ so $a^{opt}+b^{opt} = 20/3 = 4/3 \cdot f^{opt}$, while $Q(f^{opt})=1000/2$ and $Q(a^{opt}+b^{opt}) = 1000/3$, so $(a^{opt}+b^{opt})\cdot Q(a^{opt}+b^{opt} = 8/9 \ cdot f^{opt}\cdot Q(f^{opt})$, and Alice and Bob are only receiving an equal split of fee income, but only achieving 88% of their optimal possible fee income.

Generally speaking, this is a stable outcome: while Alice may reduce $a$ if Bob raises $b$, even in the new scenario, Bob remains incentivised to lower $b$ back towards $b^{opt}$.

Without the assumption of perfect knowledge, this changes, of course. In that case Alice and Bob will potentially optimise against different demand predictions $Q_A(f)$ and $Q_B(f)$, with Alice and Bob solving respectively for

$$
\frac{\partial}{\partial a} a \cdot Q_A(a+b) = 0 \\
\frac{\partial}{\partial b} b \cdot Q_B(a+b) = 0 \\
$$

which will usually result in differing values for $a$ and $b$. For example, given $Q_A(f) = 1000 - 100\cdot f$ and $Q_B(f) = 600 - 40\cdot f$, this resolves to $a=5-b/{2}$ and $b = 7.5-a/2$ or $a=5/3$ and $b=20/3$. Further, at those fee rates, $Q_A(a+b)=Q_B(a+b)=1000/3$, so channel activity alone isn't enough for either Alice or Bob to falsify their prediction of demand: if Alice is correct, Bob will make more profit by reducing his fee encouraging more traffic; if Bob is correct, Alice will make more profit by increasing her fee, despite that resulting in a reduction of traffic over the channel.

-------------------------

renepickhardt | 2023-01-10 12:29:07 UTC | #2

Before going deeper into the discussion, can you please elaborate the motivation to use a linear function $Q(f) = a - b\cdot f$  to model the fee / demand function?

-------------------------

ajtowns | 2023-01-10 12:32:10 UTC | #3

That's just an example, and it's because that's an easy example (and if you restrict your range of pricing to a sufficiently small range, most curves end up linear anyway).

-------------------------

renepickhardt | 2023-01-11 21:23:05 UTC | #4

2 comments before digging deepter: 

1.) I really thinkg that $Q(f)=b-a\cdot f$ is a strange choice. I would at least include another non-linearity as fees that are too high will probably result in the channel not being used. I think you kind of reflect this by reducing the predicted flow by a factor proportional to the fees from a given base load. However if the fee is too high this number would become negative and later when finding the optima we would have to restrict our domain to the interval on which the function stays positive. Also I am not sure how negative fees may mess this up as they would increase the base predicted demand.

2.) I don't think this can be studied for a fixed channel. If Alice want's to send to Bob she needs to study $Q_{a,b}^{out}(f)$ as her own inbound fees on all of her channels. Vice versa Bob need not only to look at his inbound fees on the $(A,B)$ channel but his outbound fees on all other channels. All those functions relate to each other and then of course one might have to consider the entire network of nodes and channels and study how changing one channel cascades through the network.

I thought I would leave those high level comments to get your input before trying to extend the observation / model. So what are your thoughts? Mine are that I am a bit afraid that I am overcomplicating the situation.

-------------------------

ajtowns | 2023-01-12 07:00:47 UTC | #5

For (1), if you're contemplating people switching to other channels because your fee is too high, then I think you immediately need something like valves or you just end up with $Q(f) = \bar{q}$ for $f \le \bar{f}$ and $Q(f) = 0$ otherwise, with $\bar{q}, \bar{f}$ being constants that are determined by what the rest of the network is doing, rather than your decisions.

I don't think the details of an actual function is useful beyond simple examples though; in practice it's unknowable (and likely changes over time as lightning adoption changes and other channels are created/closed), and instead you'd be doing point measurements of $Q(f)$ vs $Q(f+\delta)$ (by directly manipulating your channel's fee and measuring change in traffic) or just estimating values (perhaps inferring from what other channels are doing, perhaps as feedback from the impact of what your valves are doing), and then just bumping your fee by $\delta$ if $a\cdot Q(f) < (a+\delta)Q(f+\delta)$.

Note that the partial derivative equations are general, and don't depend on $Q(f)$ being linear.

There's no particular problem with negative fees here: negative fees with positive traffic (or positive fees with "negative traffic") give $a\cdot Q(a+b) < 0$ but that's a worse result than just setting $a=0$ and receiving $0 = 0 \cdot Q(0+b)$ income, so isn't a global maximum. Provided $Q(f)$ is non-increasing, you also won't get trapped at a negative local maxima if you're just doing point-wise optimisation, I think. 

If you're only looking at a single channel, negative fees are only interesting for rebalancing, but that extends your utility function beyond "make the most profit", which isn't considered here.

For (2), in this scenario Alice and Bob are forwarding payments, so the source and destination are external, and the route is predetermined by the sender. Senders can probably be expected to naturally generate some sort of non-increasing demand curve for any given channel; though as above, I expect something like valves is needed for that curve to end up being smooth/differentiable.

I think once you have something of that nature, you can probably imagine modelling all lightning payments across all channels simultaneously, which will give you $Q$ values for each channel; then tweaking fee values for a channel shows you how $Q$ changes for that channel. Calculating a global optimum or a nash equilibrium with that much interaction is probably pretty hard though?

I'm not sure it makes sense to think too hard about inbound vs outbound relationships -- there are $n^2-n$ such relationships, and you only have $2n$ knobs to twiddle (inbound/outbound fees for each channel$, so you're fundamentally limited in how well you can optimise things once you have more than 3 channels, and if you're successful at price discrimination, that just encourages the person being discriminated against to open a channel with someone else to route around you. I don't think I can even come up with a way of using it for price discrimination -- if you want to make it cheaper for X to pay Z through you than for Y to pay Z through you, why not lower the outbound cost to Z for everyone, but raise the inbound cost for Y to compensate, maintaining positive fees everywhere?

But the main point of this was to demonstrate that (a) you can get disagreement on what the optimal fee rate is between channel partners, so a fixed split enforced at the protocol level doesn't make sense, and (b) at least with sufficient knowledge about the state of the world, pure self-interest can result in a stable and fair outcome.

-------------------------

