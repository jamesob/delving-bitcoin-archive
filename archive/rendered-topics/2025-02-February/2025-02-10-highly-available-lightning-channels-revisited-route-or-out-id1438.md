# Highly Available Lightning Channels Revisited – ROUTE OR OUT

joostjager | 2025-02-10 17:01:20 UTC | #1

I wanted to revisit the idea discussed here: https://www.mail-archive.com/lightning-dev@lists.linuxfoundation.org/msg03094.html

## The core idea

The goal is to move toward a Lightning Network with zero payment failures. Currently, this is addressed using complex probabilistic scoring models — difficult to design and even harder to validate. But if the future holds a 100% success rate, probabilistic scoring becomes unnecessary. Instead, nodes could simply be skipped for an extended (nearly indefinite?) period after a single failure on a single channel. ROUTE OR OUT.

## Transitioning

To adopt this approach, nodes could signal that, for specific peers, a 100% forwarding success rate can be assumed up to a certain amount. This would allow routing node operators to gradually improve their service levels rather than making abrupt, system-wide changes. They could start small—offering guaranteed success with a single peer—and expand from there.

Senders might prioritize connections with this high availability (HA) signal and penalize aggressively if such a connection still fails. They might even be willing to pay higher fees for HA channels.

## Concerns & counterpoints

In the linked discussion, various developers raise concerns about this concept. However, the actual impact remains unknown. Moreover, nothing prevents routing nodes from modifying their software to send this signal. One could argue it’s better to formalize and standardize this signaling rather than leaving it to unofficial modifications.

In fact, developers aren’t even in control here. Routing node operators using existing implementations can already signal high availability today. The `htlc_maximum_msat` channel update parameter still has unused bits that could be repurposed for this.

For instance, in the public graph, no node currently sets `htlc_maximum_msat` to a value ending in 555. Nodes could slightly adjust their `htlc_maximum_msat` to end in 555, signaling that the channel is highly available for amounts up to that value.

Interested to hear more thoughts on the concept of high availability signaling and the proposed deployment mechanism.

-------------------------

cguida | 2025-02-10 19:04:45 UTC | #2

I like this proposal.

LN needs to be both private and fast/cheap/reliable, but it seems to me that the two aspects are mutually exclusive. However, it should be possible for the user to select whether they want payments to be fast/cheap/reliable on one hand, or private on the other hand, and different routes on the same network could serve different purposes.

-------------------------

joostjager | 2025-02-11 17:30:44 UTC | #3

Summary of discussion with a routing node operator:
* It may not be necessary to signal HA per channel. On the node level in the node announcement may also work. Note: without code changes, an alias suffix `(HA)` may be an option too?

* It may not be necessary to signal a guaranteed amount that the node should always be able to route. Can also leave it to the node operator to make sure liquidity is enough for the traffic that they see.
* Part of the strategy of a HA node could be to stop signalling depleted channels.
* Nodes apparently start to experiment with returning errors different than "temp channel failure" to try to avoid penalization. Note: they should probably just return a corrupt failure message which currently can't be attributed, to evade penalization even more.
* Node sometimes totally shut off routing for a while. Can be when chain fees are high and they don't want to risk force closes.
* HA nodes could also be selected preferentially for blip39 blinded paths, to increase the likelihood of success
* Having proper inbound liquidity could be part of the requirement to be a HA node. So this extends the responsibility of the node.
* Some nodes (especially tor) are really slow and it is currently not possible to pin point them.

-------------------------

MattCorallo | 2025-02-11 18:03:23 UTC | #4

I think this is a really bad idea.

