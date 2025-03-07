# Antoine Poinsot on Bitcoin Core's Priorities

ajtowns | 2025-02-21 16:47:25 UTC | #1

@AntoineP wrote three interesting blog posts on Bitcoin Core:

 * https://antoinep.com/posts/core_project_direction/
   * *"I think we are in this situation where everybody is working on their own thing and there is no broad agreement about what the scope of the project should even be. As a consequence, year after year we keep piling up more code, more features, more RPCs, etc.. with at best a constant amount of competent reviewers’ time. This is not tenable, as a focus which gets further dispersed will inevitably lead to reduced overall software quality."*
 * https://antoinep.com/posts/stating_the_obvious/
   * *"When you consider that Bitcoin Core’s userbase is indirectly all Bitcoin users, how to prioritise development becomes quite clear. Bitcoin Core should be a robust backbone for the Bitcoin network, balancing between securing the Bitcoin Core software and implementing new features to strengthen and improve the Bitcoin network."*
 * https://antoinep.com/posts/bitcoin_core_scope/
   * *"The multiprocess project presents an opportunity in this regard. Separate `bitcoin-node`, `bitcoin-wallet` and `bitcoin-gui` binaries communicating through well-defined interfaces open the door to splitting the current project into 3. The contributors at [github.com/bitcoin/bitcoin](https://github.com/bitcoin/bitcoin) would focus on the Bitcoin Core node and release a `bitcoin-node` program, exposing JSONRPC and IPC interfaces. The contributors at [github.com/bitcoin-core/wallet](https://antoinep.com/posts/bitcoin_core_scope/github.com/bitcoin-core/wallet) would work on the `bitcoin-wallet` process. The Bitcoin Core wallet releases would contain both the wallet and node processes, presumably with a wrapper to make it transparent to users. This project would release the equivalent of today’s `bitcoind`. Finally, the contributors at [github.com/bitcoin-core/gui](https://github.com/bitcoin-core/gui) would develop and release the `bitcoin-gui` program, which would ship with the node and wallet."*

-------------------------

AntoineP | 2025-02-21 18:10:50 UTC | #2

Next thing to figure out would be how to handle the shared code and configuration. What i had in mind was that the wallet would have the node as a subtree, likewise the GUI with the wallet. There is some common code that could be mostly split (`src/crypto`) but other (`src/util`, `src/common`, ..) will have to either be duplicate or become another interface between the projects.

Copying the configuration (the CI for instance) seems less concerning. Then the other big thing to figure out (which is a blind spot for me) is how we would organize the build system. cc @fanquake maybe you can clue me in.

-------------------------

ajtowns | 2025-02-21 20:00:22 UTC | #3

I mostly made this topic so I could reply here rather than writing up a blog entry of my own. :) So...

