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

Laz1m0v | 2026-04-03 21:51:48 UTC | #2

Congratulations on the release :sunflower:  SwiftSync and proofless IBD are impressive milestones.

I wanted to ask a question about a use case we are currently exploring: **running Utreexo inside a Trusted Execution Environment.**

We are building an autonomous Bitcoin wallet architecture ([PRECOP](https://github.com/BitcoinWorldTrustFoundation/precop)) where an AWS Nitro Enclave runs a Simplicity Bit Machine to enforce spending constraints before producing BIP-341 Schnorr signatures. Because TEEs have very tight constraints (\~256MB RAM, no persistent storage, stateless restarts), running a full node inside the enclave is not practical.

Today the enclave receives PSBTs from an external bridge process and recomputes the sighash internally after overriding `witness_utxo`. However the enclave still cannot independently verify that the input UTXOs actually exist in the current UTXO set.

Utreexo seems like a clean solution here. The idea would be:

* The enclave keeps only the Utreexo forest roots in memory (a few KB, updated with each block header)

* The bridge provides inclusion proofs for every PSBT input

* The enclave verifies the proofs against its local roots before running the Simplicity policy engine and signing

This would allow the enclave to verify UTXO existence with something close to full-node assurance while staying within a small memory footprint.

**My question:** is the `rustreexo` accumulator library stable enough to embed as a crate in a constrained environment?

Specifically we would only need the `verify` and `update` roots paths (not proving or deletion). Are those components well-tested independently of the full utreexod / Floresta node implementations?

Utreexo looks like a very natural primitive for trust-minimized signing environments.

-------------------------

