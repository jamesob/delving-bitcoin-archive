# Who will run the CoinJoin coordinators?

kravens | 2024-06-02 10:01:26 UTC | #1

Recently there has been a regulatory move against non-custodial privacy tools that led to the closing of the coordinator servers of Samourai Wallet ([police confiscation of server](https://www.justice.gov/usao-sdny/pr/founders-and-ceo-cryptocurrency-mixing-service-arrested-and-charged-money-laundering)) and Wasabi Wallet ([voluntary shutdown](https://blog.wasabiwallet.io/zksnacks-is-discontinuing-its-coinjoin-coordination-service-1st-of-june/)). These were service providers that had attracted a lot of liquidity for CoinJoin transactions that therefore had a good resulting anonymity set for participants. Now there is still the alternative [JoinMarket with Jam](https://jamdocs.org/philosophy/03-joinmarket/) as an improved user-interface that works without a central coordinator, but has a more complex setup process than the previously mentioned wallets.

One of the new developments I've jumped into is the [BTCPay Server CoinJoin plugin](https://docs.btcpayserver.org/Wabisabi/#installation) that uses the open WabiSabi protocol (used in Wasabi Wallet) and allows users to easily spin up their own coordinator.

This BTCPay coordinator also advertises itself via Nostr relay, so to make it easy to find by other wallets that implemented the WabiSabi protocol (e.g. WasabiWallet, Trezor Suite, BTCPay server). An example tool looking for active coordinator servers can be found [here](https://github.com/Kukks/WasabiNostr).

Users of Wasabi Wallet and BTCPay server can already freely choose a different coordinator, to help my local community stay private on-chain I've set up a coordinator service too ([instructions to join are here](https://wasabi.kravens.nl)).

One of the issues that new coordinators face is the lack of participants, as the wallets don't yet have a proper flow to guide them towards these alternative coordinators. Or in the case of Trezor Suite the option to switch coordinator server is simply not implemented.

I'm curious to what solutions the bitcoin privacy community will come, will we see a hydra of BTCPay Coordinators that become more numerous with each head cut off. Or will JoinMarket see more adoption from people whom depended on the centralized coordinators of Samourai and Wasabi Wallet before.

-------------------------

1440000bytes | 2024-06-02 15:23:09 UTC | #2

[quote="kravens, post:1, topic:934"]
Now there is still the alternative [JoinMarket with Jam](https://jamdocs.org/philosophy/03-joinmarket/) as an improved user-interface that works without a central coordinator, but has a more complex setup process than the previously mentioned wallets.
[/quote]

Alternative that does not require complex setup: [joinstr.xyz](https://joinstr.xyz)

> Who will run the Coinjoin coordinators?

Nobody needs to run a coordinator for joinstr as exsiting nostr relays are used to communicate between peers. Relays don't really "coordinate" coinjoin in the same way as they do in wabisabi and whirlpool. They are just used to read/write pool information publicly and PSBT in encrypted channels.

NIP: [https://gitlab.com/1440000bytes/joinstr/-/blob/main/NIP.md](https://gitlab.com/1440000bytes/joinstr/-/blob/main/NIP.md)

-------------------------

conduition | 2024-06-02 22:18:52 UTC | #3

@1440000bytes I'm as excited as the next guy to see joinstr take flight, but seems like that protocol is still too early to fully replace centralized coordinators. From the website you linked:

> **Can i use joinstr on mainnet rigth now?**
> 
> No. Please don't use it on mainnet as there are still bugs that need to be fixed for everything to work properly.

I tried Joinmarket recently but had trouble getting its automatic mixing system to work. The UX is garbage and setup is tedious. Haven't tried Jam yet.

Ecash might provide a dark horse candidate for smaller scale off-chain coin mixing. A hypothetical ecash implementation could route LN payments through multiple mints to avoid correlation by common payment hash/preimages. If the wallet uses TOR when making requests, and spaces out the transfers in time, they can avoid network level correlation. I don't think this will be effective yet, as right now the pool of money in any ecash mint is just too small and the pattern would be too obvious. 

Also remember that PTLCs will someday be a thing on Lightning. This would vastly improve privacy of payment routing by breaking the payment-hash correlation attack vector[^1]. Perhaps one of the best ways to mix coins will someday be to use PTLC-driven submarine swaps to convert one larger pre-swap UTXO into a series of smaller post-swap UTXOs. Then your money is hidden among the set of all UTXOs performing submarine swaps with that service, which could be very large in the case of popular services like LN-Labs' Loop.



[^1]: Imagine an LN node paying another node 4 hops away, using a private channel (A -> B -> C -> D -> E). If the payment is an HTLC, and nodes B and E are colluding, they can prove it was node A who paid node E, because each hop uses a common hash/preimage pair in their contracts. With PTLCs, each hop uses a unique key (point) for the contract. Nodes B and E would not be able to conclusively prove A was the sender, although they could guess based on timing and amount correlation. For hard proof, they'd need nodes C and D to also cooperate.

-------------------------

