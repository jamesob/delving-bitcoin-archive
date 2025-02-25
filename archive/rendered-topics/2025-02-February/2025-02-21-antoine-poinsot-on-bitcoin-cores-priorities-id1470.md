# Antoine Poinsot on Bitcoin Core's Priorities

ajtowns | 2025-02-21 16:47:25 UTC | #1

@AntoineP wrote three interesting blog posts on Bitcoin Core:

 * https://antoinep.com/posts/core_project_direction/
   * *"I think we are in this situation where everybody is working on their own thing and there is no broad agreement about what the scope of the project should even be. As a consequence, year after year we keep piling up more code, more features, more RPCs, etc.. with at best a constant amount of competent reviewers’ time. This is not tenable, as a focus which gets further dispersed will inevitably lead to reduced overall software quality."*
 * https://antoinep.com/posts/stating_the_obvious/
   * *"When you consider that Bitcoin Core’s userbase is indirectly all Bitcoin users, how to prioritise development becomes quite clear. Bitcoin Core should be a robust backbone for the Bitcoin network, balancing between securing the Bitcoin Core software and implementing new features to strengthen and improve the Bitcoin network."*
 * https://antoinep.com/posts/bitcoin_core_scope/
   * *"The multiprocess project presents an opportunity in this regard. Separate `bitcoin-node`, `bitcoin-wallet` and `bitcoin-gui` binaries communicating through well-defined interfaces open the door to splitting the current project into 3. The contributors at [github.com/bitcoin/bitcoin](https://github.com/bitcoin/bitcoin) would focus on the Bitcoin Core node and release a `bitcoin-node` program, exposing JSONRPC and IPC interfaces. The contributors at [github.com/bitcoin-core/wallet](https://antoinep.com/posts/bitcoin_core_scope/github.com/bitcoin-core/wallet) would work on the `bitcoin-wallet` process. The Bitcoin Core wallet releases would contain both the wallet and node processes, presumably with a wrapper to make it transparent to users. This project would release the equivalent of today’s `bitcoind`. Finally, the contributors at [github.com/bitcoin-core/gui](https://github.com/bitcoin-core/gui) would develop and release the `bitcoin-gui` program, which would ship with the node and wallet."*

-------------------------

AntoineP | 2025-02-21 18:10:50 UTC | #2

Next thing to figure out would be how to handle the shared code and configuration. What i had in mind was that the wallet would have the node as a subtree, likewise the GUI with the wallet. There is some common code that could be mostly split (`src/crypto`) but other (`src/util`, `src/common`, ..) will have to either be duplicate or become another interface between the projects.

Copying the configuration (the CI for instance) seems less concerning. Then the other big thing to figure out (which is a blind spot for me) is how we would organize the build system. cc @fanquake maybe you can clue me in.

-------------------------

ajtowns | 2025-02-21 20:00:22 UTC | #3

I mostly made this topic so I could reply here rather than writing up a blog entry of my own. :) So...

