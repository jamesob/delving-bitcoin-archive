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

moonsettler | 2025-02-25 15:48:22 UTC | #8

[quote="bruno, post:6, topic:1470"]
What makes BDK evolves so fast compared to Bitcoin Core?
[/quote]

The bitcoin core codebase is notoriously horrible to interact with. It's not just the language, but also how you use it.

-------------------------

AntoineP | 2025-02-25 19:19:21 UTC | #9

[quote="ajtowns, post:3, topic:1470"]
I’m not convinced multiprocess is the right way
[/quote]

As i shared in my comment above I don't think a split around multiprocess binaries is ideal. But i believe it is important to refocus the main project on the node parts of the software, and i just see such a split as the only realistic and socially acceptable way of achieving that. Do you have a better suggestion?

[quote="ajtowns, post:3, topic:1470"]
Having all our wallet features be available via a library (or command-line tool, like bitcoin-tx or bitcoin-wallet)
[/quote]

I agree that raw transaction parsing, PSBT updating, script decoding, signing, etc.. should not be part of the node RPC since it does not required a node in the first place. But yes it's tangential.

[quote="ajtowns, post:3, topic:1470"]
BDK also [seems to assume](https://bitcoindevkit.github.io/book-of-bdk/cookbook/syncing/electrum/) having access to indexes like that is a good idea. That’s something core has traditionally resisted
[/quote]

BDK does not assume that anymore. It now just indexes wallet transactions and has a wallet-spk-to-derivation-index index just like the Core wallet. It has a [bitcoind RPC chain data source](https://github.com/bitcoindevkit/bdk/tree/master/crates/bitcoind_rpc) as well.

[quote="ajtowns, post:3, topic:1470"]
Personally, I like the idea that bitcoin core aims to provide enough software to make Bitcoin usable
[/quote]

Me too. Unfortunately with the current resources it comes at non-trivial cost for all Bitcoin users, who are indirectly using Core, for the benefit of a handful of direct Core GUI users.

[quote="ajtowns, post:3, topic:1470"]
Having wallet functionality also seems useful to have in doing functional tests
[/quote]

We have `MiniWallet` for this?

[quote="ajtowns, post:3, topic:1470"]
and experimenting with bitcoin more broadly (eg, messing around on signet).
[/quote]

In a split future this could just be done using the Core wallet project which would release the equivalent of today's `bitcoind`? Or just any other wallet that would bundle a `bitcoin-node` (Sparrow, Liana, ..).

-------------------------

harding | 2025-02-26 08:41:17 UTC | #10

Thanks @AntoineP for thinking deeply about this subject and @ajtowns for relaying.  I'm not sure Antoine's "obvious" is obvious to me:

> Bitcoin Core’s userbase is indirectly all Bitcoin users, [so] how to prioritise development becomes quite clear. Bitcoin Core should be a robust backbone for the Bitcoin network

I think it's weird and non-obvious to define your users as including people who don't directly use your software.  My ISP presumably uses a bunch of software to manage the network that connects my home to the Internet.  Providing connectivity to me, and thousands of customers like me, is the ultimate purpose of that software---but its user is the ISP.  That distinction is important because when that software forces a choice between tradeoffs onto my ISP, they _as the user_ will make the choice that's best for them and not necessarily for me.

Similarly, if an industrial full node operator who serves data to customers is forced to choose, they will make the choice that's best for them and not necessarily for their customers.

The distinction between user and customer would be petty pedantry except for one thing: even a short lapse in enforcement of Bitcoin's consensus rules can practically change them forever.  Imagine if the operators of the predominant share of economic full nodes decided to allow a transaction spending an extra 1M BTC into the chain, and then quickly fanned out that money by opening new LN channels and forwarding the funds to channels they established before the fraud block (or using same-chain or cross-chain coinswaps, or buying non-revocable goods and services, etc...).  After 3 to 6 confirmations (30 to 60 minutes on average), performing a reorg to put those fake Bitcoins back into Pandora's box would mainly destroy the wealth of honest users while leaving the attackers enriched.  In that case, I think we as community might choose instead to accept a new limit of 22M BTC.

A hour isn't enough time for Bitcoiners to respond to such an attack, so the only practical mitigation is prevention: we need large numbers of Bitcoiners to run economic full nodes.  That is, they need to be direct users of Bitcoin Core **and** have their wallet connected to their node.  That will ensure that they will reject any attempt to open a channel with them using fake bitcoins (and the other attacks mentioned), directly protecting their wealth and indirectly protecting the wealth of anyone who doesn't run a full node by making the attack less viable.

In that sense, building a secure node isn't just about eliminating bugs and vetting protocols but also about ensuring large numbers of people use their own copy of the node to verify their own incoming transactions.  That requires a wallet and the means for non-experts to use it (a GUI).  Certainly, those don't need to be bundled with Bitcoin Core, but if the developers of the Bitcoin Core project cease to put effort into making a node-backed wallet accessible, why would we expect anyone else outside the project to go through the frustrating process of trying to get PRs merged into Bitcoin Core that will provide them with the interfaces they need?  Anyone who has tried to do that in the past has received a litany of "noes": no, we're not going to add another index; no, we're not going to extend the P2P interface to provide unverifiable info; no, we're not going to keep SSL to allow remote RPC access; etc...

Will those noes suddenly become yeses after a multiprocess-based repo split?  Or will there be even more reason to say "no" to wallet-supporting features in the node because all of the focus is on optimal verification and relay?  The idea of a node being embedded in wallet software is mentioned above, but is that realistic both from the standpoint of software development (e.g., will every wallet that wants to do that have to write thousands of lines of code to deal with Bitcoin Core's limited interfaces) and from actual usage (e.g., I don't want to run an always-on node on my mobile device; I'd rather run a node at home and connect to it as needed from my mobile)?

In considering priorities for the future of Bitcoin Core, I would ask contributors to ask where those priorities will lead the project in five or ten years.  Will it be a greater percentage of Bitcoin users validating their transactions with their own full node, or will it be a greater percentage of users entrusting verification to industry?

-------------------------

adrienlacombe | 2025-02-28 05:44:49 UTC | #11

"For technical and historical reasons, Bitcoin Core is the only reasonable full node implementation for all Bitcoin users to run. Re-implementations of Bitcoin face significant technical challenges and appear to be doomed to have consensus bugs." from [Antoine Poinsot - Stating the obvious](https://antoinep.com/posts/stating_the_obvious/)

@AntoineP do you have an idea of what work it would take for Core to allow for other clients? I imagine such change would required a hard fork anyway?
Merci

-------------------------

