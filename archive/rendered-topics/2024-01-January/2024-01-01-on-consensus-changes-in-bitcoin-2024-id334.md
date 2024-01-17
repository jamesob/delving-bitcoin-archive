# On consensus changes in bitcoin 2024

reardencode | 2024-01-03 08:52:02 UTC | #1

There has been much conversation on X and elsewhere recently of how, if, and when bitcoin consensus rules should be changed. I will try to briefly summarize some of the viewpoints, open the question of how consensus changes to bitcoin should be selected, and then provide my own thoughts on that question.

## Ways to change consensus

### Ossification

On the one hand, this perspective cannot be taken seriously, but it currently seems to hold sway with many in the bitcoin ecosystem.

This is the view that bitcoin consensus rules are fine as they are. It's perfect money already and does not require any improvements. There seem to be a few underlying reasons that some folks hold this view:

* Inscription/ordinal derangement: Some folks hold the opinion that previous changes to bitcoin enabled inscriptions/ordinals and that changes are therefore bad.
* General lack of understanding: For the non-technical, bitcoin seems to work by magic. Any change to a magical system could upset the delicate balance of magical properties that enable it.

### Trusted Leaders

There seem to be a group of bitcoiners who will only consider a change to the bitcoin protocol if they are supported or proposed by the correct people. This view is essentially similar to the magical ossification view. Bitcoin is a delicately balanced magical system and only the anointed saints can make the correct decisions on its future.

### Rough Consensus

This is the view that changes to bitcoin consensus should *roughly* follow the IETF rough consensus model. Here I'll quote a few sentences from the [IETF doc](https://datatracker.ietf.org/doc/html/rfc7282) on the topic:

> Requiring full consensus allows a single intransigent
   person who simply keeps saying "No!" to stop the process cold.

> Consensus doesn't require that
   everyone is happy and agrees that the chosen solution is the best
   one.

> all that they've done is capitulated; they've simply given up
   by trying to appease the others.  That's not coming to consensus [...]
> Even worse is the "horse-trading" sort of compromise

> Rough consensus is achieved when all issues are addressed, but not
    necessarily accommodated

> While counting
   heads might give a good guess as to what the rough consensus will be,
   doing so can allow important minority views to get lost in the noise.

#### How would rough consensus even work in bitcoin?

In the rough consensus model there is always a chair for any specific change discussion, and it is this person's responsibility to determine when rough consensus has been reached. We don't have chairs (because we sold them all for sats), so we lack a critical element of the rough consensus process.

## How _should_ consensus changes be chosen?

Are there other ideas for how consensus changes should be chosen? Can we follow one of the above models? This seems to be the most pressing question in bitcoin development as we roll into 2024. Historically, bitcoin consensus changes have followed some mix of *Trusted Leaders* and *Rough Consensus*. Our trusted leaders have wisely chosen not to lead consensus changes any further, and it seems like rough consensus got a bad name when certain people tried to use it as a bludgeon during the blocksize war.

## Where to?

First, I think we must accept that right now we do not have the ability to reach consensus. We lack Trusted Leaders and we lack Chairs. Without these we are afloat in our various pods of bias, preconception, personal history, etc. Not only can we not reach consensus, we don't even have the ability to bring the various factions of bitcoin developer mindshare together to find out if there are technical objections to any particular change. Consensus changing code is rarely proposed, rarely reviewed, etc.

Some have claimed that publishing or promoting a signaling / activation client for a consensus change is an attack on bitcoin, but I would counter that doing so may be the only way to discover consensus. An activation client makes the economic nodes the chairs of the rough consensus process. Each economic node, being their own chair, can evaluate whether consensus has been reached for themselves when deciding what software to run.

I'd suggest that for bitcoin to continue along the massively successful path that it has walked to date we do need to discover how to continue developing consensus changes, and that this is an important focus in the year ahead.

My proposal is that the bitcoin core project themselves begin publishing a client which supports validation of a variety of changes with separate signaling and let the signals fall where they may. Exact details of these activation clients are not my area of expertise, but I suspect that for this to be successful configuration options for signaling bits and lock-in on time would be necessary.

-------------------------

ursuscamp | 2024-01-01 22:15:43 UTC | #2

This is an important discussion. Thanks for opening it up. There is a severe lack of clarity these days. It is well that Bitcoin doesn’t have a Leader, but we could definitely use a Process. Arguing forever on Twitter is not a great way to do anything. 

Since we seem to be in a new era, I would like the Core maintainers to weigh in on what their criteria is for inclusion of consensus changes in Core. I have heard second hand rumors that Jeremy Rubin was told that they want new consensus features activated outside of Core first. Is this true? Have there been any public statements about this?

-------------------------

ajtowns | 2024-01-03 08:54:22 UTC | #3

