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

