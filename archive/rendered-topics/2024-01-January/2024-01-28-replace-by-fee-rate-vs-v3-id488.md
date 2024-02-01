# Replace-By-Fee-Rate vs V3

oohrah | 2024-01-28 01:33:34 UTC | #1

How do these proposals compare?

https://petertodd.org/2024/one-shot-replace-by-fee-rate
https://github.com/bitcoin/bips/pull/1541

I'm developing things on top of Lightning so I'm not a low level guy.

-------------------------

instagibbs | 2024-01-29 15:04:37 UTC | #2

At a high level, each approach is attempting to answer the question:

How do we make sure incentive compatible transactions can be mined without introducing transaction relay that isn't "paid for"? This also known as the "free relay problem".

The "free relay problem" becomes a problem if people are able to make transactions that enter into everyone's mempools, causing validation and bandwidth costs to everyone on the network. but then are ejected from the mempool later, resulting in *no fees* being paid for the privilege. 

To mitigate this problem, the current Bitcoin Core software makes attempts to minimize it:

1) Replacement transactions have to pay for what they replace("total fee"), plus "incremental fee"(today 1 sat/vbyte) to pay for the new bytes being gossiped around the network.
2) mempool minfee: When the mempool gets to full and gets "trimmed" down to ~300MB by default, the "min fee" to enter the mempool is raised to the highest paying thing that was ejected, plus the 1 sat/vbyte incremental rate. This "raises the bar" for the next things to enter, and only in the worst case allows some temporary free relay which is paid for immediately after.
3) timeout: After 2 weeks, the mempool ejects transactions as they aren't expected to be mined. This is free relay, bound by long timescales.

"V3": Relay Less
---
V3 is one attempt to get around pinning issues in an opt-in manner. If a transaction can commit to opting into a new policy, we can be more restrictive in a way that perhaps wouldn't be generally acceptable to users.

In essence, it restricts topology to avoid free relay, while mitigating pinning by restricting total size of portions of transactions. It's a mitigation, not a complete fix, but also doesn't allow new free relay avenues. [Future mempool updates](https://delvingbitcoin.org/t/an-overview-of-the-cluster-mempool-proposal/393) could likely allow this policy to evolve into something more incentive compatible and more pin resistant.

"Replace-by-Fee-Rate": Relay More
---
In contrast, some have suggested that the free relay problem isn't much of a problem at all, or that it's an unsolved problem, and that since it cannot be stopped, we should just not worry about offering services that would use it.

This is what Peter's argument essentially boils down to: He claims miners can already induce some free relay by leaving fees on the table with transaction filters, so we should just bite the bullet and allow more, and let anyone do it.

I'll let people dive into the precise arguments themselves, but it's important to note a few things:
1) If free relay is a solvable problem(through say, smarter relay logic), should we really be offering a replacement policy we may revoke later?
2) Is it really qualitatively the same if miners leaving fees on the table intentionally can do something, versus anyone on the network?
3) It's also important to note that pre-cluster mempool, reasoning about any of this is very hard to do. Peter's first iteration of the idea was [broken](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2024-January/022316.html), allowing unlimited free relay. He claims he's fixed it by hot-patching the idea with additional RBF restrictions, but like usual, reasoning about current RBF rules is very difficult, and maybe impossible. I think energy would be better focused on getting RBF incentives right, before giving up the idea of free relay protection entirely.

-------------------------

glozow | 2024-01-30 10:35:57 UTC | #3

First of all, there is an "alternatives" section of the v3 BIP. It describes common suggestions and why they do not work, are good ideas but can only be applied to limited topologies (which is what v3 is for), are only feasible to implement generally after cluster mempool (which is a step after v3), or are not conflicting with this proposal.

I'd also encourage people to read the discussions linked in the BIP and PR. There is a lot of context and history, but there's tons of publicly-viewable text to build background on it.

### General Replace by Feerate

"Get rid of Rule 3+4 and use feerate instead" is a good place to start brainstorming, but it's free relay / DoSy. Not much new to say here.

### Replace by Feerate and having 2 sets of RBF rules

