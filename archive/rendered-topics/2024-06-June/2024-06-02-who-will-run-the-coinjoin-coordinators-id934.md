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

1440000bytes | 2024-06-03 06:20:10 UTC | #4

[quote="conduition, post:3, topic:934"]
I’m as excited as the next guy to see joinstr take flight, but seems like that protocol is still too early to fully replace centralized coordinators. From the website you linked:
[/quote]

Electrum plugin can be used on mainnet when a [threading issue](https://gitlab.com/1440000bytes/joinstr/-/issues/7) is fixed.

[quote="conduition, post:3, topic:934"]
Ecash might provide a dark horse candidate for smaller scale off-chain coin mixing. A hypothetical ecash implementation could route LN payments through multiple mints to avoid correlation by common payment hash/preimages. If the wallet uses TOR when making requests, and spaces out the transfers in time, they can avoid network level correlation. I don’t think this will be effective yet, as right now the pool of money in any ecash mint is just too small and the pattern would be too obvious.
[/quote]

I had shared a relevant use case in [other thread](https://delvingbitcoin.org/t/dnm-ecash-and-privacy/916) although it works **with coinjoin** and does not replace it. Most of the privacy tools that I have used in bitcoin complement each other however users are often misguided by developers.

Example: Coinjoin a UTXO, open LN channel with toxic change and use mercury layer with post-mix UTXO or payjoin

-------------------------

everythingSats | 2024-06-04 16:00:23 UTC | #5

This is light years ahead of what's possible

-------------------------

real-or-random | 2024-06-11 06:59:48 UTC | #6

[quote="kravens, post:1, topic:934"]
Now there is still the alternative [JoinMarket with Jam ](https://jamdocs.org/philosophy/03-joinmarket/) as an improved user-interface that works without a central coordinator, but has a more complex setup process than the previously mentioned wallets.
[/quote]

Do you happen to know what it uses instead? I can't find any technical details. All the docs just say that it doesn't use a central coordinator.

-------------------------

1440000bytes | 2024-06-17 19:27:30 UTC | #7

It uses directory nodes for peer (maker and taker) discovery. IRC and Onion service for communication between peers.

[https://github.com/JoinMarket-Org/joinmarket-clientserver/blob/35e6f286fb9be0dc90c2c5c7648d380c103a3870/src/jmclient/configure.py#L143](https://github.com/JoinMarket-Org/joinmarket-clientserver/blob/35e6f286fb9be0dc90c2c5c7648d380c103a3870/src/jmclient/configure.py#L143)

```
[MESSAGING:onion]
# onion based message channels must have the exact type 'onion'
# (while the section name above can be MESSAGING:whatever), and there must
# be only ONE such message channel configured (note the directory servers
# can be multiple, below):
type = onion

socks5_host = localhost
socks5_port = 9050

# the tor control configuration.
# for most people running the tor daemon
# on Linux, no changes are required here:
tor_control_host = localhost
# or, to use a UNIX socket
# tor_control_host = unix:/var/run/tor/control
# note: port needs to be provided (but is ignored for UNIX socket)
tor_control_port = 9051

# the host/port actually serving the hidden service
# (note the *virtual port*, that the client uses,
# is hardcoded to as per below 'directory node configuration'.
onion_serving_host = 127.0.0.1
onion_serving_port = 8080

# directory node configuration
#
# This is mandatory for directory nodes (who must also set their
# own *.onion:port as the only directory in directory_nodes, below),
# but NOT TO BE USED by non-directory nodes (which is you, unless
# you know otherwise!), as it will greatly degrade your privacy.
# (note the default is no value, don't replace it with "").
hidden_service_dir =
#
# This is a comma separated list (comma can be omitted if only one item).
# Each item has format host:port ; both are required, though port will
# be 5222 if created in this code.
# for MAINNET:
directory_nodes = g3hv4uynnmynqqq2mchf3fcm3yd46kfzmcdogejuckgwknwyq5ya6iad.onion:5222,3kxw6lf5vf6y26emzwgibzhrzhmhqiw6ekrek3nqfjjmhwznb2moonad.onion:5222,bqlpq6ak24mwvuixixitift4yu42nxchlilrcqwk2ugn45tdclg42qid.onion:5222

# for SIGNET (testing network):
# directory_nodes = rr6f6qtleiiwic45bby4zwmiwjrj3jsbmcvutwpqxjziaydjydkk5iad.onion:5222,k74oyetjqgcamsyhlym2vgbjtvhcrbxr4iowd4nv4zk5sehw4v665jad.onion:5222,y2ruswmdbsfl4hhwwiqz4m3sx6si5fr6l3pf62d4pms2b53wmagq3eqd.onion:5222

# This setting is ONLY for developer regtest setups,
# running multiple bots at once. Don't alter it otherwise
regtest_count = 0,0

## IRC SERVER 1: Darkscience IRC (Tor, IP)
################################################################################
[MESSAGING:darkscience]
# by default the legacy format without a `type` field is
# understood to be IRC, but you can, optionally, add it:
# type = irc
channel = joinmarket-pit
port = 6697
usessl = true

# For traditional IP:
#host = irc.darkscience.net
#socks5 = false

# For Tor (recommended as clearnet alternative):
host = darkirc6tqgpnwd3blln3yfv5ckl47eg7llfxkmtovrv7c7iwohhb6ad.onion
socks5 = true
socks5_host = localhost
socks5_port = 9050

# IRC SERVER 2: (backup) hackint IRC (Tor, IP)
###############################################################################
[MESSAGING:hackint]
channel = joinmarket-pit
# For traditional IP:
# host = irc.hackint.org
# port = 6697
# usessl = true
# socks5 = false
# For Tor (default):
host = ncwkrwxpq2ikcngxq3dy2xctuheniggtqeibvgofixpzvrwpa77tozqd.onion
port = 6667
usessl = false
socks5 = true
socks5_host = localhost
socks5_port = 9050
```

-------------------------

1440000bytes | 2024-06-17 19:32:55 UTC | #8

Messaging protocol: https://github.com/JoinMarket-Org/JoinMarket-Docs/blob/d098c022b2fa4d148337aa45e995083e3416a3c2/Joinmarket-messaging-protocol.md

-------------------------

real-or-random | 2024-06-18 12:45:46 UTC | #9

[quote="1440000bytes, post:7, topic:934"]
It uses directory nodes for peer (maker and taker) discovery.
[/quote]

But then this mostly shifts the problem: Who will run those?

-------------------------

kravens | 2024-07-29 18:46:11 UTC | #11

We've found a way to make Trezor Suite allow CoinJoin again with alternative coordinators, a guide how to set it up has been written here: [Change Coordinator (kravens.nl)](https://wasabi.kravens.nl/#trezor)
Basically starting the Trezor Suite app with a debug argument setting a custom coordinator server, with an example given for connecting to the Noderunners Coordinator.

-------------------------

kravens | 2024-08-28 17:15:28 UTC | #12

With JoinMarket there are people that make their bitcoin available as a Maker and then people who want to do a coinjoin (named Taker) select from several Makers and pay them a certain percentage or flat fee for their liquidity. Makers are ideally online all the time, where takers only interact with them when they coordinate their own coinjoin.

-------------------------

AdamISZ | 2024-09-19 16:21:53 UTC | #13

Sorry for necroposting here but I only just noticed this thread.

So, yes you basically have it right, of course, Joinmarket never "solved" the centralized **communication** coordinator problem (see slight caveat below). Certainly, not as cleanly as your earlier Coinshuffle work did.

The only caveat is: the "directory node" concept, offering a service on *.onion, was intended to partially solve it, because its intention was to shift the role from communication coordinator to only a peer discovery service.

I say "intention" because, mainly because of difficulties in using ephemeral tor onion services, it has proved not to work very well in practice. We did have a period shortly after we added that feature, when we could happily say there were coinjoins of larger size being coordinated, and also since most of the communication occurred p2p (the idea was that makers also ran their own onion services, so that the taker would go to the directory node to advertise, then after choosing makers, it would talk to them directly), it reduced the centralization element very significantly.

But even if that did, or will, work perfectly, it still does not achieve what a protocol like Coinshuffle achieves in that regard.

-------------------------

real-or-random | 2024-09-20 14:03:45 UTC | #14

[quote="AdamISZ, post:13, topic:934"]
So, yes you basically have it right, of course, Joinmarket never “solved” the centralized **communication** coordinator problem (see slight caveat below). Certainly, not as cleanly as your earlier Coinshuffle work did.
[/quote]

Okay, I didn't have CoinShuffle in mind when I commented above. CoinShuffle(++) is not much better: when it comes to communication, it still relies on a coordinator (or strictly speaking, a "bulletin board" because the only functionality is receiving messages and broadcasting them). Moreover, you still need a way to assign participants into sessions. So it suffers from all the hard practical issues with CoinJoin...

-------------------------

AdamISZ | 2024-09-20 15:23:05 UTC | #15


[quote="real-or-random, post:14, topic:934"]
Okay, I didn’t have CoinShuffle in mind when I commented above. CoinShuffle(++) is not much better: when it comes to communication, it still relies on a coordinator (or strictly speaking, a “bulletin board” because the only functionality is receiving messages and broadcasting them). Moreover, you still need a way to assign participants into sessions. So it suffers from all the hard practical issues with CoinJoin…
[/quote]

Yes, my bad, my thinking is fuzzy there: you're solving centralized coordination of one coinjoin, but (and yes I remember your 'bulletin board' definition there) you're not actually claiming to solve communication centralization (which matters even more for those other implementations like Wasabi, Samurai) because the only type of bulletin board we know of that can't be selectively censored is the blockchain and that's not a practical option. Maybe ham radio? :smile:

-------------------------

AdamISZ | 2024-09-20 22:46:58 UTC | #17

[quote="1440000bytes, post:16, topic:934"]
TBH only coinjoin implementation that did try to fix it was joinstr.
[/quote]

How does joinstr do more on this specific issue (*communication* coordination centralization) than Joinmarket?

For context, over the years Joinmarket developed this model:

At first, it supported 1 IRC server as a central communication coordinator, but encrypted the private information between makers and takers E2E using a DH key exchange (see: libsodium). So while the central coordinator could censor messages that could affect the market-making coordination, it couldn't censor in a way relevant to the coinjoins themselves (selecting certain utxos and not others etc.). Still of course, at that level, that's super censorable and weak, because one server.

Then somewhat later it started supporting multiple redundant IRC servers, with here "redundant" very specifically meaning *all* messages passed over *all* connected servers. We also added details like signatures over the endpoints and the communication servers to prevent replay attacks across different "message channels" (which is the abstraction, with IRC servers just being a type of that). At this point it's already quite robust. Probably 6-8 different IRC servers were used in practice, though usually only 3 at a time.

A couple relevant details: obviously this required ephemeral public key identities to sign over, which were done with secp256k1 of course. Also, perhaps obvious, but we used onion endpoints of the IRC servers.

Then even later we added "directory nodes" which tried to go a little bit farther: not only are the messages between parties encrypted on the message channel, but even the parties talk p2p; the taker/maker model helps here, because makers can run their own onion services since they're always online. This model worked extremely well for a while, allowing much larger coinjoins with more performant messaging (IRC servers are a bit of a bottleneck for coordinating), and a better privacy/censorship resistance model (the directory nodes effectively coordinate discovery of makers for takers and that's it). The problem was that it kind of started to collapse about 9 months after release, because the directory node onion services just weren't working. I don't know if this has been solved (I think not).

(Also the 'redundant message channel concept' still applied, so you would send data over all available ones, whether the directory node type or the IRC type; btw it would not be at all difficult, imo, to code nostr as a further redundant message channel, here, but you'd need to have some network anonymity as we do for the others).

Is there some specific technique used by joinstr other than 'send over nostr'? You could say that communication happens peer to peer but it's still going over relays right? Would you say multiple nostr relays is better than multiple redundant IRC servers? I suspect it *is* better, but mainly in that performance will be better; I'd want to know how the anonymity part would work, but I'm certainly no expert on that.

-------------------------

1440000bytes | 2024-09-21 01:44:52 UTC | #18

[quote="AdamISZ, post:17, topic:934"]
How does joinstr do more on this specific issue (*communication* coordination centralization) than Joinmarket?
[/quote]

No hardcoded addresses for bootstrapping. Users decide their own relays and it uses SIGHASH ALL | SIGHASH ANYONECANPAY.

It can easily be upgraded to [covenants](https://x.com/1440000bytes/status/1821357538899611681).

-------------------------

AdamISZ | 2024-09-21 15:17:30 UTC | #19

Oh, so it uses SINGLE|ACP ? Every time people analyzed that idea before they stopped because of the restriction of SINGLE requiring matching in/out indices, which means you can't get privacy * . How did you get round that? Maybe there's a doc I can read somewhere?

\* I mean, a scenario where one contributor adds an input and an output then "passes on" the incomplete transaction, they have to match the position of the input and output.

[quote="1440000bytes, post:18, topic:934"]
No hardcoded addresses for bootstrapping.
[/quote]
Aren't nostr relay addresses hardcoded, as seeds? Obviously you can discover others, but that's possible with all of these approaches.

-------------------------

