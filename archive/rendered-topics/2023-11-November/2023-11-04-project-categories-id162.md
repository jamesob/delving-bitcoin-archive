# Project categories?

ajtowns | 2023-11-04 11:20:07 UTC | #1

What do people think about adding categories for bitcoin-related projects?

I've been thinking about adding one for [bitcoin-inquisition](https://github.com/bitcoin-inquisition/bitcoin/wiki) for announcements/discussions, and maybe it would be interesting for other projects that don't want to admin their own dedicated forum?

It's apparently possible to make a group for the project owners and give them permission to moderate posts in their category.

-------------------------

dgpv | 2023-11-14 10:18:04 UTC | #2

Maybe just add something like 'tools and libraries' category ?

-------------------------

dgpv | 2023-11-14 10:20:47 UTC | #3

Although with your example, bitcoin-inquisition, this might not work, as it is neither exactly a "tool" nor a "library". But maybe you could classify it as a "tool" for a "deep investigation of the value of proposed consensus changes"

-------------------------

ajtowns | 2023-11-17 04:48:42 UTC | #4

I like the term "working group" for a group that's focussed on a topic, particularly if it's probably not going to go on indefinitely, and in that vein there's now a #implementation:wg-blockrelay group/category for discussing expanding the number of block-relay-only connections bitcoin core peers make/accept cf [#28462](https://github.com/bitcoin/bitcoin/issues/28462).

Combining a category with a group makes it possible to:
 * make a private category, with posts only members of the group can see
 * make a read-only category, where only group members can post, but anyone can read the posts
 * make a public category, but allow group members to moderate it, rather than restricting that to the global moderators
 * make it so group members automatically watch the category, so they automatically get emails notifying them about new topics/comments in the category, but not get spammed with everything else going on on the site

-------------------------

ajtowns | 2023-11-30 08:22:42 UTC | #5

We've also had #implementation:wg-cluster-mempool going for a little while now; previously it was private to the group members; it's now public, but read-only if you're not a group member. Happy to setup variations of this for other groups, if there's interest.

-------------------------

josibake | 2024-05-17 10:32:15 UTC | #6

Was thinking `wg-silent-payments` might make sense? Currently having discussions in several different places about:

1. How to standardize indexes for light clients
2. PSBT support for sending/spending silent payment outputs
3. A silent payments descriptor

A lot of these discussions feel too early for draft BIPs/mailing list posts, hence a working group feels like a good fit?

-------------------------

ajtowns | 2024-05-17 10:59:05 UTC | #7

(Created #protocol-design:wg-silent-payments after discussing offline)

-------------------------

harding | 2024-05-27 17:01:33 UTC | #8

Is the goal for all `wg-` tags to indicate that a topic only accepts replies from allowlisted group members?

I just spent a bunch of time trying to understand a proposal tagged with a `wg-` and am frustrated that I can't reply unless I beg someone for permission.  I want to avoid wasting my time in the future, so I want to know if I should just read `wg-` tagged stuff as "Ignore this for now".

To be clear, I appreciate the idea of small group discussions being publicly archived for posterity.  Thank you to @ajtowns for setting that up and to everyone who is using that feature.   I just don't want to read stuff until I can do something with it.

-------------------------

ajtowns | 2024-05-27 19:28:18 UTC | #9

[quote="harding, post:8, topic:162"]
Is the goal for all `wg-` tags to indicate that a topic only accepts replies from allowlisted group members?
[/quote]

It varies per group.

[quote="harding, post:8, topic:162"]
I just spent a bunch of time trying to understand a proposal tagged with a `wg-` and am frustrated that I can’t reply unless I beg someone for permission.
[/quote]

The only group that currently has public read but not public comment/post is the silent payments working group, but as mentioned in its ["about" post](https://delvingbitcoin.org/t/about-the-wg-silent-payments-category/876), all you have to do is go to the [wg page](https://delvingbitcoin.org/g/wg-silent-payments) and click join, no begging for permission needed.

-------------------------

harding | 2024-05-28 02:53:33 UTC | #10

[quote="ajtowns, post:9, topic:162"]
It varies per group
[/quote]

Is there any way groups with different posting policies could be distinguished from each other?  Maybe `cwg-` for closed working groups ?  Alternatively a label like `closed` on any non-open discussions

I understand the benefits of closed working groups for developing ideas that aren't ready for public criticism yet, especially unthoughtful criticism from entrenched opposition---but I think it's important to label any closed discussion as such.  I don't want to see a discussion among experts and come to a mistaken belief that it represents a consensus opinion when critics didn't have an opportunity to reply.

Additionally, I think it's advantageous to members of a closed group to see that their posts are still protected from inline criticism from outside the group.  Otherwise I worry they too may come to a mistaken belief that they have discussed an idea sufficiently enough that it represents a consensus opinion.

[quote="ajtowns, post:9, topic:162"]
all you have to do is go to the [wg page](https://delvingbitcoin.org/g/wg-silent-payments) and click join,
[/quote]

If discussion of silent payments is meant to be open, why do I need to join rather than just reply like in #implementation:wg-blockrelay or have it just be a tag like for #wg-cluster-mempool::tag ?

-------------------------

