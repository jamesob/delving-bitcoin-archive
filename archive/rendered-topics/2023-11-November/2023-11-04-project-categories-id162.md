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

