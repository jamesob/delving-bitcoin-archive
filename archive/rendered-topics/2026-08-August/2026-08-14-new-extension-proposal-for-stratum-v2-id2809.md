# New Extension Proposal for Stratum V2

radikalreems | 2026-08-14 20:09:52 UTC | #1

I would like feedback on my new extension proposal to StratumV2 that would send PoW shares back to the Template Provider.

I am working on a project that would source hashrate for clients. Currently, I use DATUM to allow clients to create block templates if they like but DATUM may have some unfortunate future changes due to the BIP110 PoW hardfork. Either way, SV2 could technically be a better solution in this case as a client could simply just run the Template Provider and avoid the overhead of Pool -> ASIC connections. 

The issue is that the Template Provider never receives the work being sent to the pools and therefore cannot be sure that it's their templates that are being mined. They will get the share when a block is found but for solo-mining this is impractical as a proof of mining. Especially if a company (like mine) handles the pools and ASIC connections dynamically, the Template Provider (client) would like constant proof of their templates being mined.

I propose simply routing the existing work shares that are moving from Mining Devices (ASICs) to Pools, to also be sent to the Template Provider if they please.

This can be done with a new extension, allowing it to be opt-in for situations like this.

I have created a discussion on the sv2-spec repo that you can find here:

https://github.com/stratum-mining/sv2-spec/discussions/204


Thank you for any feedback!

-------------------------

