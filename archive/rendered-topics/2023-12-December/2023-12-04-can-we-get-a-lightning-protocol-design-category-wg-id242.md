# Can we get a Lightning Protocol Design Category/WG?

ProofOfKeags | 2023-12-04 20:15:00 UTC | #1

Title says it all. The Lightning-Dev ML is considering moving over to doing work here. I don't know if it should be in the Protocol Design category as a working group or if it should be its own category.

-------------------------

ajtowns | 2023-12-04 20:25:41 UTC | #2

If you just want something public, posting to the existing categories (implementation / protocol design) and using the #lightning tag should be fine? My impression is the fedora folks found it better to go with general categories and more specific tags on their discourse, so I think that approach should be workable? Ref: https://discussion.fedoraproject.org/t/navigating-fedora-discussion-tags-categories-and-concepts/35555

If you want a more specific workgroup I can set that up too; but need to know who should be members, if it should be private, public, or read-only public.

-------------------------

ajtowns | 2023-12-04 20:30:10 UTC | #3

BTW, if you click the #lightning link you should see a little bell icon that you can use to "watch" a tag -- that'll get you emails for new posts that use that tag. I think that's the most useful way to follow a topic by email with discourse.

-------------------------

ProofOfKeags | 2023-12-04 20:49:48 UTC | #4

I think we can try out just using the lightning tag for the time being. Right now we are piloting moving discussion here with the retirement of the Lightning-Dev ML. It's possible that in the future we will find this is insufficient but I think it's a reasonable first approach. Thanks for the tip!

-------------------------

roasbeef | 2023-12-04 23:26:27 UTC | #5

[quote="ProofOfKeags, post:1, topic:242"]
The Lightning-Dev ML is considering moving over to doing work here. I don’t know if it should be in the
[/quote]

I think for the lightning-dev ML, we'll want to just make our own instance. I have the mailing list mode on this active now, but there doesn't seem to be a way to _just_ get emails from a given Category/Sub-Forum. Otherwise, I can't just get the LN-specific stuff over email as is w/ the way things are set up rn.

-------------------------

ajtowns | 2023-12-05 00:32:17 UTC | #6

Yes, mailing list mode is "everything on the site". If you want just a category/tag, turn mailing list mode off, "watch" the category/tag, and set your notification settings to email you even if you're online.

-------------------------

roasbeef | 2023-12-05 00:43:58 UTC | #7

Found a resource that explains how to use traditional email labels/filters to subscribe to just specific categories or even tags: https://discourse.mozilla.org/t/how-do-i-use-discourse-via-email/15279

-------------------------

ajtowns | 2023-12-05 02:47:32 UTC | #8

That's a 6.5 year old page last updated 3.5 years ago; there's been a bunch of threads since then essentially about replacing mailing list mode, eg https://meta.discourse.org/t/apply-mailing-list-mode-per-category/47772 with the result that [mailing list mode was disabled by default](https://github.com/discourse/discourse/pull/11091) in 2021. Current docs are [here](https://meta.discourse.org/t/discourse-new-user-guide/96331) and apparently [this](https://delvingbitcoin.org/my/preferences/emails) is the generic link to your email preferences.

(Note also that email notifications are delayed by a few minutes so that if a topic is edited immediately after posting, you get the edited version in email)

-------------------------