Moved to the "Philosophy" category and removed the "meta" tag -- this post is exactly on-topic as a philosophical discussion about bitcoin development processes in my opinion.

-------------------------

urza | 2024-01-03 09:18:27 UTC | #4

Rusty @rustyrussell  had a blog post on related topic about " Soft Fork Activation" few years back:

http://rusty.ozlabs.org/2021/02/26/a-model-for-bitcoin-soft-fork-activation.html

While you discuss more searching consensus, his blog is more about activation.

But I quite like the main idea:

"Devs propose. Miners activate. Users override."

What do you think?

-------------------------

roasbeef | 2024-01-04 00:21:38 UTC | #5

[quote="urza, post:4, topic:334"]
“Devs propose. Miners activate. Users override.”
[/quote]

That's personally been my favorite conceptualization so far. 

I think where things are getting caught up is: devs propose + miners activate.

These days there isn't a sort of agreed upon process or mechanisms for the actual proposal portion. The `bitcoind` organization (imo) doesn't have the same level of agency in this area (softfork consensus changes, lots of other stuff is moving along nicely) as it once did (there is no longer a unified voice for the org). In the face of that structural change, it isn't clear what "proposal" actually looks like, nor what it actually takes for something to be "ready". It's all an emergent process at the end of the day, mechanisms that worked in the past aren't guaranteed to work again, we're all still tryna figure this thing out. 

Maybe we end up tending to a future where `bitcoind` never merges any soft-fork activation + validation logic until it's well activated. At face value, one could perceive that as being more chaotic, as most ppl's definition of "rough consensus" is that something is merged into `bitcoind` with devs proposing as a unified voice. Without that unified voice, there's more uncertainty in the marketplace for the set of relevant economic actors. Ultimately, BIP 9/8 still allow for multiple soft-forks to be proposed in parallel, but the socio-economic system we have today hasn't really embraced that for various reasons (coordination costs, review, etc). 

Maybe the in the future alternative implementations will actually be properly viable, which would give us another platform for proposals, but we aren't quite there yet today. Other implementations are still years behind, and don't even have 1% of the funding or dev power that `bitcoind` has, so they're far too risky from the PoV of the most conservative market participants. 

On the "miners activate" front, depending on who you talk to, that statement is tantamount to blasphemy. Not everyone took home the same lessons after the segwit+block size drama. If you sample what seems to be the popular Twitter anon/pleb/taking-head sentiment, people actually seems to want hard forks in spirit, as they reject the notion of optional softforks that don't require "everyone" to update. 

From my PoV [soft-forks we originally engineered](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2015-December/012014.html) to get around the impossibility of global coordination, as they enable optional upgrades, based on an agreement between developers and miners. However maybe the soft-fork window has now closed, somewhat more prematurely that most would have envisionsed.

-------------------------

ProofOfKeags | 2024-01-04 18:38:49 UTC | #6

### On Activation
I think the reason that it's considered blasphemy for miners to activate is a misunderstanding of the asymmetry of soft vs. hard forks. If we take all three components of Rusty's maxim, the third (users override) allows users to intervene, should any miner impropriety take place.

The issue with the block size wars is two fold. First was that the 2MBHF is a hard-fork, forcing old nodes out of consensus, which is an aggressive maneuver. Second, the change being proposed imposed significant (+100%) ongoing resource costs with cumulative storage requirements.

Miners are and always have been responsible for administrating consensus updates. Whether we like it or not, miners are already capable of executing a "progressive soft-fork" by just censoring transactions that violate the new soft-fork rules.

For this reason I believe the position of universally rejecting MASF's that some people seem to be taking is willfully ignorant of the capabilities and risks Bitcoin actually has in favor of amplifying the drama of the block size debate. I believe this is a mind virus that Bitcoin MUST overcome if it's going to succeed.

Miners imposing their own interests on the community is a real risk and in certain circumstances we should be able to coordinate action in rejecting malicious changes, but allowing them to fulfill the responsibility of smooth consensus upgrades via BIP8/9 is quite reasonable.

The main argument I hear against BIP9 (LOT=F) is that it "gives miners the ability to override the will of users". This is not what is going on. What it does is tells miners that we have decent-ish enough consensus that we want this thing to be activated but not enough to hold you by the nuts to do it. If they reject it under this arrangement then there's nothing to stop the community from actually mobilizing a LOT=T deployment and trying again. In fact, I'm nigh certain that this would happen as it would trigger a lot of people into getting out their LOT=T pitchforks.

### On Readiness
However, all of this is a bit premature as it seems that the current debate is not HOW to deploy a soft-fork, like was the case with taproot, but what even constitutes consensus when it comes to deciding "yes we should, in principle, activate this thing".

