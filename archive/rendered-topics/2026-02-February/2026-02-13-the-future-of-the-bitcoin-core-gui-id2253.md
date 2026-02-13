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

