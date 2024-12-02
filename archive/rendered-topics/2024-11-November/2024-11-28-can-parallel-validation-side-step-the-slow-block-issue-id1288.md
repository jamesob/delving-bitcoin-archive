# Can parallel validation side-step the slow block issue?

sjors | 2024-11-28 14:05:40 UTC | #1

The [Great Consensus Cleanup](https://delvingbitcoin.org/t/great-consensus-cleanup-revival/710) (revival) proposal attempts to address worst case block validation time through a consensus change. This is fundamentally a whack-a-mole game, as there could always be some unknown unknown method out there to produce blocks that take a long time to validate.

Every whack takes years and can be difficult to execute without disclosing the vulnerability.

And with every future soft fork we need to deeply worry about block validation time. Although SegWit and Taproot have made it easier to reason about this sort of thing, and entirely new schemes like Simplicity take full account of it.

**It would be really nice if we can make slow blocks a problem only for the miner who produced one.** That would make them a useless instrument for attacks, leaving only a concern for accidents (and protecting miners through standardness rules, but those can be changed more quickly).

Perhaps parallel block validation can achieve this.

The idea of parallel block validation is that if your node sees two blocks competing for the same height, it tries to validate both in parallel and picks whichever it finishes first, which it then announces and builds on.

In the current situation Bitcoin Core will evaluate whichever block is sees first, and completely ignore the outside world until it's done validating. It only matters which block reaches the node first. 

Instead, parallel block validation should cause fast-to-validate blocks to propagate better when they're directly competing with a slow-to-validate block. This increases the stale risk for slow-to-validate blocks, which increases the cost of any attack that involves such blocks.

Pure parallel validation is very complicated to implement. Instead, a simpler approach could be that while the node is busy validating a block, it keeps listening to announcements for competing blocks. If such a block appears, it aborts the ongoing validation and tries the new block instead.

This approach could be problem if a slow block arrives immediately _after_ a normal block (perhaps intentionally). So maybe the following heuristic can be used: keep track of average (weight adjusted) validation time and when validation takes more than five standard deviations longer, abort if a competing block is seen.

The slow block would not be marked invalid. If another block is mined on top of it, we'll try validation once more. But if a third block builds on the fast chain, we abort again, etc.

One problem with this solution is that it is really difficult to build, at least in Bitcoin Core. It would also leave older Bitcoin Core nodes vulnerable, as well as any alternative implementation that didn't implement parallel validation.

But putting that aside, I'm unsure if it actually solves the problem in the first place. In part because I don't fully understand the attacks that people are worried about.

[quote="AntoineP, post:1, topic:710"]
It’s well known maliciously crafted non-Segwit transactions can be pretty expensive to validate. Large block validation times could give attacking miners an unfair advantage, hinder block propagation (and its uniformity) across the network or even have detrimental consequences on software relying on block availability.
[/quote]

Does the large miner advantage stay in place even if their blocks are at increased risk of reorg?

If miners typically run higher end hardware for their node, does that mean slow blocks propagate just fine between miners, with the other nodes in the network being out of the loop?

Does it create an incentive for miners who trust each other (or collude) to skip validation so they always build on the first-seen block, ignoring a fast-to-validate competitor block?

Does this accidentally introduce an incentive to produce empty blocks? Or is the five-sigma threshold enough to prevent that? There used to be a slight advantage for smaller blocks due to the time it takes to transmit the bytes, but that's been thwarted by compact blocks (when they work). Slow validation seems to be an orders of magnitude bigger bottleneck than bandwidth (seconds or minutes vs. a few dozen milliseconds).

I'm unsure how to best analyse this.

-------------------------

evoskuil | 2024-11-28 16:08:13 UTC | #2

Concurrent validation of competing blocks would only make sense if they had the same amount of work. I'm have no idea how common this is. Given that it doesn't matter to consensus which one of two equal-work blocks is confirmed, it could make sense to validate them both and keep the one that validated in less time. There is nothing wrong with discarding the top block for another of equal work - for any reason.

Given the complexity and performance cost of attempting to synchronize the problem between two (or more?) blocks, I would take a closer look at the objective. If the objective is to propagate the faster block of the same work, then it is just as reasonable and a lot easier, to just validate any block that has equal work to the top block. Presently we just ignore that block unless its branch gets stronger than the confirmed branch. If that block validates faster than the top block, then reorg and announce it. This also has the advantage of prioritizing the faster block even if it arrives much later.

Presumably that satisfies the objective. However, I don't think this would make much of a dent in the problem.

-------------------------

sjors | 2024-11-28 16:28:17 UTC | #3

[quote="evoskuil, post:2, topic:1288"]
Concurrent validation of competing blocks would only make sense if they had the same amount of work. I’m have no idea how common this is.
[/quote]

In the extreme case of block that takes 1 hour to validate, there could be six competing blocks at the same height (i.e. same total work). But generally for every second it takes to validate a block, there's a ~1/600 chance of a competing block appearing. I just don't know it that's enough to deter such an attack though.

[quote="evoskuil, post:2, topic:1288"]
If that block validates faster than the top block, then reorg and announce it. This also has the advantage of prioritizing the faster block even if it arrives much later.
[/quote]

It's certainly a stronger measure to slow-block attacks. But it would also significantly change behaviour in a non-attack scenario. E.g. single confirmation would be a lot less safe.

And when a miner sees a new valid block, they have to choose between ignoring it or mining on top of it, based on some estimate of how likely it is another miner would produce a faster-to-validate block. This seems like a can of worms. (though my proposal may have a similar worm)

-------------------------

evoskuil | 2024-11-29 02:24:10 UTC | #5

[quote="sjors, post:3, topic:1288"]
It’s certainly a stronger measure to slow-block attacks. But it would also significantly change behaviour in a non-attack scenario. E.g. single confirmation would be a lot less safe.
[/quote]

The problem you may have here is making a clear distinction between attack and not attack. If the script is inherently an attack, then why not soft fork it out?

-------------------------

evoskuil | 2024-11-29 02:33:14 UTC | #6

[quote="sjors, post:3, topic:1288"]
generally for every second it takes to validate a block, there’s a ~1/600 chance of a competing block appearing
[/quote]

This seems to assume that all blocks of the same height have the same amount of work.

-------------------------

ajtowns | 2024-11-29 03:45:38 UTC | #7

[quote="sjors, post:1, topic:1288"]
Does the large miner advantage stay in place even if their blocks are at increased risk of reorg?
[/quote]

Arguably, it's an advantage to have mined the block, because at that point you know it's valid and can mine on top of it immediately; meanwhile others have to wait until the block propagates and validates. If you're mining empty blocks as soon as the header propagates, you might only be losing fees and adding a mild risk of building on an invalid block, but that's still some advantage for the original miner.

I think making sure core could still quickly do compact block relay (of missing transactions) while the block was being validating, would improve things a bunch. Having a well connected FIBRE relay network would also probably help (as it relays things just based on PoW and doesn't block on validation, aiui); afaik there isn't a public FIBRE network these days. At the moment, I'd expect block relay to be pretty severely impacted by slow validation.

Would be interesting to see the maths on race to first validation; reorging to fastest to validate seems like it would favour empty blocks and perhaps add a little instability (if different hardware validates blocks at inconsistent speeds eg) which doesn't seem super desirable.

-------------------------

sjors | 2024-11-29 09:00:41 UTC | #8

[quote="evoskuil, post:6, topic:1288"]
This seems to assume that all blocks of the same height have the same amount of work.
[/quote]

A reorg more than 6 blocks deep would probably cause more economic harm than a block that takes an hour to validate.

But there's also the edge of a retarget period to consider. An attacker could set the timestamp of the last block as early as allowed so the difficulty increase is maximal. And then also mine the first block in the new retarget period, and reveal both at the same time. Without manual intervention an honest miner would use wall time and thus their two blocks would have less cumulative difficulty.

[quote="evoskuil, post:5, topic:1288"]
If the script is inherently an attack, then why not soft fork it out?
[/quote]

If you know the attack long in advance you can do that. But you might only discover the attack when it happens. That's what I referred to with "unknown unknowns".

[quote="ajtowns, post:7, topic:1288"]
I think making sure core could still quickly do compact block relay (of missing transactions) while the block was being validating, would improve things a bunch.
[/quote]

So that wouldn't discourage slow-to-validate blocks, but simply make them less harmful.

-------------------------

evoskuil | 2024-11-30 07:12:33 UTC | #9

[quote="sjors, post:8, topic:1288"]
If you know the attack long in advance you can do that.
[/quote]

We are aware of such scenarios now. My point was really rhetorical. It is not actually clear what constitutes an attack.

-------------------------

evoskuil | 2024-11-30 07:18:28 UTC | #10

[quote="sjors, post:8, topic:1288"]
A reorg more than 6 blocks deep would probably cause more economic harm than a block that takes an hour to validate.
[/quote]

I think you missed my point. The only blocks that could compete in a legitimate race to validate are those with the same amount of work (or in a fork with the same amount of work). I assume that's not generally common, but I don't know. If it's rare then such a feature wouldn't accomplish much.

-------------------------

AntoineP | 2024-12-02 18:23:17 UTC | #11

Unfortunately not.

When presented a block a miner has the choice to mine on top of it or keep mining on the previous tip.
- If the block is valid, it's in his best interest (and the network's) to mine on top of it \[0].
- If the block is invalid, it's in his best interest (and the network's) to mine on top of the previous tip.