There are a variety of types of soft-forks and this is where the discussion gets nuanced enough that I fear it is increasingly going to be inaccessible to the average node-runner. Some soft-forks are purely non-invasive. They do not impact existing users' use of Bitcoin in any way other than subtly changing the global economy of Bitcoin by granting users opt-in capabilities. If one takes the position that any change that subtly impacts Bitcoin's global properties needs to benefit all users of Bitcoin, then you can generalize that to changes in relay policy, custodial wallets, new financial products etc. This position is not tenable. Thinking through this idea invariably leads to ossification and ossification will invariably lead to a flippening as at some point the properties of coins that are allowed to change will represent a greater value proposition than Bitcoin's maximal commitment to stability via ossification.

There are other soft-forks that seriously impact the whole Bitcoin economy (reducing the block reward) which meaningfully change the Bitcoin social contract for everyone. These types of soft-forks would be advisable for users to reject, should miners try to implement them sneakily.

As such, imo, there is only one real valid reason to reject a soft fork and that is saying that there are existing users whose existing coins would behave differently than they planned for when they created the utxo.

From this point the main thing that has to be figured out is how to tell whose arguments are correct when it comes to describing how the proposal impacts those users. Again, proving the utility of a change is not necessary for this step, proving safety of the existing usage of the system is. This process I believe to be fairly easy to do rigorously, as you'd be able to demonstrate by contradiction why the new change burns someones coins or changes the semantics in a way that leaves them vulnerable to theft.

As far as utility goes, making utility arguments is a good way to get people interested and interest is how we get review. If things are sufficiently reviewed and deemed to be safe then the review interest alone is enough to demonstrate the utility. Obviously not all reviews/reviewers carry equal sway and that will continue to be true in perpetuity. Regardless, more review implies more interest.

### On Process
I want to offer one final idea with regards to the process of changing consensus. It is perhaps a good idea to work backwards from the activation rather than forwards from the proposal.

I think there is a pretty strong consensus that we should do *some* kind of soft-fork in the future. I think also that there is strong consensus that we probably want it sooner than 4 years after the last one. I think even further that there is pretty overwhelming technical consensus that *some* form of covenants are desirable. Where there seems to be substantial disagreement is precisely which covenant proposal is best or whether we just want to wait an indeterminate period of time for yet another covenant proposal to emerge.

I think it's possible to improve both the speed of development as well as promote cooperation among developers and reviewers by saying "We want a 2025 soft-fork, it will happen, what's in that soft-fork is TBD but whatever is going needs to have sufficient review and we roughly agree on the north star of what we want to accomplish with it."

Make no mistake, this is a brinksmanship tactic. But if we can collectively agree to engage in that brinksmanship then maybe it may actually force us to get out of the stands and into the arena to make sure that we can come to an agreement on something that will actually improve Bitcoin, rather than slowly letting it atrophy and brain drain as we lose more brilliant devs to technology projects (cryptocurrency or otherwise) that actually have a possibility of impact.

Bitcoin's success is not pre-ordained. We need to figure this out.

-------------------------

harding | 2024-01-11 02:28:17 UTC | #7

> I think we must accept that right now we do not have the ability to reach consensus.

I strongly disagree with this viewpoint.  Technical consensus has been found on many subjects since taproot's activation.  Examples: for many years, one developer vociferously opposed encrypted peer connections---now we have those in Bitcoin Core; other developers opposed even adding full RBF as an option---we have that too in Bitcoin Core; devs for a major LN implementation didn't like onion messages and certain things built on them, yet support for onion messages and blinded paths are being added there as I write; miniscript seemed for a while like some weird Bitcoin Core science project, yet now hardware wallets are adding support; years of debate about how to solve channel jamming attacks are finally seeing experimentation with solutions; and (with a few current dissenters) there's widespread agreement after 18+ months of work that v3 relay, ephemeral anchors, and cluster mempool are solid building blocks for future upgrades.  Heck, even with millions of dollars seeming to ride on continued controversy, there's rough consensus among developers to avoid witness filtering.

