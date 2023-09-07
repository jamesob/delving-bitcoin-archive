# Public archive for Delving Bitcoin

jamesob | 2023-09-05 17:42:56 UTC | #1

It would be great to have a public archive of the posts here, so that
1. if this site goes away, we still have all the content, and
1. it is searchable by indexers like https://bitcoinsearch.xyz.

As far as I can tell, there are three approaches:

## Scraping

Use some kind of crawler (selenium, wget, etc.) to manually create a backup by scraping the site.

- I think this is not great given how JavaScript heavy Discourse is (i.e. it loads posts incrementally based on cursor position). It's also brittle - scraping code is often complicated.
 - ...but, it can be done by anyone, with no permissions necessary.

## API 

Use the [Discourse API](https://docs.discourse.org/) to pull content. This could be done either periodically as a full crawl or incrementally on a continuous basis.
- I think this is a fine way to go assuming the API offers everything we need and someone can provision API credentials.
 - We may be able to get everything we need out of the [list posts endpoint](https://docs.discourse.org/#tag/Posts/operation/listPosts).

## DB dump

Get a SQL dump of the database and run some kind of sanitization/export script on it every so often.
- The upside: easy to be complete and the schema is probably very stable.
- The downside: this can only be performed by administrators.

---

I'd say we should either go with API or DB, and then post the results on a git repo somewhere public (preferably a few places!).

Thoughts from the admins?

-------------------------

ajtowns | 2023-09-05 18:15:58 UTC | #2

Scraping seems to be what upstream recommends:

 * https://meta.discourse.org/t/archive-an-old-forum-in-place-to-start-a-new-discourse-forum/13433
 * https://meta.discourse.org/t/improving-discourse-static-html-archive/112497
 * https://meta.discourse.org/t/a-basic-discourse-archival-tool/62614
 * https://meta.discourse.org/t/how-do-i-export-the-complete-forum-as-static-html-pages/71007/3

There's also the `/raw/` endpoint: https://delvingbitcoin.org/raw/87 and the api, eg https://delvingbitcoin.org/posts.json

There's also https://meta.discourse.org/t/discourse-public-data-dump/264748 because of course exporting data for AI training is a much higher priority than just information accessibility and continuity...

-------------------------

jamesob | 2023-09-05 18:45:20 UTC | #3

[quote="ajtowns, post:2, topic:87"]
There’s also the `/raw/` endpoint: https://delvingbitcoin.org/raw/87 and the api, eg [https://delvingbitcoin.org/posts.json ](https://delvingbitcoin.org/posts.json)
[/quote]

Oh, that's great! I think these two might give us everything we need?

-------------------------

ajtowns | 2023-09-06 02:13:06 UTC | #4

Might need to do something extra to also get any uploaded images / attachments?

Also need to be prepared to add `?page=2` etc if there's more than 100 comments, eg https://meta.discourse.org/raw/69776?page=8. I think the API is rate limited kinda heavily by default, https://meta.discourse.org/t/available-settings-for-global-rate-limits-and-throttling/78612 so may need some tweaking.

-------------------------

midnight | 2023-09-06 17:52:29 UTC | #5

Having sat there and considered significantly the longterm utility of logs and access to information that Bitcoin requires to defend itself—literally transparency is one of its best and most-resulted-in-rescues defences—I would like to also recommend that you make the archival and thus witnessing something that more than one person or process can perform.

On IRC, people can create and maintain logs longer-term on a per-person basis. This creates a much more robust and participation-based consensus on what constitutes the historical record.

I have found that this is an important facet of archival effectiveness as well. There are on occasion some attacker-injection problems that can be a problem for the safety of individuals, but I have also found that the most diligent and reliable sources of archival information are also the most reasonable and realistically practical people as well—so this tends to be an addressable problem.

Thus, may I suggest that the archival process itself be made available to individuals who are interested in participating. :-)

Further, now that I'm thinking about it, I would like to point out that for those forums like Slack where the relevant historical archive is spotty, questionable, inaccessible, or otherwise opaque, these places are the sources of *significant* and *ongoing* attacker fuel—a simple propaganda-only example would be the ongoing nonsense about the "dragon's den."

-------------------------

