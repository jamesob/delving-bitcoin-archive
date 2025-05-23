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

ajtowns | 2024-02-28 11:14:56 UTC | #6

[quote="ajtowns, post:1, topic:598"]
to make it easier for future historians to still find those discussions, there’s now a #wg-cluster-mempool::tag tag that’s retroactively applied to all them, and that is locked to admins, so it can’t be added to future discussions
[/quote]

Apparently the #wg-cluster-mempool::tag link was broken for everyone but moderators; should be working now.

-------------------------

MentalNomad | 2024-03-14 19:03:52 UTC | #7

[quote="ariard, post:2, topic:598"]
Do you have logs and list of members of such private wg-cluster-mempool ?
[/quote]

You didn't seem to comprehend @ajtowns 's post nor how Discourse works.

He said the posts and comments made within the group's private Category are now public. They retain their original timestamps, edit histories, and commenter attributions.

Discourse makes it trivially easy to move a conversation from private to public while retaining the full audit history. No one need review logs to find what some secret cabal did... It was just a working group that got to focus on something without public interruption and distraction until they had developed things to a suitable level to present to others for further review and comment.

-------------------------

MentalNomad | 2024-03-14 19:10:59 UTC | #8

[quote="ajtowns, post:3, topic:598"]
(However, this site does not do end-to-end encryption of messages, so it’s should not be used for discussion of zero-day vulnerabilities whether via restricted groups or private messages)
[/quote]

Note: the [Discourse Encrypt](https://meta.discourse.org/t/discourse-encrypt-for-private-messages/107918) plugin is an option. It is an imperfect encryption approach which doesn't hide all metadata and has imperfect privacy, but it effectively encrypts contents of messages for the parties in a discussion.

-------------------------

kouloumos | 2025-01-13 08:47:28 UTC | #10

With the recent shift to the [Working Groups model](https://github.com/bitcoin-core/bitcoin-devwiki/wiki/Working-Groups) for implementing and reviewing projects within Bitcoin Core, it may be worth reconsidering the lifecycle of working groups in this forum.

For example, while the #implementation:wg-cluster-mempool group has closed here, it still exists within the Working Groups structure of Bitcoin Core, reflecting a change in how projects are organized. Having a central reference point for each group could help interested parties follow progress more easily and lower the barrier to entry.

-------------------------