A few thoughts:

 * I'm not convinced multiprocess is the right way to split things out of core in any meaningful way -- as far as I understand it, multiprocess is intended to leave its affected components [fairly tightly coupled](https://github.com/ryanofsky/bitcoin/blob/pr/ipc/doc/design/multiprocess.md#interface-stability), so changes to core or the gui or the wallet would likely affect the multiprocess api, and be annoying to coordinate/sync across repos. I think multiprocess/capnproto makes more sense for process separation for security reasons, than for development separation for prioritisation reasons, but could be convinced otherwise.
 * Having all our wallet features be available via a library (or command-line tool, like bitcoin-tx or bitcoin-wallet) that you can use without having to fire up a bitcoind to get at the RPC interface would be a win, I think, independent of anything else.
 * One approach I like is [ketan's NodeBox](https://shop.ministryofnodes.com/shop/bitcoin-nodebox/) which splits the bitcoin software into two parts: a node, index and blockchain explorer that don't retain any personally identifiable information; and the wallet software that does. That puts core, electrs, and mempool.space as the "node", and sparrow installed on a different device as the wallet, which is a somewhat similar split up to what Antoine considers above. One big difference is the inclusion of electrs for indexing transactions, which is useful both for the mempool.space explorer, and to allow a wallet that's offline most of the time to quickly sync. BDK also [seems to assume](https://bitcoindevkit.github.io/book-of-bdk/cookbook/syncing/electrum/) having access to indexes like that is a good idea. That's something core has traditionally resisted (last attempt was [PR#14053](https://github.com/bitcoin/bitcoin/pull/14053) I think), and I think that resistance perhaps makes it a bit harder to do a nice UI, especially if you want to separate your PII away from your node software, but still want very quick syncs when you open your wallet.
 * Personally, I like the idea that bitcoin core aims to provide enough software to make Bitcoin usable; so a GUI and a working wallet are an important part of that. How that's architected isn't so important to me, and of course, bitcoin core probably doesn't quite achieve that today anyway, in that these days it's missing necessary components on the mining side of things. Having wallet functionality also seems useful to have in doing functional tests, and experimenting with bitcoin more broadly (eg, messing around on signet).

-------------------------

ariard | 2025-02-22 00:14:24 UTC | #4

"*When you consider that Bitcoin Core’s userbase is indirectly all Bitcoin users, how to prioritise development becomes quite clear.* “ Excerpt - https://antoinep.com/posts/stating_the_obvious/

Who is a (average) Bitcoin Core’s user ? This is a well-informed series of posts, though it’s very centered on contributors, maintainers, dev process, etc without really asking for the obvious. 

What is a (average) bitcoin core user ? What level of know-how they should have to run `bitcoind` ? Are they more often on Windaube or a *NIX ? Do they care about running `bitcoind` to verify their stack the cypherpunk way or just because to contemplate the well-architected `bitcoind` burning CPU time ? Do they prefer a browser-like exp or are they die-hard CLI galts and guys ? What’s their level of engagement w.r.t Bitcoin, are they going to their local meetups to hear the latest about consensus changes or do they just care fiat-vs-btc NGU price ? How often are they using their `bitcoin-wallet`, a week, a month, a year, etc ?

(— yes it’s asked in the maieutic way, no right or wrong answers).

-------------------------

bruno | 2025-02-25 13:20:23 UTC | #6

> For instance [`rust-bitcoin`](https://github.com/rust-bitcoin) has never made so much progress, and [BDK](https://github.com/bitcoindevkit/bdk) is a major success [adopted by dozens of projects](https://bitcoindevkit.org/adoption/all) with probably millions of users altogether. They have a larger feature set, with the Bitcoin Core wallet still struggling to catch up even years after a feature has been largely adopted by the industry[1](https://antoinep.com/posts/bitcoin_core_scope/#fn:1).

This is interesting. What makes BDK evolves so fast compared to Bitcoin Core? The language (I see more people interested in rust than C++ and new comers prefer to contribute to rust projects. Also, there are real projects really using it)? The way they manage the project (I kinda like the way the organize the things, the discord channel, meetings, etc)?

-------------------------

AntoineP | 2025-02-25 14:40:45 UTC | #7

[quote="bruno, post:6, topic:1470"]
What makes BDK evolves so fast compared to Bitcoin Core?
[/quote]

I think language indeed plays a role. I'm less sure about project organisation (Discord, meetings, ..). It's also the case that it's more fun to "rewrite X from scratch" than work on a project with a bunch of legacy code. Having a clear set of goals for what they are trying to achieve surely helps maintain focus, too. It also solves the coordination problem to be able to actually make progress (similar to the working groups idea for Core, "if i invest 1 month implementing and polishing this, has it decent chances of making it in?") which certainly makes it more fun to work on the project. Finally, i think the bar is generally higher to get changes in the Bitcoin Core wallet than in BDK. Being more fun with a general direction and ability to make progress in turn helps bring and retain contributors.

Splitting the wallet into its own repo won't automagically bring all of these benefits but i do think there is an opportunity to get some of them. For instance lowering the bar for review temporarily would adjust to the current reality of the number of contributors there and make it easier to make progress. Having it in its own project would let contributors focus exclusively on the wallet, and coupled with [other factors](https://lclhost.org) could let contributors coordinate on a clear set of goals for the project to advance toward.

-------------------------

moonsettler | 2025-02-25 15:48:22 UTC | #8

[quote="bruno, post:6, topic:1470"]
What makes BDK evolves so fast compared to Bitcoin Core?
[/quote]

The bitcoin core codebase is notoriously horrible to interact with. It's not just the language, but also how you use it.

-------------------------

AntoineP | 2025-02-25 19:19:21 UTC | #9

[quote="ajtowns, post:3, topic:1470"]
I’m not convinced multiprocess is the right way
[/quote]

As i shared in my comment above I don't think a split around multiprocess binaries is ideal. But i believe it is important to refocus the main project on the node parts of the software, and i just see such a split as the only realistic and socially acceptable way of achieving that. Do you have a better suggestion?

[quote="ajtowns, post:3, topic:1470"]
Having all our wallet features be available via a library (or command-line tool, like bitcoin-tx or bitcoin-wallet)
[/quote]

I agree that raw transaction parsing, PSBT updating, script decoding, signing, etc.. should not be part of the node RPC since it does not required a node in the first place. But yes it's tangential.

[quote="ajtowns, post:3, topic:1470"]
BDK also [seems to assume](https://bitcoindevkit.github.io/book-of-bdk/cookbook/syncing/electrum/) having access to indexes like that is a good idea. That’s something core has traditionally resisted
[/quote]

BDK does not assume that anymore. It now just indexes wallet transactions and has a wallet-spk-to-derivation-index index just like the Core wallet. It has a [bitcoind RPC chain data source](https://github.com/bitcoindevkit/bdk/tree/master/crates/bitcoind_rpc) as well.

[quote="ajtowns, post:3, topic:1470"]
Personally, I like the idea that bitcoin core aims to provide enough software to make Bitcoin usable
[/quote]

Me too. Unfortunately with the current resources it comes at non-trivial cost for all Bitcoin users, who are indirectly using Core, for the benefit of a handful of direct Core GUI users.

[quote="ajtowns, post:3, topic:1470"]
Having wallet functionality also seems useful to have in doing functional tests
[/quote]

We have `MiniWallet` for this?

[quote="ajtowns, post:3, topic:1470"]
and experimenting with bitcoin more broadly (eg, messing around on signet).
[/quote]

In a split future this could just be done using the Core wallet project which would release the equivalent of today's `bitcoind`? Or just any other wallet that would bundle a `bitcoin-node` (Sparrow, Liana, ..).

-------------------------

harding | 2025-02-26 08:41:17 UTC | #10

Thanks @AntoineP for thinking deeply about this subject and @ajtowns for relaying.  I'm not sure Antoine's "obvious" is obvious to me:

> Bitcoin Core’s userbase is indirectly all Bitcoin users, [so] how to prioritise development becomes quite clear. Bitcoin Core should be a robust backbone for the Bitcoin network

I think it's weird and non-obvious to define your users as including people who don't directly use your software.  My ISP presumably uses a bunch of software to manage the network that connects my home to the Internet.  Providing connectivity to me, and thousands of customers like me, is the ultimate purpose of that software---but its user is the ISP.  That distinction is important because when that software forces a choice between tradeoffs onto my ISP, they _as the user_ will make the choice that's best for them and not necessarily for me.

Similarly, if an industrial full node operator who serves data to customers is forced to choose, they will make the choice that's best for them and not necessarily for their customers.

The distinction between user and customer would be petty pedantry except for one thing: even a short lapse in enforcement of Bitcoin's consensus rules can practically change them forever.  Imagine if the operators of the predominant share of economic full nodes decided to allow a transaction spending an extra 1M BTC into the chain, and then quickly fanned out that money by opening new LN channels and forwarding the funds to channels they established before the fraud block (or using same-chain or cross-chain coinswaps, or buying non-revocable goods and services, etc...).  After 3 to 6 confirmations (30 to 60 minutes on average), performing a reorg to put those fake Bitcoins back into Pandora's box would mainly destroy the wealth of honest users while leaving the attackers enriched.  In that case, I think we as community might choose instead to accept a new limit of 22M BTC.

A hour isn't enough time for Bitcoiners to respond to such an attack, so the only practical mitigation is prevention: we need large numbers of Bitcoiners to run economic full nodes.  That is, they need to be direct users of Bitcoin Core **and** have their wallet connected to their node.  That will ensure that they will reject any attempt to open a channel with them using fake bitcoins (and the other attacks mentioned), directly protecting their wealth and indirectly protecting the wealth of anyone who doesn't run a full node by making the attack less viable.

In that sense, building a secure node isn't just about eliminating bugs and vetting protocols but also about ensuring large numbers of people use their own copy of the node to verify their own incoming transactions.  That requires a wallet and the means for non-experts to use it (a GUI).  Certainly, those don't need to be bundled with Bitcoin Core, but if the developers of the Bitcoin Core project cease to put effort into making a node-backed wallet accessible, why would we expect anyone else outside the project to go through the frustrating process of trying to get PRs merged into Bitcoin Core that will provide them with the interfaces they need?  Anyone who has tried to do that in the past has received a litany of "noes": no, we're not going to add another index; no, we're not going to extend the P2P interface to provide unverifiable info; no, we're not going to keep SSL to allow remote RPC access; etc...

Will those noes suddenly become yeses after a multiprocess-based repo split?  Or will there be even more reason to say "no" to wallet-supporting features in the node because all of the focus is on optimal verification and relay?  The idea of a node being embedded in wallet software is mentioned above, but is that realistic both from the standpoint of software development (e.g., will every wallet that wants to do that have to write thousands of lines of code to deal with Bitcoin Core's limited interfaces) and from actual usage (e.g., I don't want to run an always-on node on my mobile device; I'd rather run a node at home and connect to it as needed from my mobile)?

In considering priorities for the future of Bitcoin Core, I would ask contributors to ask where those priorities will lead the project in five or ten years.  Will it be a greater percentage of Bitcoin users validating their transactions with their own full node, or will it be a greater percentage of users entrusting verification to industry?

-------------------------

adrienlacombe | 2025-02-28 05:44:49 UTC | #11

"For technical and historical reasons, Bitcoin Core is the only reasonable full node implementation for all Bitcoin users to run. Re-implementations of Bitcoin face significant technical challenges and appear to be doomed to have consensus bugs." from [Antoine Poinsot - Stating the obvious](https://antoinep.com/posts/stating_the_obvious/)

@AntoineP do you have an idea of what work it would take for Core to allow for other clients? I imagine such change would required a hard fork anyway?
Merci

-------------------------

ariard | 2025-02-28 16:06:05 UTC | #12

> [quote="harding, post:10, topic:1470"]
That is, they need to be direct users of Bitcoin Core **and** have their wallet connected to their node.
[/quote]

A pedantic point, there is no necessity to have the wallet directly connected to the node (e.g through SSH or Wireguard), as long as you can feed it with the block updates with whatever communication channel, including one-shot USB sticks. That was the idea with `multiprocess` to not have the node and the wallet running in the same memory space, generally more wallet and node hosts are dissociated better you’re are.

Otherwise, I agree a lot with what Dave says and that a greater percentage of Bitcoin users validating their transactions with their own full node is the *key metric* that matters.

[quote="adrienlacombe, post:11, topic:1470"]
do you have an idea of what work it would take for Core to allow for other clients? I imagine such change would required a hard fork anyway? Merci
[/quote]

No hard fork or soft fork required at all to have inter-compatible clients with Core, if this the question. And the bar has been reduced by many order of magnitude with the `[libbitcoinkernel](https://github.com/bitcoin/bitcoin/issues/24303)` becoming mature. It’s supposed to be a standalone library encompassing all bitcoin consensus rules, and you’re free to re-write all the others components as you wish. If you find consensus bugs or DDoS issues while hacking on `libbitcoinkernel` be at minima responsible and report them.

-------------------------

AntoineP | 2025-02-28 16:50:32 UTC | #13

[quote="adrienlacombe, post:11, topic:1470"]
@AntoineP do you have an idea of what work it would take for Core to allow for other clients? I imagine such change would required a hard fork anyway? Merci
[/quote]

A lot of work was put in the past 5 years [^0] to separate the consensus critical logic in Bitcoin Core. The result of this work is [libbitcoinkernel](https://github.com/bitcoin/bitcoin/issues/27587), a library encapsulating Core's validation engine. The primary purpose of this library was to untangle the consensus-critical part from the rest of the software, but it also [makes it possible for others to build competing node implementations](https://thecharlatan.ch/Kernel) while still giving reasonable consensus compatibility guarantees to their users.

There is no need for a Bitcoin hard fork to allow for competing full node implementations. But short of something like libbitcoinkernel being available, users of these re-implementations hard forking themselves out of the network is indeed the [most](https://delvingbitcoin.org/t/cve-2024-38365-public-disclosure-btcd-findanddelete-bug/1184?u=antoinep) [likely](https://delvingbitcoin.org/t/disclosure-btcd-consensus-bugs-due-to-usage-of-signed-transaction-version/455) [out](https://github.com/libbitcoin/libbitcoin-system/issues/1523)[come](https://github.com/libbitcoin/libbitcoin-system/issues/1526).

However if you are hinting at reimplementing Core with a scope such as the one i am proposing in my post, i think this is a terrible idea. It would just further dilute the already scarce resources allocated to the maintenance of the network.

[^0]: Work was put into it before then too but in 2020 it became its own sub-project and an explicit priority of the group.

-------------------------

AntoineP | 2025-03-03 03:04:56 UTC | #14

@harding thanks for your response. In your message you push back against my characterization of Bitcoin Core users being all Bitcoin users, and argue end users running full nodes are key to the security of the network and therefore making this more accessible should be a priority of the Core project.

With regard to the former, it seems pretty clear all users of Bitcoin are indirectly affected by the quality of the Bitcoin software. I don't think users interacting directly with the software should be prioritized over those interacting with the network in a different manner. You could even stretch this to say all Bitcoin users are Core users, only some use the private RPC interface and others the public P2P interface. Instead of basing priorities on how users interface with the software i think we should consider stakes. The Core GUI is potentially important to a handful of non-technical users to be able to use Bitcoin at all. The Core wallet is potentially *very* important to a non-trivial share of Bitcoin users, whether to make payments (6% of all transactions nowadays may be crafted by the Core wallet[^0]) or to store value (as the oldest wallet and probably the one with the strongest compatibility guarantees). The Core node is critically important to every single Bitcoin user. Given the size of the network today, it's pretty clear the stakes for the Core node are so high as to dwarf those of the two other components. This is *not* to say the wallet and GUI are not important. But it should be clear that they are order of magnitudes less so than the network itself.

I wholeheartedly agree with your other point. Making running a full node accessible to end users is a key component to the security of the network. However:
1. The Core GUI is not that. It has barely seen any development [in years](https://github.com/bitcoin-core/gui/pulls?q=is%3Apr+is%3Amerged+). It is extremely inconvenient to use and people largely prefer to it other GUI interfacing with `bitcoind` directly, such as Sparrow. In fact it's not even been usable out of the box on newer Mac for a while now[^1]. As to the other non-technical end user platform, we [disabled automated unit testing on Windows](https://github.com/bitcoin/bitcoin/pull/31284) after years of flaky CI and hours of maintainers' time sunk to keep it running, ~~as nobody stepped up to help unbreak it~~ (edit: [unfair](https://delvingbitcoin.org/t/antoine-poinsot-on-bitcoin-cores-priorities/1470/15?u=antoinep)). I wish Core's GUI was the one stop for non-technical end users to just "download and run Bitcoin", but it is not.
2. It should not come at the expense of development of the Core node, as per the previous point on the respective stakes of each project. Keeping the code for the wallet and GUI imposes significant costs on the node project, which really cannot afford it right now. Sure, if Core had unlimited resources we could tackle all important issues in Bitcoin[^2]. But we do not therefore we have to focus on the most important one first.

Splitting the GUI and wallet in their own projects gives you point 2. As i argued in my blog post they also provide an opportunity to make better progress on these projects and possibly resolve point 1.

Now some comments on the specifics.

[quote="harding, post:10, topic:1470"]
I think it’s weird and non-obvious to define your users as including people who don’t directly use your software.
[/quote]

I do not think it's weird to define your users by who is impacted by your software.

[quote="harding, post:10, topic:1470"]
My ISP presumably uses a bunch of software to manage the network that connects my home to the Internet. Providing connectivity to me, and thousands of customers like me, is the ultimate purpose of that software—but its user is the ISP.
[/quote]

I do not think it's a good analogy. Your ISP is a company providing you a service. Bitcoin is a public consensus system. If your ISP runs buggy software it may prevent you and other customers from accessing the Internet, but not other Internet users. If Bitcoin Core releases buggy software, it will likely impact Phoenix users.

[quote="harding, post:10, topic:1470"]
That distinction is important because when that software forces a choice between tradeoffs onto my ISP, they *as the user* will make the choice that’s best for them and not necessarily for me.
[/quote]

Besides disagreeing with the analogy i also disagree with this statement in isolation. Surely your ISP's software provider would prioritize "not breaking Internet access to my client's clients" over "shiny new way of my direct user to interface with my software".

[quote="harding, post:10, topic:1470"]
Similarly, if an industrial full node operator who serves data to customers is forced to choose, they will make the choice that’s best for them and not necessarily for their customers.
[/quote]

This is an additional concern about this[^3]. I think people falling pray to [demagogy that feeds on some of these truths](https://x.com/jamesob/status/1857049961235403101) to reach dangerous conclusions, could lead to a significant part of the network running significantly less qualitative software, or worse.

[^0]: Based on transaction fingerprinting. I have heard this number from reputable sources but have not checked it for myself.
[^1]: Since the binaries of all previous releases are not notarized, to run the software users would have to self-sign the binary themselves from the command line (see the 6 years old issue [here](https://github.com/bitcoin/bitcoin/issues/15774)). Hardly something we can expect non-technical users to do. This might get fixed before the upcoming 30.0 release, see [this PR](https://github.com/bitcoin/bitcoin/pull/31407).
[^2]: Cf the first two paragraphs of the [second post](https://antoinep.com/posts/stating_the_obvious) in my series.
[^3]: Cf the next to last paragraph of the [same post](https://antoinep.com/posts/stating_the_obvious).

-------------------------

ajtowns | 2025-03-02 00:48:13 UTC | #15

[quote="AntoineP, post:14, topic:1470"]
we [disabled automated unit testing on Windows](https://github.com/bitcoin/bitcoin/pull/31284) after years of flaky CI and hours of maintainers’ time sunk to keep it running, as nobody stepped up to help unbreak it.
[/quote]

That doesn't seem a very fair characterisation considering the existence of 

https://github.com/bitcoin/bitcoin/pull/31176

[quote="AntoineP, post:14, topic:1470"]
It should not come at the expense of development of the Core node, as per the previous point on the respective stakes of each project.
[/quote]

I don't think I agree with that -- rather, I'd say that if we have new developments we want to make to core that we can't make accessible to end users, then the feature isn't fully baked and either shouldn't be merged at all or should only be merged with the understanding that there's a lot more work to do to complete it. If, for instance, we separate out the GUI into a separate repo, then that still means we should block PRs that add features while breaking the API that the GUI relies on.

To me, providing wallet features (mostly) and a GUI (to a lesser extent, IMO anyway) is a way of keeping us honest to the princple of bitcoin being usable by a decentralised bunch of hackers, versus being something that you can only really use if you're a whale or an established corporation willing to make a big investment. (And not providing sufficient features to mine on mainnet directly is something important that's missing to complete that featureset, really)

Making the wallet or GUI not be a reproducible build, or not be well tested on major platforms, or not be performant, would be a bit of a regression in holding to that goal, the way I see things.

-------------------------

adrienlacombe | 2025-03-03 06:51:12 UTC | #16

thank you @AntoineP @ariard

-------------------------

AntoineP | 2025-03-03 15:39:24 UTC | #17

I gave a presentation on this topic last week at the Coredev meetup. Slides are [available here](https://antoinep.com/presentations/coredev_2025_scope.pdf).

-------------------------

AntoineP | 2025-03-03 16:19:05 UTC | #18

Some more comments on the specifics points raised by @harding i had not replied to yet.

[quote="harding, post:10, topic:1470"]
Certainly, those don’t need to be bundled with Bitcoin Core, but if the developers of the Bitcoin Core project cease to put effort into making a node-backed wallet accessible, why would we expect anyone else outside the project to go through the frustrating process of trying to get PRs merged into Bitcoin Core that will provide them with the interfaces they need?
[/quote]

I am not sure to understand how this is related at all. I expect people would go through this process for the same reason they do now: because they need the interface and/or think it's useful for Core to have.

[quote="harding, post:10, topic:1470"]
Anyone who has tried to do that in the past has received a litany of “noes”: no, we’re not going to add another index; no, we’re not going to extend the P2P interface to provide unverifiable info; no, we’re not going to keep SSL to allow remote RPC access; etc…
[/quote]

Same, i don't see how it's related. It's also completely misrepresenting the situation: the way you present it sounds like Core contributors would reject adding any new interface regardless on the merit of having the interface at all.

I believe the opposite is true on both counts. It seems to me Core's position has historically been yes by default if there's been enough reviews and it's not a bad idea. And the cases where contributors objected they did so with very good reasons.

Could you point to some examples? The ones you give strike me as bad ideas that fortunately did not make it in (spk index, SSL for remote RPC access). And there is plenty of examples of new indexes and interfaces being added in recent years: on the top of my head block filters and coinstats for instance.

[quote="harding, post:10, topic:1470"]
Or will there be even more reason to say “no” to wallet-supporting features in the node because all of the focus is on optimal verification and relay?
[/quote]

The wallet interface is not going anywhere. In addition with multiprocess all wallets will be able to use the same interface as the Core wallet, which for a long time had "privileged" access.

I don't see how you could infer from my posts that i would argue for pruning interfaces that would be consumed by wallets (or object to introducing more if there is a good reason to). Obviously a node that you can't use to check your received payments is useless to an end user and nobody is proposing this.

[quote="harding, post:10, topic:1470"]
The idea of a node being embedded in wallet software is mentioned above, but is that realistic both from the standpoint of software development (e.g., will every wallet that wants to do that have to write thousands of lines of code to deal with Bitcoin Core’s limited interfaces) and from actual usage (e.g., I don’t want to run an always-on node on my mobile device; I’d rather run a node at home and connect to it as needed from my mobile)?
[/quote]

I'm confused. Are you pointing to my suggestion of having the Core wallet project release a `bitcoind` that embeds `bitcoin-node` + `bitcoin-wallet`? If so this is completely orthogonal to splitting the projects, this would happen with multiprocess anyways.

With regards to the "thousands of lines" comment, i'm not sure what it corresponds to but as i point out in my previous comment the most popular way of running a full-node backed wallet today is probably Sparrow which does exactly that. Liana does too. In addition, multiprocess will give them access to an enhanced interface to get information and signals from the node.

Finally about whether it's realistic for people to run a node locally, this is again orthogonal... People that do today would still be able to use whatever software they use (Sparrow, Core, Liana, ..) and people that prefer to run a node at home and connect to -say- an Electrum server running on top would still be able to.

-------------------------

harding | 2025-03-07 02:58:07 UTC | #19

> I don’t think users interacting directly with the software should be
> prioritized over those interacting with the network in a different
> manner.

IMO, each contributor to Bitcoin Core should prefer prioritization of
development that benefits the type of user that they most want to use
Bitcoin Core.

I'm not a Bitcoin Core contributor, but as a user of Bitcoin Core and
author of Bitcoin documentation, the type of user that's most important
to me are individuals who strongly desire preservation of Bitcoin's
current core principles, such as confiscation resistance, censorship
resistance, and monetary supply consistency.  The security of **my**
bitcoins, and **my** ability to spend them autonomously, depends on
enough of those highly motivated individuals connecting their wallets to
their own personal full nodes so that there's no question that the
current consensus rules will be enforced with 100% reliability.

I can see how non-validating and partially validating users ("weak
validators") may benefit me in multiple ways, such as increasing the
economic reach of Bitcoin and potentially increasing the market value of
the currency.  But even universally accepted bitcoins at a high market
price are not what I want if weak validators become economically
predominate and allow the third-parties who control the remote nodes
they use to accept changes to the protocol that allow theft, censorship,
or siphoning of **my** value.

> Keeping the code for the wallet and GUI imposes significant costs on
> the node project, which really cannot afford it right now.

Why is that?  Nakamoto developed the initial wallet and GUI alongside
the node as a single individual with no funding (AFAIK).  Ten years ago,
when only a handful of Bitcoin Core developers had funding, I created a
[marketing
website](https://web.archive.org/web/20150930013906/https://bitcoin.org/en/bitcoin-core/)
for Bitcoin Core that highlighted a [wallet and
GUI](https://web.archive.org/web/20150930021614/https://bitcoin.org/en/bitcoin-core/features/user-interface)
that were comparable to many other contemporary wallets (and better than
any in at least a few ways).  Today, there's more highly active Bitcoin
Core developers than ever before and every one of them that wants
funding has it.  There are also dozens (hundreds?) of other projects
with GUI-based wallets, almost none of them with a significant fraction
of the funding that's overall provided to Bitcoin Core contributors.

It looks to me like Bitcoin Core development is well funded, at least
compared to historic norms, and that developing a wallet and GUI isn't
an extraordinary undertaking.  Why do you think the project in such dire
straits that it cannot support continued development of its wallet and
GUI components?

> If Bitcoin Core releases buggy software, it will likely impact Phoenix
> users.

I agree, but it's also the case that if everyone with a wallet connects
to Bitcoin Core through a third party, even if just for a brief period
of time, then those third parties will be able to arbitrarily change
Bitcoin's consensus rules.  Ultimately, does it matter to Phoenix users
whether their bitcoins were stolen due to a bug in Bitcoin Core or
because not enough individuals connected their wallets to Bitcoin Core?

> It’s also completely misrepresenting the situation: the way you
> present it sounds like Core contributors would reject adding any new
> interface regardless on the merit of having the interface at all.

In the past decade, the main Bitcoin Core project has added almost no
interfaces that downstream wallets would find useful and has removed
several interfaces that were used by millions of wallet users.  In each
case, the rejection or removal was made with a reasonable explanation,
but the net effect has been that I do not expect Bitcoin Core to provide
interfaces useful for connected wallets.  I think many downstream wallet
devs may share my view.

If it's not a priority to change that in the near future after years of
failing to provide widely used interfaces, then I think it's fair for me
to argue against a claim that [the purpose of the system is what it
constantly fails to
do](https://en.wikipedia.org/wiki/The_purpose_of_a_system_is_what_it_does).

> Could you point to some examples? The ones you give strike me as bad
> ideas that fortunately did not make it in (spk index, SSL for remote
> RPC access).

Let's look at those examples in detail.  A scriptPubKey (spk) index has
been proposed multiple times (AJ linked the [most recent
attempt][bitcoin core #14053] from 2018) and represents a few hundred
lines of code before tests.  It would allow wallets currently used by
millions of users to connect to those user's full nodes (with either
some small modification to the wallet software or by Bitcoin Core adding
some more code to support mature spk-requesting protocols).  

AFAIK, the main
[argument](https://github.com/bitcoin/bitcoin/pull/14053#issuecomment-747648440)
against it has been that "it's a bad idea for any infrastructure to be
built all that relies on having fully-indexed blockchain data available
(this also applies to txindex, but we can't just remove support...)."
That quote is from 2020, but I recall essentially the same argument
being made years earlier.  I personally agree that relying on full spk
indexing is not scalable in perpetuity and significantly increases the
cost of operating a wallet for all users who chose this model due to the
disk space requirements.  But it seems worse to either prevent millions
of users from privately validating using their own node or to require
them to install separate software in order to use an spk index anyway.

SSL for remote RPC access was always problematic on multiple levels, but
despite that some wallets implemented support for it, indicating demand.
After RPC-SSL support was
[removed](https://github.com/bitcoin/bitcoin/blob/d6a92dd0ea42ec64f15b81843b4db62c7b186bdb/doc/release-notes.md#ssl-support-for-rpc-dropped),
demand remained and some of those wallets instead
[recommended](https://x.com/hrdng/status/1053629086617223168) users bind
RPC to a their public IP to accept any incoming connections, which was
[worse](https://bitcoinops.org/en/newsletters/2018/10/23/#over-1-100-listening-nodes-have-open-rpc-ports).
Despite this demonstrated demand from wallet users to remotely connect
to their own personal node for RPC access, no secure interface in
Bitcoin Core for doing so has ever been provided.  The best that's been
offered is a recommendation to use stunnel or an httpd reverse proxy,
which is not something I expect everyday users to do.

If you would like, I can cite at least a half-dozen other examples of
interfaces that would have allowed individual wallet users to validate
transactions against Bitcoin Core without requiring any additional
software or complicated set up.  In every case, the interface was either
rejected, brainstormed but never fully developed, or (if it already
existed) removed or neutered.  As I wrote earlier, each occasion was
accompanied by a reasonable explanation, but the net effect has been
that almost no wallets today can be connected to Bitcoin Core by a
typical user unless that user operates a third-party-maintained software
stack (like the NodeBox mentioned by @ajtowns above).

You mentioned Sparrow wallet and Liana as examples of wallets people
prefer to use to Bitcoin Core GUI, but I note that they both appear to
use a watch-only Bitcoin Core wallet in the background, similar to the
approach pioneered by Electrum Personal Server (EPS) and advanced by
Bitcoin Wallet Tracker (bwt).  I take this as further evidence that the
node itself does not currently provide useful interfaces for personal
full validation.

> there is plenty of examples of new indexes and interfaces being added
> in recent years: on the top of my head block filters and coinstats for
> instance.

How does coinstats help people validate their wallet transactions with
their personal full node?  My understanding is that the main advantage
of coinstats is that it makes it easier to validate UTXO files for
assumeutxo, which is something almost entirely done by devs and power
users.

IIRC, adding support for compact block filters to Bitcoin Core was an
uphill battle.  It took almost three years from when the [first
PR](https://github.com/bitcoin/bitcoin/pull/12254) for it was opened
until the final [main PR](https://github.com/bitcoin/bitcoin/pull/19070)
was merged.  By the time it was merged, one of its early and significant
contributors (Tamas Blummer) had died and the primary developer (Jim
Posen) had retired from Bitcoin protocol development.

I would be interested in any other examples you have of interfaces being
added to Bitcoin Core node in the past decade that are useful for
external wallets to validate against a user's personal full node.
Without some much more positive examples, I continue to maintain that
the modern Bitcoin Core project is hostile to the introduction of the
types of new interfaces that wallet authors want.

> I don’t see how you could infer from my posts that i would argue for
> pruning interfaces that would be consumed by wallets (or object to
> introducing more if there is a good reason to).

What the heck!  Just two paragraphs before you wrote this, you discussed
a pruned interface (RPC over SSL) and objections to adding an interface
(an spk index).  In both cases you argued exactly the way you now say
you wouldn't argue!  

> multiprocess will give them access to an enhanced interface to get
> information and signals from the node 

Is that a guarantee, or will we be back here in a year or two talking
about how it would simplify maintenance of the node to remove
multiprocess's privileged interface for wallets?  If no wallet today is
currently using the new interface, except for Bitcoin Core Wallet, how
do we know it's actually a useful interface for other wallets?

In conclusion, I'm genuinely concerned that narrowing the scope of the
Bitcoin Core project will accelerate the longstanding trend of making it
increasingly difficult for individuals to validate their wallet
transactions with Bitcoin Core.  I don't understand why you think that
scope reduction is necessary or useful.  It's also clear to me that you
and I disagree about the utility of Bitcoin Core's external interfaces
and the likelihood of those interfaces being maintained (or even
improved) if the project scope is narrowed.  Rather than prioritize the
production of bug-free software, I think it would be great to see the
project prioritize getting (say) a million users to validate their
wallet transactions with their own full nodes (hopefully along a path
that's still reasonably good at minimizing the number of bugs).

Thank you for the opportunity to discuss this subject.

-------------------------