Technical consensus has also been found on many bad ideas, or ideas that were not compelling.  Mainly that happens by an idea simply not gaining traction.  Examples from Optech newsletters since taproot activation: two-digest BOLT11 invoices (#256), relay of annexes before a defined need (#255), LN QoS bit (#239), global LN reputation tokens (#228), (at least some versions of) LN capacity-dependent feerates (#219), perpetual subsidy (#209), and fee accounts (#182).

If we don't have technical consensus for what soft fork to perform yet, then I think the reason must be something besides our ability to form consensus.  One reason I would suggest for consideration is that none of the proposals so far has been compelling.  If a better proposal comes along, or if some of the existing proposals are made more compelling, then it's possible that will attract reviewers and lead to eventual technical consensus.

> My proposal is that the bitcoin core project themselves begin publishing a client which supports validation of a variety of changes with separate signaling and let the signals fall where they may. 

Please, no.  A soft fork that not everyone agrees is active can create a chain split.  If a chain split is eventually healed, some miners are guaranteed to lose money.  If it's not healed, Bitcoin's economy fractures, harming users.  In either case, many poorly informed users, or users who wrongly predicted the outcome of the chain split, are likely to lose money.  Those who take precautions not to lose money will likely lose the ability to transact for part or all of the chain split.  It's an all-around bad mechanism for making upgrades to the system.

We can do better.  Or, if we can't do better and putting Bitcoin at significant risk of a chain split is the only way to change the consensus rules, then sign me up for team Immediate Ossification.

-------------------------

reardencode | 2024-01-11 03:07:06 UTC | #8

[quote="harding, post:7, topic:334"]
I strongly disagree with this viewpoint. Technical consensus has been found on many subjects since taproot’s activation.
[/quote]

Reaching consensus on changes to bits of software around bitcoin, even bitcoin core, is in almost no way similar to reaching consensus on changes to the consensus rules. Say consensus one more time, Rearden.

[quote="harding, post:7, topic:334"]
Please, no. A soft fork that not everyone agrees is active can create a chain split. If a chain split is eventually healed, some miners are guaranteed to lose money.
[/quote]

This does not concern me in the slightest. If it does concern you then you can lobby for an activation client that uses LOT=false, which has approximately no risk of a chain split.

I have almost no opinion on the details of activation, but I believe it fairly likely that the only way for bitcoin to reach consensus on consensus rule changes is to publish activation clients and let users lobby miners on which bits they want set.

-------------------------

harding | 2024-01-11 03:04:16 UTC | #9

> Reaching consensus on changes to bits of software around bitcoin, even bitcoin core, is in almost no way similar to reaching consensus on changes to the consensus rules.

I disagree.  The people and the process are largely the same.  The only significant difference is that the stakes are higher.

> an activation client that uses LOT=false, which has approximately no risk of a chain split.

This is wildly wrong.  Miners have false signaled in the past leading to chain splits, e.g. the 4 July 2015 chain splits: https://en.bitcoin.it/wiki/July_2015_chain_forks 

That was an accident, but in a situation where many people aren't running a particular client, miners could false signal to steal money from people.

> I believe it fairly likely that the only way for bitcoin tor each consensus on consensus rule changes is be to publish activation clients and let users lobby miners on which bits they want set.

That's insane.  If miners signaled that they were only going to include OFAC-compliant transactions, would you have us all accept that as a new part of Bitcoin's consensus rules?

-------------------------

moonsettler | 2024-01-11 09:46:02 UTC | #10

[quote="reardencode, post:8, topic:334"]
If it does concern you then you can lobby for an activation client that uses LOT=false
[/quote]

i second this. just leave the BIP out of it! stand on a soap box and preach about the evils of LOT=true! that should go well.

miners dicking around with false signaling and ending up losing money on that, is acceptable.

-------------------------

MattCorallo | 2024-01-16 23:05:10 UTC | #11

I think what I wrote in 2020 is just as true today as it was then: https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-January/017547.html. There's a set of goals we must define, and I think the 5 listed there basically summarize what the goals for the Bitcoin system should be (it doesn't mention how this implicates Bitcoin Core at all). One of the key themes here is that we should avoid, in all the ways we can, chain splits and losing hashpower onto one fork or another. Its impossible to avoid in a general sense, but all the technical design decisions that go into the fork should attempt to ensure we do not lose hash power or users onto a fork where a decision can be made to do so.

This also implies that ideas like "just ship code that lets users make uninformed decisions resulting in a fork" are right out. This also implies we must rely on users having upgraded by the time any activation happens, and have very high confidence that a vast majority of users will be using the new consensus logic by the time any activation happens. Miner signaling is useful, but as @harding points out, it is very, very far from sufficient.

There's a ton of ideas on philosophy out there on what roles different parties can or should take, but we shouldn't lose sight of the fact that this is a production network protecting billions of dollars of other peoples' money in a naive attempt to achieve some philosophically perfect outcome.

-------------------------

reardencode | 2024-01-16 23:10:55 UTC | #12

Thanks for your thoughts Matt! I think I had read that post back in 2020, but forgotten it. I agree entirely with your email.

I'm not entirely sure how this applies to not publishing LOT=FALSE clients and seeing what shakes out? With LOT=FALSE, 95% threshold, either it activates safely with minimal loss of users or hash, or it doesn't. Seems completely consistent with what you wrote.

-------------------------

MattCorallo | 2024-01-17 01:30:45 UTC | #13

Not quite - "seeing what shakes out" implies that we're not confident that users will be running the modified consensus rules by the time any possible activation happens. In such a case, we're putting the network at risk of splitting if miners decide to signal while no such large-scale upgrade has completed.

-------------------------

