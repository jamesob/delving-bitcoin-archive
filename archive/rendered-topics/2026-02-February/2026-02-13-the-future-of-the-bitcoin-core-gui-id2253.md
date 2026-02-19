# The future of the Bitcoin Core GUI

AntoineP | 2026-02-13 15:04:59 UTC | #1

A year ago i posted a call to action regarding Bitcoin Core's priorities, which led to interesting discussions between some Bitcoin Core contributors and some community members in [this thread](https://delvingbitcoin.org/t/antoine-poinsot-on-bitcoin-cores-priorities/1470). Not much has really happened since, in terms of taking a decision on priorities, or even on agreeing to a common goal for the project. Until today, when the Bitcoin Core GUI maintainer [called during the weekly Bitcoin Core meeting](https://www.erisian.com.au/bitcoin-core-dev/log-2026-02-12.html#l-203) for the deprecation of the GUI (which has been on life support for half a decade) and its eventual removal.

I open this thread to revisit this topic in light of this update and enable a longer form -and hopefully more productive- discussion than the IRC meeting allows for.

First and foremost, i must say it is frustrating how much wishful thinking is involved every single time this topic is brought up. I hope we can have a discussion without speaking in absolutes but considering the reality of the limited resources of the project.

<details>
<summary>Click to expand longer rant on wishful thinking</summary> 

 During this week's meeting we again read statements such as "users should (:tm:) be able to use Bitcoin without a third-party frontend", or "we must ship a GUI". Of course any contributor to Bitcoin Core would agree that all other things equal, making Bitcoin Core more useful and accessible is better! The point is obviously that all other things are **not equal**, and the decision for Bitcoin Core to ship a GUI should not be met with an unrealistic "We Must (:tm:)", but rather beg the question "at what costs?". Costs related to shipping a GUI come in various forms:
