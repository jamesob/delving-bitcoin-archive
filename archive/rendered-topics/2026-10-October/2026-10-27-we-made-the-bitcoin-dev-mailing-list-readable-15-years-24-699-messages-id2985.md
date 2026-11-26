# We made the bitcoin-dev mailing list readable: 15 years, 24,699 messages

LearnBitcoin.com | 2026-10-27 14:06:15 UTC | #1

We have put up a readable mirror of the bitcoin-dev archive, refreshed daily.

What it adds over public-inbox: one page per thread with quotes folded, stable permalinks by Message-ID, and three indexes over the same data: topic timelines, a page per BIP discussed, a page per author. Messages are verbatim; the only words of ours are the topic blurbs. Every message links to its original.

Two things we would like to get right and could use your eyes on. First, name merging: spellings one address used are folded into one page (“Luke-Jr” and “Luke Dashjr”), never through list or group addresses; if a merge is wrong or you would rather not have a page, say so and it is done. Second, threading: References first, a subject match within 180 days for the few hundred replies that lack them; mis-threaded threads are easy to fix if you point at one.

If it is useful, we would be glad to be listed among the public archives of the list. If any of it is unwelcome, say that too.

[https://www.learnbitcoin.com/archive](https://www.learnbitcoin.com/archive)

-------------------------

AdamISZ | 2026-10-30 03:50:20 UTC | #2

Looks like a very useful resource, thanks for the work!

Is there a way to browse it directly by date?

-------------------------

LearnBitcoin.com | 2026-10-30 13:40:19 UTC | #3

Yes, by year and month. The archive’s section nav now has a Years entry, which lists every year since 2011 with its counts; each year page, for example <https://www.learnbitcoin.com/archive/2015>, lists the threads started that year grouped by month in order of first message, with a row of month links at the top to jump without scrolling. The month headings are anchors too, so <https://www.learnbitcoin.com/archive/2015#m-6> goes straight to June 2015. The home page’s “Recently active” list is the other way in, newest message first.

The nav entry used to say “Numbers”, which hid that it was the way in by date; renamed today, and the month bar is new as well, both because you asked, thank you.

-------------------------

LearnBitcoin.com | 2026-11-09 22:48:59 UTC | #4

A few things added since the first post, the main one being where each BIP landed in Bitcoin Core.

Every BIP page now has an “In Bitcoin Core” box: the version that first shipped it, the later versions that touched it, and the note from Core’s doc/bips.md word for word, pull request links included. For example https://www.learnbitcoin.com/archive/bips/141 shows the 48 list threads on BIP 141 next to “0.13.0, defined for mainnet as of 0.13.1, buried since 0.19.0”. That covers the 74 BIPs doc/bips.md lists.

There is also a page per Core release, 137 of them from 0.1.5 on: https://www.learnbitcoin.com/archive/releases . Each has its date, the list threads that named it (release candidates and the announcement), the BIPs it first shipped, and links to the notes. One thing that fell out of building it: the last release announcement we can find on the list is 24.0.1, December 2022.

The mapping is only as good as doc/bips.md. If you know of a BIP Core implements that the file leaves out, or a version it gets wrong, fixing it upstream fixes it here on the next refresh. And if a release thread is matched to the wrong version, please let me know.

Smaller things: the archive now refreshes twice a day, and seven people whose mail was split across two addresses have been merged into one page each.

-------------------------

alex | 2026-11-18 22:29:31 UTC | #5

Any chance you could do the Segwit2x mailing list as well? It's my favorite piece of Bitcoin history.

-------------------------

LearnBitcoin.com | 2026-11-20 15:26:54 UTC | #6

Yes. Good timing too, because that list nearly disappeared: the Linux Foundation took its archive down, and nobody mirrored it the way bitcoin-dev was mirrored. It survives in the Wayback Machine as the monthly files, May to December 2017, a bit under 700 messages.

I'm looking at adding it as its own list since it’s small and uses the same archive format as lightning-dev.

-------------------------

LearnBitcoin.com | 2026-11-26 12:55:59 UTC | #7

It's up: https://www.learnbitcoin.com/archive/lists/segwit2x

722 messages in 75 threads, May 31 to December 1, 2017. (My earlier post said a bit under 700. I had undercounted July.)

The Linux Foundation's copy is gone and I found no other mirror, so this was rebuilt from what the Wayback Machine kept. The monthly archive files gave the headers, which is what makes threading and permalinks by Message-ID work. The text comes from each message's own archived page, because the monthly files had replaced every non-ASCII character with a question mark. Every message links to its Wayback capture.

Same treatment as bitcoin-dev: one page per thread, messages as sent, no summaries. It shows up in the archive search too.

Two things came with it. Anyone who wrote to both lists has one people page covering both, matched by email address and never by name. And a link inside a message to one of this list's old lists.linuxfoundation.org pages now opens that message here, on both lists. Those addresses forward to gnusha's redirector, which had no entry for this list when I checked, so the links had been dead.

If anyone kept an mbox of the list, I would like to check this copy against it.

Next is lightning-dev. It was a Linux Foundation list in the same archive format, so the importer built for this one already reads it. It's about six times the size, roughly 4,000 messages from 2015 to 2023, so no date yet. If anyone has an mbox or a tarball of it, that would make it faster and more exact than rebuilding from page captures. When it's up I'll post it in its own thread rather than here.

-------------------------

