# Signet faucet using TRUC and RBF

jsarenik | 2025-02-06 16:08:22 UTC | #1

The same faucet is now available at

* https://alt.signetfaucet.com
* https://signetfaucet.bublina.eu.org
* https://signet25.bublina.eu.org

It pays most of the time just the minimal increment of sats (`vsize`). Sometimes it even shows on Mempool.space like if the new sat/vB fee-rate was just `0.9` more than previous.

TRUC helps with getting into block cleanly without any possibility of lowering the overall fee-rate by CPFP (or rather "CSFP" for Child Steals From Parent?).

I am not emulating mempool so if there would be too many (more than 230, but depends on the tx_out lengths of scripts/addresses used) requests it would just not replace the current TRUC payout transaction. The requests stay between blocks as they are just miniature files containing the `tx_out.script length` followed bt `tx_out.script` in hex encoded as ASCII - example for `tb1pfees9rn5nz` (LN anchor on testnets) is `04 51024e73`. This LN anchor address is always available to test the faucet with only 1-per-block limit. Beware that whatever comes to that address can be spent by anyone and it is very easy to setup a read-only and blank Bitcoin Core wallet, import the descriptor just by address, set a `walletnotify` script and then just "wait for the mouse".

I was dreaming about such a faucet since 2018 when I started learning about Bitcoin. Thanks to Kalewoof and @ajtowns for ideas and signet coins!


# Environment details

The two HTML files are static (i.e. there is no possibility of 502 Errors or likes, which was the reason I started working on this faucet in November 2021) and served by Caddy2 webserver which gets proxied by Cloudflare. It all runs from home on a 50/10 Mbps fiber optic line (the cheapest available at my place - only 12.90 EUR per month). All (including the main OpenWRT router) is Linux and the faucet extensively uses the `tmpfs` (i.e. writing temporary files which reside in RAM). All runs on a recycled Mac Book Pro '12 which I bought myself in BestBuy near San Mateo (CA).

I get a new IPv4 (only) address on restarting the fiber optic Huawei device. Soon after a regular script notices the DNS record gets updated with the current IPv4 address.

I run three Bitcoin nodes here. One on the main server (MacBook), one on the laptop I write this from (which also runs some Blockstream Liquid Elements) and one in Termux running on latest and unrooted Motorola edge 2022 that I got for free from a friend. All pruned but with current mempool synced also with mempool.space/api over HTTPS. They produce a sound of wooden block when a new block on mainnet gets verified (`blocknotify=script` in `bitcoin.conf`). The same sound as [here](https://display.anyone.eu.org/price.html).

Thank you for reading this far.

-------------------------