- direct burden related to maintenance of this component (bug fixing, build system compatibility, etc..);
- indirect burden related to both the presence and the active development of this component (unrelated maintenance of more important parts of the codebase ends up affecting this component, which needs fixing, or just concurrent development of this component conflicting with development in other parts of the software);
- direct risks related to shipping this component when it is significantly less reviewed and maintained than other parts of the software, and pulls significantly more less well-reviewed or naturally more at-risk dependencies;
- indirect risks related to shipping this component (risks from one component could [leak](https://github.com/bitcoin/bitcoin/issues/29914) into another, it invites code churns, and of course any limited resource spent on derisking shipping this component is not allocated to derisking more critical components);

The costs exist. You may argue they are manageable, or that the benefits of shipping a GUI outweigh them. But simply stating that we must push through no matter the costs is absolutely unconvincing, and growingly frustrating after years of no development, barely any maintenance and tons of funding poured in dead-end projects that bring more churn (and therefore risk) to the project than they solve real world problems.

</details>

With that being said, let me try to summarize what i think are compelling points made during the meeting to try and kick-off discussions.

@Hebasto starts off by stating:
> Deprecate and abandon the Bitcoin Core GUI. It is insufficiently developed and reviewed, and current users have viable alternatives.

That is in my opinion one of the main points of this discussion, and it was barely even discussed during the meeting. What should be our quality threshold for software we release? What should we do if we are unable to meet that threshold?

Others, some who have been involved with the GUI, [pointed](https://www.erisian.com.au/bitcoin-core-dev/log-2026-02-12.html#l-257) out they disagreed with the assessment. That seems like a **good starting point** for a discussion. If we can't agree on whether the GUI is in a shippable state, it's unlikely that we will find much agreement regarding what to do of it.

The other point raised in the gist is to call for an IPC interface for anyone to build a GUI, without the need to be tightly coupled to Bitcoin Core's internals. Making Bitcoin Core more modular, and creating public interfaces to expose functionalities to external software, appear to me entirely uncontroversial among the current group of contributors. So i think it's better to focus the discussion on the previous point of contention, that is deprecating the GUI.

Another point that was brought up is how the Bitcoin Core GUI is the only way for non-technical users to interface with a Bitcoin Core wallet. If we stopped shipping the GUI in version 34.0 (roughly, end of 2027) as per the GUI maintainer's suggested plan, users would either not be able to upgrade anymore or be forced to use the CLI. So a followup question to the previous point is assuming that we believe the GUI does not meet the bar to be shipped by Bitcoin Core, do we believe that shipping it is still the least bad option in practice, because it would otherwise lead to users not being able to upgrade?

An alternative of course is that by the time we stop shipping the GUI, a replacement based on the IPC interface would be available to users. Some people [raised concerns](https://www.erisian.com.au/bitcoin-core-dev/log-2026-02-12.html#l-294) that since Bitcoin Core has shipped a GUI for so long, it disappearing in favour of alternatives not distributed by Bitcoin Core may be a fertile ground for distributing malware. If reasonable independently-distributed alternative(s) exist, do we believe we can then stop releasing a GUI? Or should we keep releasing one forever?

It's worth pointing out the IPC interface for those alternatives is not currently available in any released version of Bitcoin Core, and most likely also won't be included in the upcoming version 31. That being said, multiple contributors pointed out that IPC is not a strict requirement for a GUI. The available RPC interface enables an application to have access to wallet functionalities and therefore be compatible with Bitcoin Core wallets. Nobody has ever built an RPC-based GUI specifically to manage a Bitcoin Core wallet, although there exists GUIs built on top of the Bitcoin Core RPC interface to manage a more modern wallet (backed by a watchonly Core wallet for monitoring the chain).

Finally, all those points raised during the meeting are about direct costs (is it reasonable to ship the GUI? Who will take care of maintaining it?), but i'm also interested in what people think about indirect costs and risk. Do we think the indirect burden and risks of keeping the GUI around is worth it? If so, who will make sure it at least does not break, or better yet maintain it?

-------------------------

AntoineP | 2026-02-13 17:58:59 UTC | #2

In my opinion, we should center the discussion around support for existing Bitcoin Core wallet users.

It is [obvious](https://antoinep.com/posts/stating_the_obvious/) to me that if Bitcoin Core never had any wallet or GUI, it would be ridiculous to suggest that Core contributors take some time off maintaining the software underpinning a trillion-dollars network to develop and maintain yet-another Bitcoin wallet/GUI. Of course, this is not to say that those have not been historically important components, material to Bitcoin's success. But Bitcoin is very different now than when they were introduced. There are now superior alternatives to both, and the mission of maintaining the node has become critical, with orders of magnitude more people (and value) that depend on it, and this is where our focus should be.

That said, precisely because Bitcoin Core has shipped with a wallet and a GUI for so long, such a shift can't happen overnight. So i agree on the goal, but i am not sure about the timeline. I agree with marking it as deprecated ASAP, if only to set expectations for users that it does not meet the bar we set for ourselves and may be removed in the future. But there is currently no reasonable replacement to manage a Bitcoin Core wallet. So to buy into the complete removal timeline i would like to first be convinced that the burden of keeping the GUI around is a higher cost than its users being unable to upgrade their node and their wallet past version 34.

Let me also try and propose and alternative plan. My main issue with the proposed timeline is that it may lead to part of our users being stuck on version 34, and not be able to get security updates for their full nodes nor their wallet. Of course if nobody is willing to maintain this component anymore, we can't force them to keep doing it, and users will have to move to a different wallet or use the CLI. That's not the end of the world, but i hope we can offer a smoother transition, whereby users may not update their GUI until there is interest in developing it, but may still be able to receive security updates for their full node and wallet.

What i'm suggesting is to:
1. Let go of the sunk cost fallacy of the update to QML for the time being. Clearly this is not happening, or at least not until the GUI is cleanly split off from the main Bitcoin Core project and people can more easily iterate.
2. Release the existing GUI only as a separate process that talks to the node and the wallet through the IPC interface.
3. Split off the GUI into its own project, released at its own schedule.

I had previously heard that QT widget would be deprecated, but turns out it [will not](https://www.erisian.com.au/bitcoin-core-dev/log-2026-02-12.html#l-420). So that seems reasonable to focus on the existing GUI than the forever-1-year-away one.

We have as a project already chosen to bear most of the cost of multiprocess (additional dependencies), so we may as well reap all the benefits and include the remaining interfaces. Of course that's still a lot of work. Off the top of my head we'd need to:
- Merge the remaining interfaces;
- Merge process spawning logic;
- Figure out Windows support for multiprocess;
- Figure out some stability guarantees for the interface that would be used by the GUI.

There would also be costs associated with a split, notably in terms of some code and build system logic duplication. That's a non-trivial amount of work (though most items have been implemented already), but not more than building an entirely new GUI, and in my opinion preferable to stopping to release a GUI altogether when no alternative is available. This also enables the competition between alternative GUIs talked about in the gist, if anybody becomes interested in taking part in that. Delivering this for version 34, as per the original timeline, seems achievable.

Now i'll try to answer the specific questions i asked everyone in OP.

[quote="AntoineP, post:1, topic:2253"]
What should be our quality threshold for software we release? What should we do if we are unable to meet that threshold?
[/quote]

Inertia is always a strong force, and i think especially so in leaderless structures such as ours where there is noone to call the shots and impose hard (but sometimes necessary) choices. If we believe that a piece of software we release is significantly less vetted than others, it is paramount that we let users know. Stopping to ship it is a more difficult tradeoff, as discussed above, but making more information available comes with little downsides. In practice i think this could translate into a warning included on our download page.

[quote="AntoineP, post:1, topic:2253"]
they disagreed with the assessment. That seems like a **good starting point** for a discussion. If we can’t agree on whether the GUI is in a shippable state
[/quote]

I cannot talk to the quality of the code in the GUI because i am not familiar with it (like, i think, the vast majority of contributors). Development is certainly slow ([20 merged PR created in the past year](https://github.com/bitcoin-core/gui/pulls?q=is%3Apr+is%3Amerged+created%3A%3E%3D2025-02-13+)), but not dead to the point of raising a red flag. A bit more concerning is the number of reported issues that go unfixed for years (for instance just [search for "crash"](https://github.com/bitcoin-core/gui/issues?q=is%3Aissue%20state%3Aopen%20crash) on the issue tracker), or the fact that [it is half-broken in some circumstances which may introduce more risks than it solves problems](https://www.erisian.com.au/bitcoin-core-dev/log-2025-09-19.html#l-76), so somewhat of an orange flag here.

Overall i do not think this software is in such a bad shape that it would be irresponsible for us to ship, as long as we make it clear to users it does not receive the same level or review as the rest of Bitcoin Core. In my view the issue with the GUI are more on the indirect cost it imposes on the project, more on that below.

[quote="AntoineP, post:1, topic:2253"]
do we believe that shipping it is still the least bad option in practice, because it would otherwise lead to users not being able to upgrade?
[/quote]

I think there are two suboptimal paths we want to avoid. These are 1) breaking compatibility for unsuspecting users too soon 2) not be able to deprecate the GUI and eventually split if off to focus on maintaining the Bitcoin network. Unfortunately we don't have any obviously-good, simple solution, and currently i think my suggestion from above is the least bad option but i am happy to be convinced otherwise by either side of this argument.

[quote="AntoineP, post:1, topic:2253"]
indirect costs and risks
[/quote]

As i have previously shared at length in previous posts and in person, i think this is the major issue with mission creep. Small, dispersed, costs that accumulate over time and across contributors. Trying to do everything in one software invites code churn that risks impacting every single Bitcoin user, only to potentially benefit a tiny portion thereof. In the case of the GUI, for instance it could be [code or build system updates of the main project](https://github.com/bitcoin/bitcoin/pulls?q=is%3Apr+in%3Abody+qt+OR+gui) for GUI purposes, or GUI related changes [hindering](https://github.com/bitcoin/bitcoin/pull/33494#issuecomment-3380876967) development [of](https://github.com/bitcoin/bitcoin/pull/33511#issuecomment-3360247170) the [node](https://github.com/bitcoin/bitcoin/pull/32998#issuecomment-3082011233). These happen all the time, but another one worth mentioning is when [a multiprocess bugfix](https://github.com/bitcoin/bitcoin/pull/33511#issuecomment-3360247170) for the node could not be applied, because it broke the GUI, so the GUI had to be unbroken before the full node could be fixed.

-------------------------

neonrooks | 2026-02-14 09:17:15 UTC | #3

I use the Bitcoin Core gui (bitcoin-qt) exclusively because it is more convenient for me. I’m concerned about one day not being able to update after the gui is depracated. Is there a solution for non-technical users?

-------------------------

1440000bytes | 2026-02-14 13:41:24 UTC | #5

[quote="neonrooks, post:3, topic:2253, full:true"]
I use the Bitcoin Core gui (bitcoin-qt) exclusively because it is more convenient for me. I’m concerned about one day not being able to update after the gui is depracated. Is there a solution for non-technical users?
[/quote]

I think the node could have a web UI like i2p router web console. Not sure about the wallet.

<details>
<summary>Archive</summary>

https://web.archive.org/web/20260214133933/https://delvingbitcoin.org/t/the-future-of-the-bitcoin-core-gui/2253/5

</details>

-------------------------

janb84 | 2026-02-16 10:39:56 UTC | #6

**Thanks for this write‑up.**

First of all, I want to say I’m a big proponent of having choices and competition.

I would suggest marking the GUI as being in **“maintenance mode,”** which would accurately represent its current state of development (it’s missing new features, etc.). This would also give GUI users an honest picture of its status without deprecating it outright. Stating that the GUI is in “maintenance mode” also gives people who aren’t following day‑to‑day development an option to respond. They can step up and help bring the GUI to a point where we can remove the “maintenance mode” designation or, if no meaningful response emerges, move toward deprecation. AKA **“use it or lose it.”**

Directly **deprecating** the GUI could trigger a negative reaction from those who don’t track daily development (with all the consequences that entails). I would like to prevent that.

Regarding your suggestions:

1. **“Let go of the sunk‑cost fallacy of the update to QML … to iterate more easily.”**
   Yes, this feels unlikely to land. I would say we “remove” it from the project but feel free to work on it anyway. It’s not a goal of bitcoin core anymore.
   **(1.b)** Mark the current GUI as being in “maintenance mode.”

2. **“Release the existing GUI only as a separate process that talks to the node and the wallet through the IPC interface.”**
   Yes, this seems the best way forward to create options and allow interaction with Bitcoin Core without tight coupling. If this has to be IPC or RPC? idk.

3. **“Split off the GUI into its own project, released on its own schedule.”**
   In time, perhaps. The downside is that “the project” would no longer have direct control over quality, and the GUI could become just “one of many.” if that’s a bad thing ? IDK.

-------------------------

redundant | 2026-02-16 01:52:29 UTC | #7

Changing the GUI to run as a separate process with IPC communication sounds like a good next step. It’s impossible to get a hard count on the number of users who rely on the GUI for interacting with their bitcoin so I don’t think it should be dropped just yet. My long-term preference is to keep it packaged with Bitcoin Core but have it maintained as a separate repository (like secp256k1). That way maintenance doesn’t have to be in the hands of those who don’t want to work on it, while there would still be an ‘official’ Bitcoin Core GUI.

-------------------------

adys | 2026-02-17 01:20:54 UTC | #8

I believe removing the GUI will negatively impact node adoption and create significant security risks by forcing users toward third party alternatives. If Bitcoin Core includes a wallet, it needs a UX.

I think Core should provide a full solution. In an era where malicious interfaces are so easy to generate, providing a UX that carries the same trust level as the underlying node is vital for user safety.

We often underestimate the power of a good UX to attract users. If the interface is great, more people will use it, and we all want more people to run nodes.

The GUI is actually how I got my start in Bitcoin.

-------------------------

andyschroder | 2026-02-17 02:50:30 UTC | #9

I haven’t read through all the arguments, but I’m generally in favor of removing the GUI. I feel like inter process communication might be too tight of a coupling through. I feel like if some of the issues within

https://github.com/bitcoin/bitcoin/issues/29183

could be fixed and we could have better pruned node support that can take efficiently take advantage of watching only wallets like Sparrow Wallet does through the already existent RPC. Sparrow is a killer UI and the main reason I think it doesn’t encourage connecting to a local instance of bitcoin core (or even bundling it with Sparrow) is because wallet rescan is too slow and difficult to manage when nodes are pruned. I think a lot of people can deal with a one time 1-2 day wait to do full validation if they don’t need to use much disk space after that and can do a rescan more quickly for any new wallet that needs to be added. If that barrier were eliminated, I think there could be a lot more Sparrow Wallet users running bitcoin core locally and more wallets that are built that have a similar way that they interact with bitcoin core (and potentially bundle it) as Sparrow Wallet does.

-------------------------

AntoineP | 2026-02-17 16:26:49 UTC | #10

Thanks everyone for the feedback. It's good to hear from "power" users[^1]. I hope to hear from more contributors to the Bitcoin Core project too.

[quote="neonrooks, post:3, topic:2253"]
Is there a solution for non-technical users?
[/quote]

That is what this thread aims to figure out. I hope we can find a solution that is not leaving existing GUI users unable to upgrade their node (and wallet internals), but also isn't continuing to take resources from supporting all Bitcoin users in order to benefit a minority thereof (those using Bitcoin through the Bitcoin Core GUI). This is the motivation for [my proposal](https://delvingbitcoin.org/t/the-future-of-the-bitcoin-core-gui/2253/2?u=antoinep) above.

[quote="janb84, post:6, topic:2253"]
I would suggest marking the GUI as being in **“maintenance mode,”** which would accurately represent its current state of development (it’s missing new features, etc.). This would also give GUI users an honest picture of its status without deprecating it outright.
[/quote]

I think deprecating would be preferable as i don't see how it could be the long term goal of the Bitcoin Core project to maintain a GUI. I guess the counter to that is deprecating without a clear timeline could be confusing to users.

I am fine with starting by marking it in "maintenance mode" if that is more consensual among contributors to the project.

[quote="janb84, post:6, topic:2253"]
They can step up and help bring the GUI to a point where we can remove the “maintenance mode” designation
[/quote]

That is where i disagree. We have been doing that for the past 5 years, with the result that we are seeing now. I think maintenance mode should be as much an acceptation of the reality as a statement from the project, that the only purpose of the GUI is to support existing users of the wallet. If you want a modern GUI with new features, there are existing better alternatives for it. If someone wants to build a modern GUI for the Bitcoin Core wallet, this should be a separate project, possibly IPC-based, but more straightforwardly RPC-based.

[quote="janb84, post:6, topic:2253"]
Yes, this seems the best way forward to create options and allow interaction with Bitcoin Core without tight coupling. If this has to be IPC or RPC? idk.
[/quote]

The chain and wallet IPC interfaces are probably worth it regardless of someone building a Core wallet GUI, so it doesn't matter too much with regard to making a decision. I guess one could build it on top of IPC to do like the cool kids, but if what they want is a pragmatic GUI that just works for everyday people then RPC is just fine and will be compatible with existing Bitcoin Core releases as well.

[quote="janb84, post:6, topic:2253"]
In time, perhaps. The downside is that “the project” would no longer have direct control over quality, and the GUI could become just “one of many.” if that’s a bad thing ? IDK.
[/quote]

Many are attached to the idea that Bitcoin Core should ship a GUI. As long as it is not tied to the full node, and users are informed it does not receive the same amount of vetting as the full node software does, as you could expect of a standalone GUI software vs a full node software, then i think it's fine for Bitcoin Core to keep shipping it on its own schedule.

[quote="redundant, post:7, topic:2253"]
My long-term preference is to keep it packaged with Bitcoin Core but have it maintained as a separate repository (like secp256k1). That way maintenance doesn’t have to be in the hands of those who don’t want to work on it, while there would still be an ‘official’ Bitcoin Core GUI.
[/quote]

What really matters is releases not Github repositories. The GUI is already in [its own Github repository](https://github.com/bitcoin-core/gui) at the moment, but still shares code, build system, CI, and release schedules with the [main project](https://github.com/bitcoin/bitcoin). In order to be completely separated, it needs to be split off from the release, and the sources, of the node.

[quote="adys, post:8, topic:2253"]
I believe removing the GUI will negatively impact node adoption and create significant security risks by forcing users toward third party alternatives.
[/quote]

All other things equal, sure. But we should not overstate how much bitcoin-qt contributes to adoption nowadays. Someone introduced to Bitcoin today and want to use a full node, they won't (and probably shouldn't) create a no-mnemonics hot wallet on bitcoin-qt. Instead, they use a signing device that they connect to a modern wallet, and connect that to their bitcoind.

[quote="adys, post:8, topic:2253"]
I think Core should provide a full solution. [...] a UX that carries the same trust level as the underlying node is vital for user safety.
[/quote]

Two things.

First, you seem to be arguing from a position of ignoring all the costs. Of course if we ignore all the downsides, anybody agrees that the upsides are worth it! The subset of Bitcoin Core contributors that argues for removing, or separating, the GUI from the full node are not cold-hearted engineered that hate users, but they consider that we should not overlook the dispersed costs to all Bitcoin users of only looking at the concentrated benefits to a minority of Bitcoin users.

It may well be that the benefits of shipping a GUI outweigh the associated costs, but you are not going to convince anybody who thinks the opposite by insistently pointing at the upsides with superlatives, as if the person was unaware.

Secondly, i think it is an unreasonable expectation to set on the Bitcoin Core project, certainly as things stand but also in general, to be providing every single piece of software the vast majority of Bitcoin users interface with. Maintaining and improving the full node that underpins a trillion dollars network is no easy task, and i think we should focus all our resources on making it more robust and reducing the risk of a fuckup that would indirectly affect every single user of the Bitcoin network.

It was useful for some time to have one big monolith that does everything. But this does not scale, and i think we should move towards an ecosystem of software that do one thing, and do it well.

[^1]: I say "power" users because i expect Bitcoin Core users that are aware of the Delving forum, tracking posts there and even replying, to be among the most informed users.

-------------------------

adys | 2026-02-18 00:50:35 UTC | #11

Hi Antoine, I am a new contributor. 
 
[quote="AntoineP, post:10, topic:2253"]
All other things equal, sure. But we should not overstate how much bitcoin-qt contributes to adoption nowadays. Someone introduced to Bitcoin today and want to use a full node, they won’t (and probably shouldn’t) create a no-mnemonics hot wallet on bitcoin-qt. Instead, they use a signing device that they connect to a modern wallet, and connect that to their bitcoind.
[/quote]

I think (and know for a fact from our local community node workshops) that most new users (and not only) struggle to connect wallets to a node. It is often not trivial, even for technical people, and leads many to  abandon the idea of running a node. For us its so trivial to run shell commands, but most users will find it scary. 

I’m not suggesting that the current bitcoin-qt is helping adoption, but I do believe a full solution (node + wallet + UX) does. Isn't the fundamental value of a full node the ability to install, sync, and transact independently of any other third party software? If a user must download a second software just to interface with the first, we’ve added a dependency that many beginners aren't ready to manage.

To clarify, I am fully in favor of IPC and the move toward a new UX; I just believe that this UX should remain an official, integrated part of the Bitcoin Core project. 
 
[quote="AntoineP, post:10, topic:2253"]
Two things.

First, you seem to be arguing from a position of ignoring all the costs. Of course if we ignore all the downsides, anybody agrees that the upsides are worth it! The subset of Bitcoin Core contributors that argues for removing, or separating, the GUI from the full node are not cold-hearted engineered that hate users, but they consider that we should not overlook the dispersed costs to all Bitcoin users of only looking at the concentrated benefits to a minority of Bitcoin users.

It may well be that the benefits of shipping a GUI outweigh the associated costs, but you are not going to convince anybody who thinks the opposite by insistently pointing at the upsides with superlatives, as if the person was unaware.

Secondly, i think it is an unreasonable expectation to set on the Bitcoin Core project, certainly as things stand but also in general, to be providing every single piece of software the vast majority of Bitcoin users interface with. Maintaining and improving the full node that underpins a trillion dollars network is no easy task, and i think we should focus all our resources on making it more robust and reducing the risk of a fuckup that would indirectly affect every single user of the Bitcoin network.

It was useful for some time to have one big monolith that does everything. But this does not scale, and i think we should move towards an ecosystem of software that do one thing, and do it well.
[/quote]

True, I was speaking from an ideal functional perspective without accounting the maintenance costs for now. I’m not trying to argue, just sharing my opinion 🙂

I believe we should first define the ideal solution and then see how much of it we can achieve with available resources.

-------------------------

AntoineP | 2026-02-18 14:36:55 UTC | #12

[quote="adys, post:11, topic:2253"]
Hi Antoine, I am a new contributor.
[/quote]

Welcome!

[quote="adys, post:11, topic:2253"]
I think (and know for a fact from our local community node workshops) that most new users (and not only) struggle to connect wallets to a node. It is often not trivial, even for technical people, and leads many to abandon the idea of running a node. For us its so trivial to run shell commands, but most users will find it scary.
[/quote]

Yes, i'm well aware. Based on my own experience onboarding new users, i would never advise a non-technical person to create a hot, non-BIP39, wallet for any substantial amount.

I agree with you any decent GUI should make the connection to the full node as simple as possible. What we did for [Liana](https://github.com/wizardsardine/liana) is to give the option to a user to install a pruned Bitcoin Core locally with a single click. The bitcoind binary is downloaed from bitcoincore.org and verified, IBD is performed with a progress bar on the Liana GUI, and once finished a watchonly wallet is created to track coins. That's not rocket science and does not require anything else from Bitcoin Core besides maybe a more complete prune support as @andyschroder points out.

That said, for better or for worse, many technical users have now been used to think a full node as a physical thing, due to the success of node-in-a-box solutions. In this situation, these users "connect their wallet to their nodes" (usually through an exposed Electrum interface). Surprisingly, many non-technical users are actually more comfortable with that, and resist having their wallet just run the full node locally on their machine if they already have their node-in-a-box thing at home.

-------------------------

adys | 2026-02-18 15:03:46 UTC | #13

[quote="AntoineP, post:12, topic:2253"]
Welcome!

[/quote]

Thank you :)


[quote="AntoineP, post:12, topic:2253"]
I agree with you any decent GUI should make the connection to the full node as simple as possible. What we did for [Liana](https://github.com/wizardsardine/liana) is to give the option to a user to install a pruned Bitcoin Core locally with a single click. The bitcoind binary is downloaed from [bitcoincore.org](http://bitcoincore.org) and verified, IBD is performed with a progress bar on the Liana GUI, and once finished a watchonly wallet is created to track coins. That’s not rocket science and does not require anything else from Bitcoin Core besides maybe a more complete prune support as @andyschroder points out.
[/quote]
Yes that is great, I wish more wallets do that. 
Off topic you guys really pushed Miniscript to adoption with Liana :raising_hands:


What will be your dream option given unlimited resource?

-------------------------

ArmchairCryptologist | 2026-02-18 18:47:16 UTC | #14

Speaking as someone who used it as a wallet back when it was still named Bitcoin-Qt, but has mostly used the CLI for the last ten years or so, I do think the GUI as it is today has somewhat played out its role. Where we are today, I don’t even think casual users *should* be encouraged to run Bitcoin Core as a wallet, seeing as it is both far more simple and far more secure (as far as preventing loss of funds go) to install something like Electrum or Sparrow and link it up with a Trezor or whichever hardware wallet they prefer. Hell, even a smartphone wallet is significantly more secure than running one on your average insecure desktop. Is it less private? Definitely. Will they be second-class citizens as far as transaction validation and overall network security goes? Yup. But at least some random malware cannot just trivially nab their coins; it has to at the very least go through the trouble of tricking the user to cough up their seed phrase.

On the other hand, I’m sure some power users and system administrators would very much like a basic graphical UI for checking the status of the node, and doing basic tasks like simply shutting it down. Most of the problems you are listing seem to be involving the wallet integration, so I’m curious if it would not solve most of your issues if you simply stripped that out in its entirely, and left intact a barebones UI for status readouts and basic node control?

(Just to clarify, I’m not suggesting stripping out the wallet from the node, only the GUI parts of it.)

-------------------------

Fiach-Dubh | 2026-02-19 01:47:15 UTC | #15

Hebasto should collaborate with other devs and step aside if he can no longer find ways to develop the GUI as is, others may have a better idea on how to intervene without castrating the client entirely. Him hitting a brick wall is not a reason to immediately deprecate the GUI within one release cycle with no fallback solutions.

This proposal on it’s face seems like a huge foot gun.

-------------------------

neonrooks | 2026-02-19 01:15:48 UTC | #16

Bitcoin nodes should be accessible to anyone without the hassle of learning commands or a unix terminal. 

Also, why are we conflating gui with the wallet? Being able to see the block height, what kind of peers you are connected to, and node configuration at a glance is an experience that should be promoted as accessibility for money for everyone, and celebrated as the technical achievement that it is. Running Bitcoin is not just for technical users, or am I wrong here?

-------------------------