There has also been lots of exploration of the idea "in certain special situations, use a different set of rules based on {feerate, miner score}." One-Shot Replace by Feerate is one. There's also ["[PROPOSAL] Emergency RBF (BIP 125)"](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2019-June/016998.html ) and ["Fees in Next Block and Feerate for the Rest of the Mempool"](https://gist.github.com/glozow/25d9662c52453bd08b4b4b1d3783b9ff#fees-in-next-block-and-feerate-for-the-rest-of-the-mempool), among others.

For me, a key takeaway from those discussions was:
- A replacement should confirm "faster" than the replacee. Users should not be paying higher fees and feerate only to have their transaction confirm slower.
- We should have a "miner score" or some miner incentive compatibility metric that we can use to compare the replacement and replacee (obviously individual feerate does not work). And we should require that replacements increase miner score.

Some major problems with this include:
- Free relay is usually still present. See [this](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2024-January/022302.html) or [this](https://gist.github.com/glozow/797bb412868ce959dcd0a2981322fd2a#free-relay-problem) on infinite replacements. Intuitively, having 2 sets of rules designed to bypass each other can easily result in this kind of loop. I'm not saying it's impossible to design 2 that don't do this, but just some intuition if you don't want to go through the math.
- The notion of "would confirm soon" or "in the top N portion of the mempool" such that it's *safe* and *useful* to employ this other set of rules is not well-defined. It's also not at all easy to implement (see next point).
- The mempool as it exists to today doesn't support an efficient way to calculate "miner score" or incentive compatibility, due to unbounded cluster sizes.
  - There is no static calculation using the cached ancestor set values. Lots of people propose ancestor feerate (including myself, see [this PR](https://github.com/bitcoin/bitcoin/pull/23121) which I do think is deserving of the "half baked" criticism), but it doesn't work. Also see [this more deatiled breakdown of 4 options for static calculation](https://gist.github.com/glozow/25d9662c52453bd08b4b4b1d3783b9ff#mining-score-of-a-mempool-transaction).
  - Actually calculating the miner score (see [this implementation](https://github.com/bitcoin/bitcoin/pull/27021) for privileged wallet which needs to halt at 500) is too computationally complex to include as a step in mempool validation.
  - I've [proposed](https://gist.github.com/glozow/25d9662c52453bd08b4b4b1d3783b9ff#mempool-changes-need-for-implementation) meeting halfway and caching the top block's worth. That is also [considered](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-February/019879.html) too complex.

One advantage of cluster mempool is being able to calculate things like miner score and incentive compatibility across the mempool. Similarly, one advantage of v3 is being able to do this before cluster mempool because of restricted topology. Before people took on the challenge of designing and implementing cluster mempool, I had been [framing](https://bitcoincore.reviews/25038) v3 as "cluster limits" without having to implement cluster limits, as it's one of the only ways to codify a cluster limit (count=2) using existing package limits (anc=2, desc=2. Once you go up to 3, you can have infinite clusters again). Another advantage of v3 is that it helps unblock cluster mempool, which is imo a no-brainer.

In summary, I don't think the One-Shot Replace by Feerate proposal works (i.e. doesn't have a free relay problem and is feasible to implement accurately). The path of upgrades proposed (v3 package RBF, v3 sibling eviction, 1p1c package relay, cluster mempool) is the most complete solution by far. V3 is simple (and perhaps that leads people to think it is not well thought out?) and useful both as a building block and for solving specific problems in general.

-------------------------

oohrah | 2024-02-01 05:08:19 UTC | #4

[quote="glozow, post:3, topic:488"]
Free relay is usually still present. See [this](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2024-January/022302.html) or [this](https://gist.github.com/glozow/797bb412868ce959dcd0a2981322fd2a#free-relay-problem) on infinite replacements. Intuitively, having 2 sets of rules designed to bypass each other can easily result in this kind of loop. I’m not saying it’s impossible to design 2 that don’t do this, but just some intuition if you don’t want to go through the math.
[/quote]

What do you think of Peter's fix that he published to bitcoin-dev?

https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2024-January/022326.html

I don't do deep protocol design so maybe I'm missing something. But the fee-rate/depth argument looks pretty solid to me and I guess it must be actually deployed in Libre Relay nodes.

In [your](https://gist.github.com/glozow/797bb412868ce959dcd0a2981322fd2a#free-relay-problem) writeup, I don't understand how step #3 is following one-shot rbfr rules. Isn't that just a normal replacement with a tx of a higher fee? 16,500 > 5000

Not sure I understand your argument there.

-------------------------

