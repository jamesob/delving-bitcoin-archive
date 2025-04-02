# Signet faucet using 25-long-tx-chain & RBF

jsarenik | 2025-04-02 05:26:46 UTC | #1

The same Signet faucet is now available at

* https://alt.signetfaucet.com
* https://signetfaucet.bublina.eu.org
* https://signet25.bublina.eu.org

Its git repository resides at https://github.com/jsarenik/bitcoin-faucet-shell since Nov 2021 but the recent 25-long-tx-chain and RBF features were implemented in the beginning of 2025 along with added usage of Cloudflare Turnstile which significantly impoved humanness of the faucet.

The faucet pays just the minimal increment of sats (`vsize`) for its fee to replace the previous transaction whenever possible or matches fee-rate when adding minimal increment is not enough. It even [shows on mempool.space](https://mempool.space/signet/address/tb1p4tp4l6glyr2gs94neqcpr5gha7344nfyznfkc8szkreflscsdkgqsdent4) like if the new sat/vB fee-rate was the same as in the just-replaced transaction.

TRUC (V3 transaction) was originally used to help with getting into block cleanly without any possibility of lowering the overall fee-rate by CPFP (or rather "CRFP" for Child Receives From Parent?). Back to using V2 transactions now (March 2025) where a chain of 25 is in the mempool and replacements are done on its last leaf.

The requests stay between blocks until they are served - they are just miniature files containing the `tx_out.script length` followed by `tx_out.script` in hex encoded as ASCII - example for `tb1pfees9rn5nz` (LN anchor on testnets) is `04 51024e73`. This LN anchor address is always available to test the faucet with only 1-per-block limit ~~in addition to the always-present 240 sats big output~~. Beware that whatever comes to that address can be spent by anyone and it is very easy to setup a read-only and blank Bitcoin Core wallet, import the descriptor just by address, set a `walletnotify` script and then just "wait for the mouse".

I was dreaming of such a faucet since 2018 when I started learning about Bitcoin. Thanks to Kalewoof and @ajtowns for ideas and signet coins!


# Environment details

The two HTML files are static (i.e. there is no possibility of 502 Errors or likes, which was the reason I started working on this faucet in November 2021) and served by Caddy2 webserver which gets proxied by Cloudflare. It all runs from home on a 50/10 Mbps fiber optic line (the cheapest available at my place - and it is paid for anyway so this faucet has no influence). All (including the main OpenWRT router) is Linux and the faucet extensively uses the `tmpfs` (i.e. writing temporary files which reside in RAM). All runs on a recycled Mac Book Pro '12 which I bought myself in BestBuy near San Mateo (California).

I get a new IPv4 (only) address upon restarting the fiber optic Huawei device. Soon after a script notices the changed IPv4 address the DNS record gets updated and everything works.

I run multiple Bitcoin Core nodes here. One on the main server (the MacBook) and one in Termux running on latest and unrooted Motorola edge 2022 that I got for free from a friend. All pruned but with current mempool. They produce a sound of wooden block when a new block on mainnet gets verified (`blocknotify=script` in `bitcoin.conf`). The same sound as [here](https://display.anyone.eu.org/price.html).

Thank you for reading this far.

-------------------------

jsarenik | 2025-04-01 09:03:48 UTC | #2

Hoping the clustermempool would still live nice together with signet25 - see https://delvingbitcoin.org/t/post-clustermempool-package-rbf-per-chunk-processing/190/10?u=jsarenik 😇

-------------------------