However this is assuming the miner knows instantly whether the block is valid. Until he knows the validity of the block the miner only has two poor choices: keep mining on a likely outdated tip or take the risk of mining on top of a potentially invalid block.

It seems likely that a miner would prefer the first option, by assuming mining on the old tip is wasted anyways and that a miner wouldn't waste a valid PoW on an invalid block. This is also [what we observe(d)](https://bitcoin.stackexchange.com/q/38437/101498) in practice.

The idea you present involves 1) a miner being presented a block, figuring out it's expensive to validate, discarding it, and instead continuing to mine on top of the old tip to then 2) find a normal block at the same height before other miners are done validating the expensive one and 3) other miners being presented with this normal block to start building on top of it in place of the competing expensive block.

Besides all the issues that may be created by implementing 3) and the low likelihood of 2), i don't think we can reasonably expect 1) to happen in the first place. A minority miner does not want to discard a potentially-valid block.

It is also possible, although improbable, that a race between a normal and an expensive block could also happen naturally, but in this case the expensive block is already at a severe disadvantage due to how its propagation is hindered. In this case having miners who received the expensive block first also validate the normal block in parallel would further disadvantage the expensive block. But this is only a marginal improvement in an unlikely scenario, at the cost of high implementation complexity.

Finally i'd like to point out there is a third option for miners when presented a block that is potentially expensive to validate: just stop mining until you are done validating. This is unfortunate for the network. When miners distrust each other this is also more rationale than wasting electricity mining on a potentially outdated tip or a potentially invalid block. Unfortunately what i expect instead is that miners would build tighter trust relationship to reduce their risk of SPV mining. In this sense expensive block validation is an additional mining centralization pressure which i think we should mitigate by bounding the maximum block validation time.

