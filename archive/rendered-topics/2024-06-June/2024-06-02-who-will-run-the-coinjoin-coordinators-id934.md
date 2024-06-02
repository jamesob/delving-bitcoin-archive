# Who will run the CoinJoin coordinators?

kravens | 2024-06-02 09:59:08 UTC | #1

Recently there has been a regulatory move against non-custodial privacy tools that led to the closing of the coordinator servers of Samourai Wallet ([police confiscation of server](https://www.justice.gov/usao-sdny/pr/founders-and-ceo-cryptocurrency-mixing-service-arrested-and-charged-money-laundering)) and Wasabi Wallet ([voluntary shutdown](https://blog.wasabiwallet.io/zksnacks-is-discontinuing-its-coinjoin-coordination-service-1st-of-june/)). These were service providers that had attracted a lot of liquidity for CoinJoin transactions that therefore had a good resulting anonymity set for participants. Now there is still the alternative [JoinMarket with Jam](https://jamdocs.org/philosophy/03-joinmarket/) as an improved user-interface that works without a central coordinator, but has a more complex setup process than the previously mentioned wallets.

One of the new developments I've jumped into is the [BTCPay Server CoinJoin plugin](https://docs.btcpayserver.org/Wabisabi/#installation) that uses the open WabiSabi protocol (used in Wasabi Wallet) and allows users to easily spin up their own coordinator.

This BTCPay coordinator also advertises itself via Nostr relay, so to make it easy to find by other wallets that implemented the WabiSabi protocol (e.g. WasabiWallet, Trezor Suite, BTCPay server). An example tool looking for active coordinator servers can be found [here].(https://github.com/Kukks/WasabiNostr)

Users of Wasabi Wallet and BTCPay server can already freely choose a different coordinator, to help my local community stay private on-chain I've set up a coordinator service too ([instructions to join are here](https://wasabi.kravens.nl)).

One of the issues that new coordinators face is the lack of participants, as the wallets don't yet have a proper flow to guide them towards these alternative coordinators. Or in the case of Trezor Suite the option to switch coordinator server is simply not implemented.

I'm curious to what solutions the bitcoin privacy community will come, will we see a hydra of BTCPay Coordinators that become more numerous with each head cut off. Or will JoinMarket see more adoption from people whom depended on the centralized coordinators of Samourai and Wasabi Wallet before.

-------------------------