The goal of getting payments down to reliably succeeding on the first try is, of course, critical to lightning being a successful payments platform, anything less than perfect reliability is a failure. However, the reality is pathfinding, today, is not a major barrier to that. Obviously doing pathfinding well takes a pretty nontrivial amount of work (see https://lightningdevkit.org/blog/ldk-pathfinding/) plus probing regularly, but given that work has already happened, payments *do* go through the first time (if you're willing to pay the required fee). Some of this, of course, is because of the network being fairly strongly connected, but I'm confident our pathfinding logic will scale to somewhat larger networks (again, given you're doing background probing, and maybe utilizing trampoline if we get a much larger network).

There are, of course, many cases where payments today *do* fail, but the reasons for that are rarely due to fundamental limitations in the pathfinding step - payments often fail if your node either doesn't regularly probe or doesn't fetch scoring information from a node that does (something which all nodes really should do by default!). Payments also often fail because nodes simply do not have the available liquidity to make the payment - often  the recipient doesn't have some JIT channel service (again something that all nodes targeting non-hobbyists really need to be doing by default!) or the sender may not have enough actual capacity (due to dust limits and poor feerate estimators, funds split across multiple channels pre-splicing, reserve values, etc, etc, etc...things that are all slowly being improved, especially with an upcoming channel type based on TRUC).

On the other end, this proposal has real social costs. Many senders are likely to take the "easy way out" - instead of fixing their pathfinding logic they'll just assume its some fundamental issue and only route through "HA Nodes". Average routing node operators will be caught between a rock and a hard place, then can:
 * not signal "HA", and not get any material routing volume,
 * signal "HA", and find their liquidity occasionally depleted (it happens, even on a well-balanced node sometimes you just don't have the liquidity), causing them to be marked "bad signalers" and lose their routing volume,
 * signal "HA" and try to do JIT rebalancing, having to charge a corresponding fee increase. But of course rebalancing doesn't always work - usually if you don't have enough liquidity in one "direction" someone else might not either, so you'll still fail some payments (unless you want to charge *dramatically* more than market rate for relaying), causing you again to lose your routing volume.

But big custodial operators running big nodes won't have this issue - they can agree to open 0conf channels with each other, use that for "JIT rebalancing" and signal "HA". They'll get all the naive sender routing volume and we'll end up with ~all "end user" lightning payment volume routing through big AML operators...the opposite of the goal of lightning.

In general, before we take actions that have substantial social implications for the network, we should seek to ensure that (a) they're actually necessary, (b) that we've thoroughly explored other options to fix the underlying issues. I don't think that this is currently necessary, and driving it today confuses issues that are not fundamental to lightning routing with issues that are.

-------------------------

joostjager | 2025-02-12 09:27:45 UTC | #5

I see your point. And I also see that there are uncertainties. For instance that pathfinding is not a major barrier today. It might be a barrier for certain groups of senders. Probing and/or downloading probe data isn't ideal either. I would rather minimize that if possible.

Regarding senders taking the easy way out: I'd argue that they might do so already but in a worse way. Just download a list of good nodes and prefer those. For nodes that aren't on that list for whatever reason, it may be even more difficult to get on it in comparison to fairly earning the HA badge.

I don't think that nodes appending `(HA)` to their alias should be a problem. If it is and has substantial social implications, then we'd better find that out sooner rather than later.

-------------------------

MattCorallo | 2025-02-12 15:17:46 UTC | #6

[quote="joostjager, post:5, topic:1438"]
For instance that pathfinding is not a major barrier today. It might be a barrier for certain groups of senders.
[/quote]

I agree with that, I'd just prefer to pay the social costs of doing HA-based pathfinding until we hit those barriers.

[quote="joostjager, post:5, topic:1438"]
For instance that pathfinding is not a major barrier today. It might be a barrier for certain groups of senders. Probing and/or downloading probe data isn’t ideal either. I would rather minimize that if possible.
[/quote]

Its certainly not ideal, but it seems wayyyy more ideal than the potential social implications of "HA" signaling. And, really, its not all that bad. Individuals probing don't necessarily generate all that much traffic (especially now that we don't see almost any stuck HTLCs anymore). Downloading probing results seems even more inoffensive - the prober can influence your pathfinding somewhat, potentially hurting your privacy if they are already a potential path for you, but in many cases we're fetching probe results from an LSP, who is gonna see all your payments anyway. This leaves us, basically, only with people running full lightning nodes without an LSP and without making a lot of payments or probing having reduced privacy, but these nodes are increasingly few and far between, and that's probably a good thing for total payment success anyway.

-------------------------

