# BIP324 Proxy: easy integration of v2 transport protocol for light clients (PoC)

theStack | 2024-03-13 17:32:11 UTC | #1

Hi,

I've been working on a small tool for enabling peer-to-peer encryption for
bitcoin clients that haven't implemented BIP324 yet.

The basic idea is to run a local process with the sole purpose of translating between p2p v1 and v2 protocols.
The proxy starts a server socket on local TCP port 1324 [1] and spins up a new thread for every
incoming v1 connection. The remote peer address is determined from the incoming first
VERSION message which conveniently contains this information in the `addr_recv` field
(see https://en.bitcoin.it/wiki/Protocol_documentation#version).
After performing the v2 handshake [2], the previously received VERSION message is sent to the remote node
and subsequently, the proxy fowards incoming messages from either side to the opposite side,
translating it to the corresponding p2p version format. 

![image|690x355, 75%](upload://w5cJ83Hb5sef1EN4HLxg8tNPoHG.png)


To take use of BIP324 proxy, all a light client has to do is to initiate all outbound
P2P connections to localhost:1324 instead of the actual remote peer address on the TCP level.
This usually doesn't need more than a single-line patch if hardcoded, though ideally a client should
provide this as a command-line option (e.g. `-bip324proxy=`...).

The implementation is written in Python3 without any external dependencies. Most of the code,
especially the cryptographic primitives, are not by me, but taken from Bitcoin Core's
BIP324 implementation in the functional test framework (Kudos to stratospher and sipa).
Note that this project is still in proof-of-concept phase, terribly slow and vulnerable to side-channel attacks. It's not recommended to use it for anything but tests right now. To reflect that, only signet is supported (though it's trivial to enable other networks by changing the `NET_MAGIC` constant).

The plan is to do an efficient rewrite in Rust. As I'm not fluent in this language, this
could take a while. Here are the links to the Github repository and a presentation that I held recently in a Brink engineering call:

https://github.com/theStack/bip324-proxy
https://github.com/theStack/bip324-proxy/raw/master/doc/bip324-proxy_presentation.pdf

The slides contain a few examples of light clients that I've tested the BIP324 proxy with,
showing patches that are needed for redirection to the proxy.

Suggestions and ideas are very welcome.

Cheers,
Sebastian

[1] Note that the port 324 would have been a nice choice reflecting the BIP number, but TCP/IP
ports below 1024 need superuser privileges at least on Unix-like operating systems,
hence I added a "1" (standing for "v1") in front. 1324 is not taken yet, according to some well-known
port lists that I checked, so it shouldn't collide with other local services and I think
it's still a nice, somewhat easy-to-remember choice.

[2] Note that reconnection with v1 (if the v2 handshake fails) is implemented, but currently hardcoded to be disabled, in order to keep the full promise of a "BIP324 proxy", i.e. _only_ v2 connections are allowed. As v2 support is still not widespread on listening nodes, it's probably not a good idea to use this yet, as one could end up without any connection.

-------------------------

0xB10C | 2024-03-14 01:28:33 UTC | #2

Cool project!

[quote="theStack, post:1, topic:678"]
The remote peer address is determined from the incoming first VERSION message which conveniently contains this information in the `addr_recv` field.
[/quote]

There are P2P clients that don't put their address in `addr_recv`. They put, e.g. `127.0.0.1`, `0.0.0.0`, or whatever in there. Would this be a problem?

[quote="theStack, post:1, topic:678"]
The plan is to do an efficient rewrite in Rust.
[/quote]

I haven't seen anyone working on a BIP324 implementation in Rust. Might fit well in the excellent [rust-bitcoin](https://github.com/rust-bitcoin/rust-bitcoin) library or some other place in their ecosystem (e.g. rust-bitcoin/bip324).

-------------------------

theStack | 2024-03-14 02:20:39 UTC | #3

[quote="0xB10C, post:2, topic:678"]
[quote="theStack, post:1, topic:678"]
The remote peer address is determined from the incoming first VERSION message which conveniently contains this information in the `addr_recv` field.
[/quote]

There are P2P clients that don’t put their address in `addr_recv`. They put, e.g. `127.0.0.1`, `0.0.0.0`, or whatever in there. Would this be a problem?
[/quote]

Oh, that's good to know, I wasn't aware. If the VERSION's `addr_recv` field doesn't contain the real address of the remote node, that would indeed be a problem, as this is the only way for the proxy to know where to initiate the v2 connection to. If set to an arbitrary value, the connection would then very likely fail (or connect to a different peer than intended, which is probably even worse).

Do you know of any concrete P2P clients that follow this practice (or is it more like connections with obscure user agents that can't be tied to a concrete implementation)? I might add a prerequisites section to README.md mentioning the reliance on `addr_recv` being set correctly, together with a list of clients that are already known to be incompatible with BIP324 proxy.

[quote="0xB10C, post:2, topic:678"]
[quote="theStack, post:1, topic:678"]
The plan is to do an efficient rewrite in Rust.
[/quote]

I haven’t seen anyone working on a BIP324 implementation in Rust. Might fit well in the excellent [rust-bitcoin ](https://github.com/rust-bitcoin/rust-bitcoin) library or some other place in their ecosystem (e.g. rust-bitcoin/bip324).
[/quote]

Yeah, I also thought that putting some parts of BIP324 to a library might be a good idea. Will for sure take a deeper look at rust-bitcoin at some point in the course of my upcoming Rust journey.

-------------------------

0xB10C | 2024-03-14 10:57:55 UTC | #4

[quote="theStack, post:3, topic:678"]
Do you know of any concrete P2P clients that follow this practice (or is it more like connections with obscure user agents that can’t be tied to a concrete implementation)? I might add a prerequisites section to README.md mentioning the reliance on `addr_recv` being set correctly, together with a list of clients that are already known to be incompatible with BIP324 proxy.
[/quote]

I briefly looked into it and it seems like all **inbound** [LinkingLion](https://b10c.me/observations/06-linkinglion/) connections set `127.0.0.1` and **inbound** i2p and tor (presumably) Bitcoin Core connections set `0.0.0.0`.

Based on the "BIP324 proxy scenario" graphic and the slides you linked, I noticed that the proxy is only for outbound connections for now, correct? You mention "Investigate inbound connections support via reverse proxy" as TODO. I incorrectly assumed you're implementing in and outbound. For outbound only, it should be fine. 

However, only version 1 address serialization is possible in (inbound/outbound) version messages. [BIP-155](https://github.com/bitcoin/bips/blob/b3701faef2bdb98a0d7ace4eedbeefa2da4c89ed/bip-0155.mediawiki) address serialization of e.g. TorV3, I2P, CJDNS isn't possible. These will always be `0.0.0.0` (https://github.com/bitcoin/bitcoin/blob/e1ce5b8ae9124717c00dca71a5c5b43a7f5ad177/src/net_processing.cpp#L1547).

-------------------------