[quote="sjors, post:1, topic:1288"]
And with every future soft fork we need to deeply worry about block validation time.
[/quote]

In any case we always do need to deeply worry about how a new feature may impact block validation time?

[quote="sjors, post:1, topic:1288"]
**It would be really nice if we can make slow blocks a problem only for the miner who produced one.**
[/quote]

Note that even if this resolved the attack against other miners this would also be a concern, in the sense that it would be a new dimension to optimize for when building a block template.

[quote="sjors, post:1, topic:1288"]
Does the large miner advantage stay in place even if their blocks are at increased risk of reorg?
[/quote]

The attacker does not need to be large, although this could degenerate in a race to the bottom where large miners eventually benefit. But i think the race to the bottom scenario is unlikely if the attack necessitates a large number of preparation blocks.

That said as discussed previously i don't think your idea makes it so their blocks are at a higher risk of reorg.

[quote="sjors, post:1, topic:1288"]
If miners typically run higher end hardware for their node, does that mean slow blocks propagate just fine between miners, with the other nodes in the network being out of the loop?
[/quote]

If "miners" is a group of tightly interconnected nodes, it means expensive blocks relay faster (not "just fine" though :slight_smile:) between them. But i don't think we can assume such a network topology.

Also an attacker exploiting this would probably want to directly connect to as much of his competition as he can.

[quote="sjors, post:1, topic:1288"]
Does it create an incentive for miners who trust each other (or collude) to skip validation so they always build on the first-seen block, ignoring a fast-to-validate competitor block?
[/quote]

I think it is a likely outcome, yes. Along with (further) tightening of trust relationship between miners to edge their risk of doing this, as discussed above.

[quote="sjors, post:1, topic:1288"]
Does this accidentally introduce an incentive to produce empty blocks?
[/quote]

Empty blocks are a symptom of SPV mining due to propagation time. If the miner is stalled due to validation time he already has the full block and therefore can include transactions in his template.

In passing, the fact miners are SPV mining due to propagation time, which are tiny compared to the potential validation times we are talking about, confirms we should be concerned expensive blocks would significantly incentivize them to skip full validation / collude.


----

\[0] As long as the miner doesn't have more than 50% of the hashrate of course.

-------------------------

AntoineP | 2024-12-02 18:39:59 UTC | #12

[quote="evoskuil, post:6, topic:1288"]
This seems to assume that all blocks of the same height have the same amount of work.
[/quote]

All blocks with the same parent will always have the same amount of work.

[quote="evoskuil, post:2, topic:1288"]
Concurrent validation of competing blocks would only make sense if they had the same amount of work. I’m have no idea how common this is.
[/quote]

[Fork.observer](https://fork.observer/) says there has been 23 1-block reorgs since block height [845,625 (2024-05-29)](https://blockstream.info/block/000000000000000000034231f368448e06b644716fc11e28b4aa23005b3f0cdd).

[quote="ajtowns, post:7, topic:1288"]
At the moment, I’d expect block relay to be pretty severely impacted by slow validation.
[/quote]

This is also what i would expect and is confirmed by the couple basic experiments i ran on Warnet. Since the transaction in the block wasn't previously announced the propagation time to a node is the validation time times the number of nodes on the shorter path between it and the emitter.

-------------------------

evoskuil | 2024-12-02 19:49:02 UTC | #13

[quote="AntoineP, post:12, topic:1288"]
All blocks with the same parent will always have the same amount of work.
[/quote]

Yes, my mistake - thanks.

-------------------------