A few thoughts:

 * I'm not convinced multiprocess is the right way to split things out of core in any meaningful way -- as far as I understand it, multiprocess is intended to leave its affected components [fairly tightly coupled](https://github.com/ryanofsky/bitcoin/blob/pr/ipc/doc/design/multiprocess.md#interface-stability), so changes to core or the gui or the wallet would likely affect the multiprocess api, and be annoying to coordinate/sync across repos. I think multiprocess/capnproto makes more sense for process separation for security reasons, than for development separation for prioritisation reasons, but could be convinced otherwise.
 * Having all our wallet features be available via a library (or command-line tool, like bitcoin-tx or bitcoin-wallet) that you can use without having to fire up a bitcoind to get at the RPC interface would be a win, I think, independent of anything else.
 * One approach I like is [ketan's NodeBox](https://shop.ministryofnodes.com/shop/bitcoin-nodebox/) which splits the bitcoin software into two parts: a node, index and blockchain explorer that don't retain any personally identifiable information; and the wallet software that does. That puts core, electrs, and mempool.space as the "node", and sparrow installed on a different device as the wallet, which is a somewhat similar split up to what Antoine considers above. One big difference is the inclusion of electrs for indexing transactions, which is useful both for the mempool.space explorer, and to allow a wallet that's offline most of the time to quickly sync. BDK also [seems to assume](https://bitcoindevkit.github.io/book-of-bdk/cookbook/syncing/electrum/) having access to indexes like that is a good idea. That's something core has traditionally resisted (last attempt was [PR#14053](https://github.com/bitcoin/bitcoin/pull/14053) I think), and I think that resistance perhaps makes it a bit harder to do a nice UI, especially if you want to separate your PII away from your node software, but still want very quick syncs when you open your wallet.
 * Personally, I like the idea that bitcoin core aims to provide enough software to make Bitcoin usable; so a GUI and a working wallet are an important part of that. How that's architected isn't so important to me, and of course, bitcoin core probably doesn't quite achieve that today anyway, in that these days it's missing necessary components on the mining side of things. Having wallet functionality also seems useful to have in doing functional tests, and experimenting with bitcoin more broadly (eg, messing around on signet).

-------------------------

ariard | 2025-02-22 00:14:24 UTC | #4

"*When you consider that Bitcoin Core’s userbase is indirectly all Bitcoin users, how to prioritise development becomes quite clear.* “ Excerpt - https://antoinep.com/posts/stating_the_obvious/

Who is a (average) Bitcoin Core’s user ? This is a well-informed series of posts, though it’s very centered on contributors, maintainers, dev process, etc without really asking for the obvious. 

What is a (average) bitcoin core user ? What level of know-how they should have to run `bitcoind` ? Are they more often on Windaube or a *NIX ? Do they care about running `bitcoind` to verify their stack the cypherpunk way or just because to contemplate the well-architected `bitcoind` burning CPU time ? Do they prefer a browser-like exp or are they die-hard CLI galts and guys ? What’s their level of engagement w.r.t Bitcoin, are they going to their local meetups to hear the latest about consensus changes or do they just care fiat-vs-btc NGU price ? How often are they using their `bitcoin-wallet`, a week, a month, a year, etc ?

(— yes it’s asked in the maieutic way, no right or wrong answers).

-------------------------

jsarenik | 2025-02-23 08:37:34 UTC | #5

Let me share one, not caring how much average or not:

There is a person who runs Bitcoin Core nodes, re-reads Satoshi's whitepaper every few months and is totally satisfied with v28.1. All the ideas he has need not changes in Bitcoin Core as much as his time to observe how it works and contemplate upon possible improvements in his domain, without changing the code of Bitcoin Core but rather his local "code" i.e. scripts that make use of generally available Bitcoin Core releases. See [alt.signetfaucet.com](https://alt.signetfaucet.com) which is a living result of all the ideas he had since 2017 (Linux-only nocoiner until then). He does not go to meetups much although he likes to hear from friends who attend them.

He has questions like "Will my hand-crafted tx spending unkeyed LN Anchor all on fees get mined even though it does not have enough additional fees to replace the previous in-mempool transaction (which is sending sats to a keyed address) in most of Bitcoin Core nodes' mempools?" so for the sake of experiment he does prioritize the hand-crafted tx in mempools of his three nodes, waits a few days and viola, it got [mined](https://mempool.space/tx/ad1507f186ccdd448cc58a5ed68367249de6ad55d7f4b70bb3fcc5c88a85c626?mode=details) on mainnet. He wonders "Does RBF work multiple-times?" and it [does](https://mempool.space/signet/tx/6283defdc43b1fead93c851a8d11d7d79d796d8d1596216bae7b213e9f49f5aa?mode=details) - proven on signet. And so on and so forth.

> As for me and my household we will run Bitcoin Core nodes. And we enjoy its development process and everyone involved (incl. @ajtowns and @ariard from this post). Also including, though not mentioning here, others - it would be a long list and someone might be missing still. And even those silent Bitcoiners and Twitter randos. We teach children of all kinds: wise, wicked, simple and the ones who do not know how to ask - as far as they are interested and curious about Bitcoin.

-------------------------

bruno | 2025-02-25 13:20:23 UTC | #6

> For instance [`rust-bitcoin`](https://github.com/rust-bitcoin) has never made so much progress, and [BDK](https://github.com/bitcoindevkit/bdk) is a major success [adopted by dozens of projects](https://bitcoindevkit.org/adoption/all) with probably millions of users altogether. They have a larger feature set, with the Bitcoin Core wallet still struggling to catch up even years after a feature has been largely adopted by the industry[1](https://antoinep.com/posts/bitcoin_core_scope/#fn:1).

This is interesting. What makes BDK evolves so fast compared to Bitcoin Core? The language (I see more people interested in rust than C++ and new comers prefer to contribute to rust projects. Also, there are real projects really using it)? The way they manage the project (I kinda like the way the organize the things, the discord channel, meetings, etc)?

-------------------------

AntoineP | 2025-02-25 14:40:45 UTC | #7

[quote="bruno, post:6, topic:1470"]
What makes BDK evolves so fast compared to Bitcoin Core?
[/quote]

I think language indeed plays a role. I'm less sure about project organisation (Discord, meetings, ..). It's also the case that it's more fun to "rewrite X from scratch" than work on a project with a bunch of legacy code. Having a clear set of goals for what they are trying to achieve surely helps maintain focus, too. It also solves the coordination problem to be able to actually make progress (similar to the working groups idea for Core, "if i invest 1 month implementing and polishing this, has it decent chances of making it in?") which certainly makes it more fun to work on the project. Finally, i think the bar is generally higher to get changes in the Bitcoin Core wallet than in BDK. Being more fun with a general direction and ability to make progress in turn helps bring and retain contributors.

Splitting the wallet into its own repo won't automagically bring all of these benefits but i do think there is an opportunity to get some of them. For instance lowering the bar for review temporarily would adjust to the current reality of the number of contributors there and make it easier to make progress. Having it in its own project would let contributors focus exclusively on the wallet, and coupled with [other factors](https://lclhost.org) could let contributors coordinate on a clear set of goals for the project to advance toward.

-------------------------

