# Should there be a "Network Data" category?

0xB10C | 2024-01-03 13:46:14 UTC | #1

What do you think about adding a category about Bitcoin (network) data with the primary goal of requesting/sharing/discussing data (unprocessed or processed as, e.g, graphs) related to Bitcoin development?

Personally, I think moving requests like the following to a forum where I can reply with images and multiple people can join in and discuss would be beneficial.
- @AntoineP on IRC: "do you have data about the percentage of fees in the block reward in the past years by any chance? Or anyone else of course"
- @ajtowns on IRC: "did you see tx vbyte volume in september comparable to https://mempool.space/graphs/mempool#6m ? (or anyone else ofc)"
- @instagibbs on X: https://twitter.com/theinstagibbs/status/1742285787494748254
- ...

-------------------------

AntoineP | 2024-01-03 13:51:41 UTC | #2

That sounds reasonable to me. I know a few other people have an interest in data collection and processing. Regrouping discussions and requests here would be beneficial.

-------------------------

cdecker | 2024-01-04 12:31:59 UTC | #3

Definitely a good idea, though we could just get started with a single post, and if there is enough traffic we can give it a dedicated category.

Fwiw I have 11 years of network propagation data (TX and Block hashes and which IP saw them at which timestamp). Ping me if anybody is interested in either historical orphan rates and/or historical mempool reconstruction, and I'll provide access to it :-)

-------------------------

ajtowns | 2024-01-05 07:38:57 UTC | #4

Maybe "Measurements" or "Observations" would be more general?

Or could just put it in "Protocol Design" on the basis that knowing more about how the network behaves will help us improve the design of protocols?

If we have detailed benchmarks/performance analysis of IBD sync or secp256k1 signing or similar, would that be better in "Implementation" (in that it would help improve particular implementations) or would it be better to have it nearby other data analysis work?

-------------------------

AntoineP | 2024-01-05 23:47:44 UTC | #5

Having benchmarks in "implementation" sounds reasonable as it's how it happens for instance on Github. But it seems plausible some performance analysis could be contributed that are not directly related to an implementation.

If the criterion for "Protocol Design" is anything which helps making decision about how to design the protocol, the whole forum would fall under this category. :p

A "Measurements" category (no opinion on the name) seems more appropriate to me. For benchmarks best judgement can be used between posting it next to an implementation discussion or contributing it in the broader "Measurements" category.

-------------------------

