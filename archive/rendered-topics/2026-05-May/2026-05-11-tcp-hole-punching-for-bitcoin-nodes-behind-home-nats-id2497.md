# TCP hole punching for Bitcoin nodes behind home NATs?

0xB10C | 2026-05-11 13:26:22 UTC | #1

While @willcl-ark and I were discussing 
https://bnoc.xyz/t/did-the-number-of-reachable-nodes-in-residential-isps-increase-since-bitcoin-core-v30-0/122, which indicates that making `-natpmp=1` in Bitcoin Core might not have had the hoped effect yet of making more nodes behind a home router NAT reachable, it occurred to us that we might be able to use TCP hole punching to connect two otherwise unreachable home nodes together.

This should not replace inbound-outbound connections, and isn't a replacement of `natpmp` either, but offers a potential best-effort extension for extra connectivity of home nodes. We discussed this at a recent in-person Bitcoin Core developer meeting and want to keep the conversation going here. We are far from having a proposal for standardization nor an implemenation. At the moment, there are likely more open questions than answers.


## TCP hole punching

TCP hole punching lets two computers behind certain NATs connect directly, without relaying traffic through a server. My understanding of one way we could use this:

- Alice & Bob: unreachable home nodes behind a NAT
- Charlie: reachable node

1. Both nodes Alice and Bob connect via TCP to a reachable coordinator node Charlie. Charlie could advertise it's coordinator/matchmaking capabilities in a service flag.
2. Charlie learns the public endpoints of Alice and Bob. When Alice's packets pass through her NAT, the NAT rewrites them with a public IP and port. Charlie sees this "outside" address and does the same for Bob.
3. Charlie swaps endpoints. It tells Alice Bob's public IP:port, and tells Bob Alice's.
4. Both sides simultaneously initiate a TCP connection to each other (See "Figure 7: Simultaneous Connection Synchronization" in [RFC 9293](https://datatracker.ietf.org/doc/html/rfc9293#name-simultaneous-connection-syn)) using the *same local port* they used to talk to Charlie (this requires `SO_REUSEADDR`/`SO_REUSEPORT` on the socket). Each side sends a SYN to the other's public endpoint.
5. The outbound SYNs punch holes. When Alice's SYN leaves her NAT heading for Bob, her NAT creates a mapping: "traffic coming back from Bob's address is allowed in." Bob's NAT does the same in reverse.
6. The SYNs cross in flight. Now each NAT has a hole that permits the other side's SYN through. The two stacks see this as a "simultaneous open" and complete the handshake directly - no server in the middle.

The tricky part is that TCP is stateful and timing-sensitive, so it's flakier than UDP hole punching. 

Additionally, it only works for certain types of NAT: [RFC 4787](https://datatracker.ietf.org/doc/html/rfc4787#section-4.1) defines three NAT address and port mapping behaviors that are relevant here: 

- Endpoint-Independent Mapping (EIM): picks one external (IP, port) for a given internal socket and reuses it regardless of where the packet is going. Once Alice has sent any outbound packet, her external endpoint is fixed and predictable. **Hole punching works.** Possibly common on consumer residential routers  (cable, fibre, DSL).
- Address-Dependent Mapping (ADM): the external port stays the same while Alice is talking to the same destination IP, but a new external port is assigned the first time she contacts a different destination IP. **Hole punching is hard**. One option is to try to predict ports, but this likely does not work reliably.
- Address+Port-Dependent Mapping (APDM; or symmetric NAT): Every distinct destination (IP, port) gets a fresh external port. We cannot predict what external port Alice's NAT will assign for any specific peer. **Hole punching does not work.** Typical of CGNAT deployments, mobile carriers, and restrictive enterprise networks.

## Usage in the Bitcoin protocol 

Two approaches were discussed initially, but I feel like the design space is not fully explored yet.

1. Rendevouz node: The home-nodes Alice and Bob both want to offer connection slots, but aren't reachable from the internet. Both happen to pick node Charlie offering a rendezvous/matchmaking service flag and connect to it. Alice might connect first, and needs to wait until Bob connects too. Once both are connected, Charlie then tells them the respective other IP (step 3.). and that they should try to holepunch now. The connection to Charlie now stops. Alice and Bob try for a while to connect to each other.

2. Inbound handoff instead of eviction: An alternive approach is that both Alice and Bob connect to Charlie, but since Charlies inbound slots are full, it tells them it's going to hand them off to each other and the connection stops. Currently, when inbound slots are full and peers are evicted, we close the connection. Here, we offer them a way to retain a connection, but with a different node.

In both of these approaches, Charlie learns that Alice and Bob might now be connected, but not for how long. Alice learns that Bob and Charlie were connected. Bob learns that Alice and Charlie were connected. Both don't know how long they were connected for.

There also was brief discussion on how to make this more private by not requiring a coordnator Carol. Can Alice and Bob, once they've figured out their public IP:port behind a EIM NAT, use a e.g. Tor connection to communicate their IP:port to establish a clearnet connection. A protocol without a coordinator makes it a lot easier to reason about the connection either being inbound or outbound, and is likely a lot easier to implement.

## Implementation in Bitcoin Core?

Bitcoin Core currently uses the notion of "inbound" and "outbound" connections. What would hole-punched connections be? Outbounds are generally more trusted since we choose them, relay transactions faster, etc. Additionaly, there's the concept of "outbund-full-relay" and "outbound-block-only" connections.

The connections intially were two "outbound"'s to Charlie. But since Alice didn't choose the connection to Bob from her addrman (Charlie did while matchmaking), this is not a "we-picked-this-address-from-our-addrman" outbound. So would they be inbound-inbound? We probably have to rethink (and refactor) how we think about connections a bit.

Additional, e.g. BIP-324 has the concept of a Initiator and Responder for encrypted transport. Charlie can assign this to one of the sides. There might be other places where this is useful.

## Open Questions 

- How well does TCP hole punching work in practice? Are there any stats on this by people using it? Does it work with most home routers? How common are EIM NATs and A(P)DM NATs?
- Is there theoretical/academic research or P2P projects using this in the wild that we can learn from? Can we come into contact with some them?
- When using a coordinator, what happens to Alice if either Bob or Charlie are malicious? What attacks can be done? This likely opens opportunities for sybils or ecplise attacks. How dangerous are these?
- Does having "holepunch connections" really address problems we have today? Does it increase the connection slots the network and allows for more peer diversity for unreachable nodes? Does it offer better node inter-connectivity and increases resistance against partition and eclipse attacks? Does it allow for shorter relay paths between unreachable nodes (and is this something we want to optimize for)?
- How much new and changed code is needs to implement this? It might change a lot of our current reasoning and understanding of P2P connections. How do we refactor the existing code for this?

## Some resources

- Paper: Communication Across Network Address Translators: https://pdos.csail.mit.edu/papers/p2pnat.pdf
- https://en.wikipedia.org/wiki/TCP_hole_punching
- libp2p hole-punching https://libp2p.io/docs/hole-punching/ and dcutr (matchmaking via a relay, not coordinator) https://libp2p.io/docs/dcutr/
- Paper: NAT Hole Punching Revisited: https://kops.uni-konstanz.de/server/api/core/bitstreams/29a35a1d-40f1-4290-9d03-dae21f2b9c36/content

-------------------------

0xB10C | 2026-05-12 14:52:17 UTC | #2

[quote="0xB10C, post:1, topic:2497"]
How common are EIM NATs and A(P)DM NATs?
[/quote]

I've set up a server on two different IPs that returns the NAT `IP:port` to a connecting client. One can use [`nat-check.py`](https://github.com/0xB10C/tcp-nat-check/blob/15784f5d0897e5a3e38e9c9729fc39c60ef0d8fd/nat-check.py) to connect to these and get a classification of their NAT. The code for this, mostly written by an LLM, can be found in https://github.com/0xB10C/tcp-nat-check. Making requests to my hosts leaks your IP address to me, so you might chose to run your own servers. IP addresses are masked in output by default.


For me, at home and also using a phone hotspot indicates APDM for IPv4 and "no NAT" for IPv6:

```bash
$ python3 nat-check.py http://b10c.me:7770 http://b10c.me:7771 http://bnoc.xyz:7770 http://bnoc.xyz:7771

nat-check  TCP NAT mapping classifier

── IPv4 ──

local source port: 53321

  destination                            external addr             
  ────────────────────────────────────────────────────────────────
  b10c.me:7770 (x.x.x.1)                 x.x.x.2:61314             
  b10c.me:7771 (x.x.x.1)                 x.x.x.2:59431             
  bnoc.xyz:7770 (x.x.x.3)                x.x.x.2:63172             
  bnoc.xyz:7771 (x.x.x.3)                x.x.x.2:64569             

classification: ADDRESS+PORT-DEPENDENT MAPPING (APDM, 'symmetric')

  External port varies per destination (IP, port). TCP hole punching is very
  unlikely to work for you: the coordinator cannot predict the external port
  your NAT will assign for any given peer. This pattern is typical of CGNAT,
  mobile carriers, and restrictive enterprise networks.

── IPv6 ──

local source port: 53322

  destination                            external addr             
  ────────────────────────────────────────────────────────────────
  b10c.me:7770 (x::x:4)                  x::x:5:53322              
  b10c.me:7771 (x::x:4)                  x::x:5:53322              
  bnoc.xyz:7770 (x::x:6)                 x::x:5:53322              
  bnoc.xyz:7771 (x::x:6)                 x::x:5:53322              

classification: NO NAT

  External address (x::x:5:53322) equals the local address. There is no NAT;
  hole punching is unnecessary. However, your home router may still have a
  stateful firewall that blocks unsolicited inbound connections. You may need to
  open the port on your router to be reachable.
```

Via [Obscura VPN](https://obscura.net/refer#amerch) it's EIM for both IPv4 and IPv6: 

```bash
python3 nat-check.py http://b10c.me:7770 http://b10c.me:7771 http://bnoc.xyz:7770 http://bnoc.xyz:7771
── IPv4 ──

local source port: 53486

  destination                            external addr             
  ────────────────────────────────────────────────────────────────
  b10c.me:7770 (x.x.x.1)                 x.x.x.2:53486             
  b10c.me:7771 (x.x.x.1)                 x.x.x.2:53486             
  bnoc.xyz:7770 (x.x.x.3)                x.x.x.2:53486             
  bnoc.xyz:7771 (x.x.x.3)                x.x.x.2:53486             

classification: ENDPOINT-INDEPENDENT MAPPING (EIM)

  All four destinations saw the same external port. Your NAT maps (internal IP,
  internal port) to a single external port regardless of destination. TCP hole
  punching has a strong chance of working: a coordinator can reliably tell a
  peer which external port to send to.

── IPv6 ──

local source port: 53487

  destination                            external addr             
  ────────────────────────────────────────────────────────────────
  b10c.me:7770 (x::x:4)                  x::x:5:53487              
  b10c.me:7771 (x::x:4)                  x::x:5:53487              
  bnoc.xyz:7770 (x::x:6)                 x::x:5:53487              
  bnoc.xyz:7771 (x::x:6)                 x::x:5:53487              

classification: ENDPOINT-INDEPENDENT MAPPING (EIM)

  All four destinations saw the same external port. Your NAT maps (internal IP,
  internal port) to a single external port regardless of destination. TCP hole
  punching has a strong chance of working: a coordinator can reliably tell a
  peer which external port to send to.
```

So IPv4 TCP hole punching would not work at home nor via phone hotspot due to being APDM and IPv6 likely requiring opening the firewall, but would work when using Obscura VPN on both IPv4 and IPv6 as it's EIM.

I would be interested in seeing results from others.


|Who and what | IPv4 NAT | IPv6 NAT|
|--- | --- | ---|
|b10c at home & mobile hotspot | APDM | no NAT|
|Obscura VPN | **EIM** | **EIM**|
|@sipa at home | **EIM** | no NAT|
|@sipa using conference wifi | **EIM** | no IPv6|
|@willcl-ark via starlink (business local priority) | **EIM** | no NAT|
|@dunxen at home | **EIM** | no NAT|
|@cedarctic at university campus | APDM | -|

-------------------------

willcl-ark | 2026-05-11 11:05:03 UTC | #3

I also vibe-coded a "fun" holepunch program in python to test out the process a little bit, in case it's of interest to anyone else:

https://github.com/willcl-ark/natcat/

It has an optional `stun` command which will hit a [stuntman](https://stunprotocol.org/) server I am running to get your external IP address and port as viewed by a remote entity. (In bitcoin core we can get this info from our peers, if we don't already know it).

Swapping this info with a friend (or second machine) and using the `peer` command at ~ the same time on both instances will attempt a simultaneous connection.

`stdin` is sent to the other side, so you can send characters, files etc.

We had pretty decent results in testing, although we occasionally had to wait a few seconds for router firewalls to forget about their mappings (when we changed ip addresses but not ports), and were thwarted by one hotel NAT configuration.

As noted in the readme we also tried from within various types of containers, over starlink, through a VPN, and all of those chained together, and still saw success.

-------------------------

sipa | 2026-05-11 11:28:20 UTC | #4

I also vibecoded a demo application: https://github.com/sipa/holeroulette

A server is running, you can test with `./client.py 144.217.240.89` to be connected to a random other client.

-------------------------

dunxen | 2026-05-11 12:05:17 UTC | #5

Thanks for this. It’s a topic I’ve also been interested in regarding node reachability although from having messed around with other hole-punching tech like Iroh.
I’ll take some more time to read up, but just wanted to share the results running your script:

On my home internet connection:
IPv4: EIM;
IPv6: no NAT

-------------------------

0xB10C | 2026-05-11 13:14:16 UTC | #6

[quote="0xB10C, post:1, topic:2497"]
There also was brief discussion on how to make this more private by not requiring a coordnator Carol. Can Alice and Bob, once they’ve figured out their public IP:port behind a EIM NAT, use a e.g. Tor connection to communicate their IP:port to establish a clearnet connection. A protocol without a coordinator makes it a lot easier to reason about the connection either being inbound or outbound, and is likely a lot easier to implement.
[/quote]

I've been thinking about this a bit and think something like this might work:

A Bitcoin node run at home might not be reachable via clearnet due to NAT, however, many home nodes now support Tor and/or I2P, which allows inbound connections via Tor. It seems possible to coordinate TCP hole punching through Tor or I2P.

A node might want to offer inbound slots via clearnet, but is not reachable. One way to work around this could be the following protocol with node A and node B both being nodes behind a EIM NAT, while also being connected to the Tor and/or I2P network.

- Node A thinks it's unreachable (TBD how to figure this out reliably) but wants to offer clearnet inbound connections.
- It first needs to figure out if it's behind a EIM NAT. It can do so by opening and establishing two (possibly feeler?) connections to other nodes using `SO_REUSEADDR`/`SO_REUSEPORT` on the socket and checking if the peers return the same port in the version message. This needs to be done only once per network, assuming the NAT configuration does not change.
- Node A starts to listen on a **dedicated** hole-punch coordination Tor or I2P endpoint only for coordinating hole-punching. Address, transaction, or block relay is not supported on this endpoint. We don't link any other Tor or I2P endpoints that this node might have to this dedicated endpoint.
- This coordination Tor or I2P endpoint is advertised via `addrv2` message on **clearnet only** (to not link clearnet and Tor/I2P address) with a (new) service flag: e.g. `NODE_HOLEPUNCH`(?). These addresses are relayed as a best effort, but not stored in addrman. This has the goal of them not being relayed after around 10 minutes and not using up space in addrman. The frequency we relay these is TBD. We don't need to self-announce them, if we don't want any more holepunch-inbound connections. This message could also be a new BIP-155 address type for hole punching which could include the Tor or I2P coordination endpoint and the clearnet address (coordinate on `abcdef.onion`  to connect with `203.0.113.24`) the nodes wants inbound connections on. This allows other nodes to filter by e.g. netgroup / AS without needing to make a connection to the coordination endpoint.
- Node B receives such an announcement via an addrv2-message. It decides it wants to try to open an outbound connection to Node A. It has previously figured out that it's behind a EIM NAT.
- Node B connects to node A's hole-punch coordination endpoint, and does a version handshake. As part of the connection to this hole-punch coordination endpoint, a request for a hole-punch connection is implicit.
- Node A now opens a new outbound connection (e.g. a feeler?) to a known good address with `SO_REUSEADDR`/`SO_REUSEPORT` on the socket. The goal is to do a version handshake and learn the NAT `IP:port` of the socket. This likely requires some rate-limiting for some external party to cause Node A to make too many outbound connections.
- Node B does the same.
- Node A & B now have a EIM NAT mapped socket they can use to do the simultaneous TCP open and punch through their NATs.

A few notes on how to make this easier, but not without introducing tradeoffs:

- With EIM NATs, we might be able to predict our NAT port (it's often the same we bind on locally, at least from my limited observations). This might allow us to skip the outbound connections Node A and B need to make while already coordinating, making this a lot easier, faster, and causing less churn on the Bitcoin network. This will fail, when our NAT has already a mapping for the same port. The tradeoff here is possibly higher failure rates.
- An alternative might be to have well-known, static, (and centralized!) Bitcoin-protocol-speaking-servers-but-not-nodes run on the P2P network that just do a version handshake returning the IP:port they see your NAT IP from and then close the connection. These might be run by community members, similar to the DNS seeds. They'd learn about who is trying to open a hole-punch connection, and a passive observer (ISP) would see you making connections to them.
- Yet another alternative is that all nodes on the network would allow a special IPv4/IPv6 NAT check connection to them where it's implicit that the connection will be shut down right after the version handshake. No need for them to be centralized, but would basically require implementing a STUN service for the P2P network in node software (e.g. Bitcoin Core).

Compared to the approach with a coordinator, this has the benefit of having clear inbound-outbound mechanics. Node B chooses Node A as an outbound. Node B is an inbound to A. The downside is that we need connectivity over Tor/I2P/(CJDNS). However, many node-in-a-box home nodes seem to ship with Tor / I2P on by default.

-------------------------

davidgumberg | 2026-05-11 16:53:32 UTC | #7

A description of a reasonable design (that others came up with) for the Bitcoin P2P part of the rendezvous protocol that is over clearnet and requires a third coordinating node, but that might get a real outbound rather than something mutual-inbound:

1. Alice wants to receive inbound connections to her Bitcoin node but is behind NAT.
2. Alice learns through her ADDRMAN that Charlie runs a node that offers RENDEZVOUS services and opens a RENDEZVOUS-only connection to Charlie.
3. Alice uses a new ADDRv2 network type to advertise that she can be reached through Charlie.
4. Bob is looking for nodes to make an outbound connection to and decides to connect to Alice via RENDEZVOUS with Charlie.
5. All three use the procedure described [above](https://delvingbitcoin.org/t/tcp-hole-punching-for-bitcoin-nodes-behind-home-nats/2497#p-7486-tcp-hole-punching-1) to attempt a TCP-hole-punched connection from Bob to Alice.
6. If Alice still has inbound slots available she reconnects to Charlie and awaits new connections.

I am having a hard time reasoning about whether or not there is any trust assumption in Charlie, or if Charlie has any power he wouldn't have had otherwise. Since Charlie does get to decide whether or not Bob's connection to some chosen Alice succeeds iff Charlie is the rendezvous, Charlie seems to have some sway over Bob's outbounds, but how is this any different than if Charlie had advertised addresses that don't work?

If there is some trust in Charlie, this can be avoided by treating all addresses `Charlie` or routable via `Charlie` as one address, so Bob will make a random choice of e.g. `Charlie` to connect to then a second choice randomly from the pool of `Charlie`+`Routable_via_charlie`. In this case `Bob` can treat the connection as a real outbound, this achieves the effect of increasing the number of inbound slots on the network, but I think that the number of independent parties `n` on the network that you have a 1-of-`n` assumption for when bootstrapping onto the network remains the same.

And I think that the Tor/I2P approach described above is an idealized case of this where each node can serve as it's own `RENDEZVOUS`, but this would still have the problem that you have to be picking/trusting the tor/i2p rendezvous address, not the destination address, since an attacker could be spamming the network with fake/bad relays for real addresses.

-------------------------

cedarctic | 2026-05-12 14:46:56 UTC | #8

Reading up on the discussion here - hole punching seems like an interesting idea to increase network connectivity and robustness to network adversaries.

[quote="0xB10C, post:2, topic:2497"]
One can use [`nat-check.py`](https://github.com/0xB10C/tcp-nat-check/blob/15784f5d0897e5a3e38e9c9729fc39c60ef0d8fd/nat-check.py) to connect to these and get a classification of their NAT.

[/quote]

A campus network that I used for testing had APDM and varying external IP addresses. IPv6 was disabled.

[quote="0xB10C, post:1, topic:2497"]
Is there theoretical/academic research or P2P projects using this in the wild that we can learn from?

[/quote]

[This recent paper](https://arxiv.org/html/2604.12484v1#S3) claims a \~70% success rate with both TCP and QUIC (UDP-based transport) using libp2p’s DCUtR.

Thinking ahead if UDP hole punching proves to be much more effective, has there been any discussion in implementing QUIC or a custom UDP-based transport in core?

-------------------------

janb84 | 2026-05-12 14:35:09 UTC | #9

I wonder if this “hole punching” also works for VPN connections, making the node semi clear-net somewhat private.  (Proton has some [special NAT options](https://protonvpn.com/support/moderate-nat)) 

[quote="0xB10C, post:6, topic:2497"]
Compared to the approach with a coordinator, this has the benefit of having clear inbound-outbound mechanics. Node B chooses Node A as an outbound. Node B is an inbound to A.

[/quote]

I like this approach without a central  coordinator via TOR / I2C. Don’t think it’s the easiest option to get working.

-------------------------

