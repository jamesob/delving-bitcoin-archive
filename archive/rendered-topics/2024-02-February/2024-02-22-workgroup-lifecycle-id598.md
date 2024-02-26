# Workgroup lifecycle

ajtowns | 2024-02-22 05:53:20 UTC | #1

I've just closed out the cluster mempool working group, and thought it might be interesting to document the lifecycle it went through:

1. Initial private group:
   * created a wg-cluster-mempool category for posting ideas within the group, so that thoughts didn't get lost when instant messages expired, and were easier to search than random gists
   * setup the private group by the same name, and made the category private to that group
   * seeded the category by [copying](https://delvingbitcoin.org/t/cluster-mempool-rbf-thoughts/156) over a gist
   * (This was around 2023-11-01)

2. Public but read-only:
   * after a while group members wanted to share some of the discussion with others who weren't members of the group. rather than expand the group we decided to open the existing topics up to the public
   * this involved changing the category to be publicly readable, but new posts and replies were still limited to wg members
   * at this point optech discovered and [reported on](https://bitcoinops.org/en/newsletters/2023/12/06/#cluster-mempool-discussion) those discussions
   * (This was around 2023-11-30)

3. Closing out the working group
   * with the project moving from "research" to "release" (eg, it's proposed as a [priority project for bitcoin core 28.0](https://github.com/bitcoin/bitcoin/issues/29439#issuecomment-1958088049)) there's little reason to limit discussions any longer, so closing out the working group seems to make sense
   * all the discussions have been recategorised into the main #implementation category, so they're no longer restricted to the wg in any way (either for reading or writing)
   * to make it easier for future historians to still find those discussions, there's now a #wg-cluster-mempool::tag tag that's retroactively applied to all them, and that is locked to admins, so it can't be added to future discussions
   * the old category still exists, but only has an "About" post, which has a link to the tag so you can still find the old posts
   * (And here we are at 2024-02-22)

Anyway, just documenting that as it seems like it's a process that may be useful to repeat for other projects.

-------------------------

ariard | 2024-02-26 02:38:31 UTC | #2

> * created a wg-cluster-mempool category for posting ideas within the group, so that thoughts didn’t get lost when instant messages expired, and were easier to search than random gists
> * setup the private group by the same name, and made the category private to that group
> * seeded the category by [copying ](https://delvingbitcoin.org/t/cluster-mempool-rbf-thoughts/156) > over a gist
> * (This was around 2023-11-01)

Do you have logs and list of members of such private wg-cluster-mempool ?

Because if one outcome of such working group is this https://github.com/bitcoin/bitcoin/issues/29319, this is a complete design process failure, given some of the mechanisms are either weak (v3 policy) or apparently useless (sibling evictions to remove CPFP carveout). 

While we cannot prevent private communications among members belonging to and funded by the same development entity, I think we should socially discourage private communication channels about FOSS software among members of different development entities. Lack of publicity might be jeopardizing the MIT / Apache 2 license and the de facto entrance in the domain public of Bitcoin design ideas.

At the very least, people engaging in such private communication channels should consult lawyers in the main major juridictions, if such communication practice is not specially tainting their responsibilities in case of future FOSS software defect in some way.

-------------------------

ajtowns | 2024-02-26 05:33:28 UTC | #3

[quote="ariard, post:2, topic:598"]
Do you have logs and list of members of such private wg-cluster-mempool ?
[/quote]

Yes, discourse keeps lots of logs.

[quote="ariard, post:2, topic:598"]
Because if one outcome of such working group is this [Cluster mempool, CPFP carveout, and V3 transaction policy · Issue #29319 · bitcoin/bitcoin · GitHub](https://github.com/bitcoin/bitcoin/issues/29319), this is a complete design process failure, given some of the mechanisms are either weak (v3 policy) or apparently useless (sibling evictions to remove CPFP carveout).
[/quote]

The goal of this forum is to allow people working on bitcoin projects to discuss their ideas and work towards implementing them in a constructive environment. If you want to use this forum for discussing your own projects that's great, please do so. If you want to use this forum for harassing other people, your posts doing so will be removed, and if it's repeated, your account disabled.

[quote="ariard, post:2, topic:598"]
While we cannot prevent private communications among members belonging to and funded by the same development entity, I think we should socially discourage private communication channels about FOSS software among members of different development entities. Lack of publicity might be jeopardizing the MIT / Apache 2 license and the de facto entrance in the domain public of Bitcoin design ideas.

At the very least, people engaging in such private communication channels should consult lawyers in the main major juridictions, if such communication practice is not specially tainting their responsibilities in case of future FOSS software defect in some way.
[/quote]

Neither the MIT nor Apache licenses discourage private communications, and, obviously, private communications on this site are supported and will continue to be. (However, this site does not do end-to-end encryption of messages, so it's should not be used for discussion of zero-day vulnerabilities whether via restricted groups or private messages)

What we **should** do isn't discourage private discussions, but rather **encourage** public discussion. Making legal threats and declaring results you dislike a "process failure" is the opposite of that, and you really should stop it. Be constructive: spend your time building something better.

-------------------------

ariard | 2024-02-26 18:32:45 UTC | #4

> Yes, discourse keeps lots of logs.

So can you publish them ?

> The goal of this forum is to allow people working on bitcoin projects to discuss their ideas and work towards implementing them in a constructive environment. If you want to use this forum for discussing your own projects that’s great, please do so. If you want to use this forum for harassing other people, your posts doing so will be removed, and if it’s repeated, your account disabled.

In the consideration that ideas and works discussed on this forum are now leveraged to justify changes in Bitcoin Core, there is certainly the financial interests of the end-users to consider over any laziness or incompetency arising among us, Bitcoin Core contributors (in a large fashion, whatever one level of involvement).

If we cannot discuss improvement in the development process itself, including pointing out when some reviewers are doing less than a great job, I do think we’re aiming towards stagnations or technical disasters. Personally, I don’t have any issue about the “intellectual honest”y of my works and actions being questioned, as long as everyone can defend itself it’s fair game.

>  If you want to use this forum for harassing other people, your posts doing so will be removed, and if it’s repeated, your account disabled.

If I estimate myself victim of being harassed by your behavior or you’re overriding your janitorial role in the maintenance of this communication plateform, please be known there is always the ability to pursue remediation in front of the Australian authority or any other competent one.

I don’t like this eventuality. Yet I don’t think some developers weaponing the Bitcoin public communication space to outlaw opinions questioning their actions, like you’re currently threatening to do so is either great, or sane on the long-term.

You’re more convincing me, we shall move usual Bitcoin development conversations to Nostr, on a native “distrust-the-admin” plateform. 

All that said if the burden of maintaining this communication plateform is too high for you, my pleasure if I can contribute in a constructive fashion, either paying servers costs if any or whatever. You’re free to reach out in private.

> Neither the MIT nor Apache licenses discourage private communications, and, obviously, private communications on this site are supported and will continue to be. (However, this site does not do end-to-end encryption of messages, so it’s should not be used for discussion of zero-day vulnerabilities whether via restricted groups or private messages)

Here I would recall you this great quote of Pieter Hintjens (Zero MQ) in one of its book on FOSS.
https://hintjens.gitbooks.io/social-architecture/content/chapter1.html

" Transparency is very important to get rapid criticism of ideas and work in progress. If a few people in a team go off and work on something together for some time -- a few days seems harmless, a few weeks is not -- then what they make can be presented to the group as a *fait accompli*. When one person does that, the group can just shrug it off. When two or more people do that, it becomes much harder to back off from bad ideas. Secrecy and incompetence seem bound together. Groups that work in secret do not achieve wisdom.

*TIP:* When one person does something in a dark corner, that's an experiment. When two or more people do something in a dark corner, that's a conspiracy.”

On the MIT and Apache licenses and especially public domain considerations, it can have an impact w.r.t IP, better to have everything related to deep areas of Bitcoin Core put in public as early as we can. 

> What we **should** do isn’t discourage private discussions, but rather **encourage** public > discussion. Making legal threats and declaring results you dislike a “process failure” is the opposite > of that, and you really should stop it. Be constructive: spend your time building something better.

On one hand, we cannot discourage private discussions, that’s the nature of electronic communications, and somehow it’s a spontaneous and healthy process to nurture new ideas (cf. Hannah Arendt, “The Crisis of Culture”, her essay on education).

On the other hand, the underpinning idea of FOSS is working under the spotlight and I’m really worried about a trend where major technical discussions are happening substantially in the dark (unless for the exceptions you are yourself pointing out). I agree on encouraging public discussions is the more positive social habits.

On the legal threats, where I did give you the understanding I made some ? If you have been confused I made some, my pleasure to dismiss them.

On the “process failure” I dislike, well it’s technical appreciation which is better to be discussed on the Bitcoin Core repository, where I give many code and wordly explanations of the limits or weakeness of v3 policy. It’s not like I pointed out years ago the limit of the “bitcoin-core-only-dev” discussions approach with the tx-relay workshops aiming to cross cognitive dissonance due to layers communication silos.

On the spending time building something better - If we all focus on shipping stuff rather than methodically demolishing bad ideas, we become a “Care Bears” ecosystem and few years from now we’re like other cryptocurrencies.

Realistically, if we wish a sane competition of sounds ideas and code, we have to admit the critical questioning, even if it hurts the ego of some. Feel free to propose a “lady’s and gentleman’s norms of discussions” (like done by public parliamentary assembly where opposite group of ideas are confronting each other), at least that make a clear boundary between “saying an idea is weak” and “saying this author idea is dumb”.

-------------------------

