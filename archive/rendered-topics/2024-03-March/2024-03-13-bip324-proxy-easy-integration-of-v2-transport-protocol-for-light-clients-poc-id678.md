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

theStack | 2024-03-14 12:55:53 UTC | #5

[quote="0xB10C, post:4, topic:678"]
I briefly looked into it and it seems like all **inbound** [LinkingLion ](https://b10c.me/observations/06-linkinglion/) connections set `127.0.0.1` and **inbound** i2p and tor (presumably) Bitcoin Core connections set `0.0.0.0`.
[/quote]

Thanks for taking a deeper look! :ok_hand: 

[quote="0xB10C, post:4, topic:678"]
Based on the “BIP324 proxy scenario” graphic and the slides you linked, I noticed that the proxy is only for outbound connections for now, correct? You mention “Investigate inbound connections support via reverse proxy” as TODO.
[/quote]
Yes, it only works for outbound connections for now. The reverse proxy idea seemed interesting to me from a technical perspective, but thinking more about it, it's probably not that useful in practice. There is already the possibility to run a listening node with BIP324 now (by running Bitcoin Core v26.0+), and I'd assume that relevant alternative node implementations implement it soon (at least significantly earlier than most light clients).

[quote="0xB10C, post:4, topic:678"]
I incorrectly assumed you’re implementing in and outbound. For outbound only, it should be fine.
[/quote]
Ok, that's good news.

[quote="0xB10C, post:4, topic:678"]
However, only version 1 address serialization is possible in (inbound/outbound) version messages. [BIP-155](https://github.com/bitcoin/bips/blob/b3701faef2bdb98a0d7ace4eedbeefa2da4c89ed/bip-0155.mediawiki) address serialization of e.g. TorV3, I2P, CJDNS isn’t possible. These will always be `0.0.0.0` ([bitcoin/src/net_processing.cpp at e1ce5b8ae9124717c00dca71a5c5b43a7f5ad177 · bitcoin/bitcoin · GitHub ](https://github.com/bitcoin/bitcoin/blob/e1ce5b8ae9124717c00dca71a5c5b43a7f5ad177/src/net_processing.cpp#L1547)).
[/quote]
Good point. I haven't really checked how the proxy idea would work together with any of these protocols. I guess it just doesn't, but intuitively I would say it's fine if that's not supported, as these protocols already offer encryption on another layer anyways.

-------------------------

josibake | 2024-03-15 15:20:27 UTC | #6

Really cool! 
[quote="theStack, post:1, topic:678"]
The plan is to do an efficient rewrite in Rust. As I’m not fluent in this language, this could take a while.
[/quote]

I haven't looked into how you have the tool designed, but any interest in making this a rust library? For example, https://github.com/cloudhead/nakamoto has it on their roadmap to add BIP324, and I'm sure there are others. Seems like having a library would be generally useful and something that your proxy could be built around.

I'm also not fluent in Rust but learning and looking for small-ish projects to work on, so I'd be more than happy to help work on a BIP324 rust library if you're interested in collaborating.

-------------------------

theStack | 2024-03-16 08:46:33 UTC | #7

[quote="josibake, post:6, topic:678"]
I haven’t looked into how you have the tool designed, but any interest in making this a rust library? For example, [GitHub - cloudhead/nakamoto: Privacy-preserving Bitcoin light-client implementation in Rust ](https://github.com/cloudhead/nakamoto) has it on their roadmap to add BIP324, and I’m sure there are others. Seems like having a library would be generally useful and something that your proxy could be built around.
[/quote]
Yup, I agree that creating a BIP324 library makes a lot of sense (see also [0xb10c's post above](https://delvingbitcoin.org/t/bip324-proxy-easy-integration-of-v2-transport-protocol-for-light-clients-poc/678/2?u=thestack) where one idea is to add it to rust-bitcoin).

[quote="josibake, post:6, topic:678"]
I’m also not fluent in Rust but learning and looking for small-ish projects to work on, so I’d be more than happy to help work on a BIP324 rust library if you’re interested in collaborating.
[/quote]

Sure, that would be great! I forgot to mention this explicitly in the OP, but of course everyone is more than welcome to contribute. For starters, it should be pretty straight-forward to create a module for the BIP324 cipher suite. I think we can use the same interface as `BIP324Cipher` in Bitcoin Core (see `src/bip324.{h,cpp}`). Given that there are (hopefully) already Rust crates available for efficient implementations of the cryptographic primitives (`secp256k1-ellswift` bindings, ChaCha20(Poly1305), HKDF-SHA256 etc.), this shouldn't even be too much code.

-------------------------

rustaceanrob | 2024-03-17 18:40:09 UTC | #8

Hey, super cool project. I am happy to see other people excited about BIP324 and building tools to drive p2p towards v2. A friend and I have been working on a Rust implementation that will act as an API for clients to place into their pre-existing TCP logic, and we would appreciate some further guidance/contribution. Our main priority has been removing dependencies, and we are now down to just a few from `rust-bitcoin`. My main concern now is building the top level API around the Floresta Rust client, and I will be working on that this week. Our code is [here](https://github.com/rustaceanrob/bip324).

-------------------------

theStack | 2024-03-17 19:48:10 UTC | #9

[quote="rustaceanrob, post:8, topic:678, full:true"]
Hey, super cool project. I am happy to see other people excited about BIP324 and building tools to drive p2p towards v2. A friend and I have been working on a Rust implementation that will act as an API for clients to place into their pre-existing TCP logic, and we would appreciate some further guidance/contribution. Our main priority has been removing dependencies, and we are now down to just a few from `rust-bitcoin`. My main concern now is building the top level API around the Floresta Rust client, and I will be working on that this week. Our code is [here ](https://github.com/rustaceanrob/bip324).
[/quote]

Nice, that's great to hear and comes at the perfect point of time! I'm sure there will be some further questions / suggestions when it comes to the details of the API, but at a first glance this seems like exactly what we need for BIP324 Proxy. Just being curious, what was the motivation to reduce the cryptography dependencies and reimplement them? Do the available Rust crates have any significant drawbacks, or is that more a generic philosophical decision of the library? I'm not saying it's a bad idea (we actually do the same in Bitcoin Core), I just assumed that in Rust the package management works well enough, and I think I wouldn't have a problem depending on a well-maintained cryptographic library, which ideally has the resources to investigate optimizations etc.

As a small update, I've started the Rust rewrite branch yesterday: https://github.com/theStack/bip324-proxy/tree/rust_rewrite
It's still tiny, so far it only creates the local server socket and displays a mesage for an incoming client connection, without spinning up a new thread yet. The plan would be to implement a dummy proxy (v1<->v1) first and only then plug in all the BIP324 related stuff. Let's see how that goes. As always, contributions in any form are welcome (even if it's recommended book material / common pitfalls for network programming in Rust or whatever).

-------------------------

rustaceanrob | 2024-03-17 20:37:32 UTC | #10

Perfect, will keep a pulse on your `src`. My favorite Rust book is [Effective Rust](https://www.lurklurk.org/effective-rust/), and the `tokio` docs have good async TCP examples. On cryptography, the `RustCrypto` collection of crates contain `unsafe` code blocks and are not particularly auditable. I have been in touch with the `rust-bitcoin` maintainers, and they recommended the dependency reduction as a step towards being merged to their community, presumably so the crypto is concise and readable. I am also working on this project as part of a Chaincode program, and I think any additional "proof of work" I can put out will hopefully put my name out as a FOSS dev! If you would like to join forces, you can create a `main` in our `src` and implement the proxy logic there as well. I might also fork yours and see if I can build up a proxy with my crate. Cheers

-------------------------

yonson | 2024-04-14 22:05:16 UTC | #11

Hey @theStack , @rustaceanrob and I have been hacking along in our BIP324 repo and have a working rust-based v2 proxy (heavily influenced by your Python version) I thought you might find interesting! It is async, using the Tokio runtime: https://github.com/rustaceanrob/bip324/blob/fae064c22f77c62e3541cceea33690ef3efad52b/proxy/src/bin/async.rs

I am also going to be working on a threaded implementation since this has been a pretty useful exercise to help inform what type of interface we want our bip324 protocol library to expose. We have a bit more to clean up and iterate on there, but have generally adopted a "sans I/O" approach to keep it runtime agnostic.

-------------------------

theStack | 2024-04-15 17:12:42 UTC | #12

@yonson: That's very cool, thanks for working on that! I'm looking forward to try it out later and do some manual testing with clients that I've tried bip324-proxy with before (i.e. primarily nakamoto, neutrino, bcoin in SPV mode, and last but not least, Bitcoin Core). Also happy to hear that the project helps shaping the interface for the bip324 library.

> I am also going to be working on a threaded implementation since this has been a pretty useful exercise to help inform what type of interface we want our bip324 protocol library to expose. We have a bit more to clean up and iterate on there, but have generally adopted a “sans I/O” approach to keep it runtime agnostic.

Can you elaborate in what aspects the threaded implementation will differ from the current one? To my understanding, a thread is already spawned for each incoming v1 connection (otherwise, only one connection would be supported).

-------------------------

yonson | 2024-04-15 17:35:23 UTC | #13

> Can you elaborate in what aspects the threaded implementation will differ from the current one? To my understanding, a thread is already spawned for each incoming v1 connection (otherwise, only one connection would be supported).

Sure! The async implementation is ["spawning"](https://tokio.rs/tokio/tutorial/spawning) new threads of work, but these are "green threads" not tied 1:1 with operating system threads.

My use of "threaded implementation" is a bit vague with all this concurrency talk, but I specifically mean an operating system thread implementation. The big benefit of this is that we could then code up standard library based wrappers which work against the std::io::Read/Write traits (these generally don't play nice in async-land). The cost of the OS thread impl simplicity is the standard heavy resource usage per connection.

-------------------------

