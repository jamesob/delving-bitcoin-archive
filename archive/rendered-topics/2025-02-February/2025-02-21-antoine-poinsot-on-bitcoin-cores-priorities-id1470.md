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

