# TCP hole punching for Bitcoin nodes behind home NATs?

0xB10C | 2026-05-11 07:55:30 UTC | #1

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

-------------------------

0xB10C | 2026-05-11 09:56:46 UTC | #2

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


| Who | IPv4 NAT | IPv6 NAT |
|--- | --- | --- |
|b10c at home & mobile hotspot | APDM | no NAT (but likely firewalled) |
|Obscura VPN | **EIM** | **EIM** |

-------------------------

