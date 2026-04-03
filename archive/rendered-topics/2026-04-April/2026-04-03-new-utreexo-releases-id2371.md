# New Utreexo releases

adiabat | 2026-04-03 15:13:20 UTC | #1

Announcing the release of Utreexod v0.5, with new Proofless IBD, powered by SwiftSync\[tm\] technology, and Floresta 0.9

Utreexod is based on btcd and is a full node which supports all utreexo node types:

* Archive: stores block proofs for IBD (though this mode will be later removed)
* Bridge: stores the entire Merkle forest and can create inclusion proofs for any valid transaction seen in it’s mempool
* CSN or Compact State Node: stores only the roots of the Merkle forest, and relies on inclusion proofs from peers to
  validate incoming transactions

Floresta is a minimal CSN implementation with focus on versatility. It can embedded into other applications or used as a standalone daemon. It comes with an integrated Electrum Personal Server that keeps track of your balance and can be used with any electrum-enabled wallet. By default it uses assumeutreexo to make the node ready-to-use in a few minutes, optionally backfilling old blocks. Floresta can be built as an embeddable library, allowing existing applications to ship it along with their own binary. This simplifies the process of integrating a wallet and node and can remove dependence on external servers for wallets currently using public APIs.

The big change in this release is that it uses techniques from SwiftSync (the cryptographic aggregator) to remove the need for nodes do download and verify accumulator inclusion proofs during IBD.  In the worst case scenario (with no caching) this reduces the extra data downloaded from 1.4TB to around 200GB (essentially the rev0000.dat files in bitcoin core’s blocks directory).  That rev data can also be cached, to be reduced further.

This new IBD strategy is a significant change and is not yet reflected in the utreexo BIPs (181, 182, 183).  Our goal is to test this new functionality out with utreexod and floresta before updating the specification in the BIPs to reflect the new “proofless” IBD.  After that utreexod will remove support for the old IBD strategy.

As there are still significant changes happening in utreexo, we’re not recommending people put all their coins into these wallets just yet.  But we are inviting developers to try out the wallet integrations, as those interfaces should be stable.  Let us know how integrating utreexo works with wallets, we’d love to hear feedback.

Utreexod [here](https://github.com/utreexo/utreexod) 

Floresta [here](https://github.com/getfloresta/floresta)

Special thanks to Vinteum for hosting the utreexo summit, and Ruben Somsen for the joining and helping figure out how to use SwiftSync and Utreexo together.

-------------------------

