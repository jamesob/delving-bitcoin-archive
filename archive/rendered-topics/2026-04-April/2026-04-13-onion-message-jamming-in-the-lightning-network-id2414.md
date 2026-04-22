# Onion Message Jamming in the Lightning Network

erickcestari | 2026-04-13 17:13:42 UTC | #1

## Background

BOLT 4 acknowledges that onion message routing is inherently unreliable and recommends that implementations apply rate limiting. A common additional measure is to only relay onion messages from peers with whom the node shares a channel.

All implementations that currently support onion message forwarding enforce incoming rate limits and drop any messages that exceed the threshold. LND, which has not yet released onion message forwarding, will also include rate limiting. This protects the node but does nothing to protect the onion message network itself. None of the current implementations use the backpropagation-based approach proposed by t-bast (discussed in section 3 below). Here is a brief overview of how each implementation handles rate limiting today:

- **Core Lightning:** Token bucket that allows up to 4 onion messages per second per peer. Peers that exceed the limit receive a warning message and further onion messages are silently dropped until tokens replenish.

- **Eclair:** Hard cap of 10 onion messages per second per peer. Only receives and relays onion messages to/from peers with channels.

- **LDK:** Deprioritizes onion messages rather than using a strict rate limit. Channel messages (including ping/pong) are always sent first. Onion messages are only enqueued when the outbound buffer is empty, capped at 32 messages per processing tick.

- **LND:** Per-peer mailbox of 50 messages with Random Early Drop (RED). Messages are probabilistically dropped starting at 80% capacity (40 messages), with drop probability increasing linearly until all messages are dropped at full capacity. An [open PR](https://github.com/lightningnetwork/lnd/pull/10713) adds a two-tier token bucket rate limiter (per-peer and global) applied at ingress before cryptographic processing, dropping (rather than queueing) over-limit messages, and restricts onion message acceptance to peers with open channels.

This post originates from a discussion during the Lightning Network Spec Meeting on March 9th, 2026. During that meeting, TTLs for onion messages were floated as a possible mitigation, mainly against replays. While useful for replay prevention, TTLs do not directly address the flooding problem since an attacker generates fresh messages rather than replaying old ones. The mitigations discussed below focus on that core flooding vector.

## The Problem

The existing rate-limiting strategy is precisely what makes onion message jamming possible. An attacker can craft onion messages and propagate them across the network. Each intermediate hop requires a minimum of 86 bytes of payload (see Annex A). While BOLT 4 suggests two standard onion message sizes, 1,366 bytes (16 hops) and 32,834 bytes (382 hops), it does not enforce a maximum length. The actual upper bound is the Noise Protocol's 65,535-byte message limit, allowing a single onion message to span up to 761 hops (see Annex A).

By spinning up nodes and opening channels, the attacker floods the network with spam onion messages, triggering rate limits across enough nodes that legitimate messages are silently dropped alongside the spam. The high hop count makes this worse: because onion messages use source routing and intermediate nodes cannot inspect the full path, an attacker can craft a single message whose hops alternate back and forth between two victim nodes (e.g., Victim 1 -> Victim 2 -> Victim 1 -> Victim 2 -> ...), bouncing over 500 times within a single worst-case message. Each victim sees the other as the source of the flood, so per-peer rate limiting is applied against the other victim rather than the attacker. One crafted message causes amplified damage on the link between two honest nodes.

![Bounce amplification attack: the attacker crafts an onion message that loops between two victim nodes, causing them to rate-limit each other]
![image|690x258](upload://7VSbr7COwLvQ5uQ8EcH3og5Wwlj.png)

## Mitigation

Simply limiting the maximum number of onion hops does not fix the issue on its own, as the attacker just needs more entry points to achieve the same congestion. However, it does help by raising the cost of an attack, which is the same approach Tor adopted to mitigate similar flooding. It works best when combined with other mitigations. After reviewing several proposals, here are the four I believe best address this issue:

### 1. Upfront Fees (Per-Message Unconditional Payment)

Introduce a cost for sending onion messages, making large-scale flooding economically impractical. **Carla Kirk-Cohen's upfront HTLC fee proposal ([lightning/bolts#1052](https://github.com/lightning/bolts/pull/1052))** provides the most promising foundation. Originally designed for fast channel jamming, the mechanism extends naturally to onion messages, avoiding separate anti-jamming systems for payments and messages.

For HTLCs, nodes advertise an unconditional fee as a percentage of their success-case fees via a new TLV in `channel_update`, paid at each hop regardless of whether the payment succeeds. Simulations show 1% is sufficient, capped at 10% to prevent nodes from intentionally failing payments to collect the upfront portion. For onion messages, nodes advertise a flat per-message fee instead, included in the onion payload and deducted at each hop. Nodes that receive a message with insufficient fees for the next hop simply drop it. Notably, under a spam attack the targeted nodes actually profit from forwarding fees rather than suffering from it.

**Settlement.** No new protocol messages are needed. The onion message carries the fee in its encrypted per-hop payload, so both peers know the exact amount. The forwarder constructs the updated commitment transaction (crediting itself the fee) and sends `commitment_signed`. The sender confirms with `revoke_and_ack`, at which point the forwarder relays the message. If the sender does not confirm, the forwarder does not relay. Alternatively, peers could relay immediately and settle owed fees in a single `commitment_signed` after a message count or time interval, at the cost of per-channel fee accounting until settlement. No HTLCs are needed: the onion message acts as an implicit channel state update. This generalizes to channel jamming, where the only difference is that an HTLC output is also added to the commitment.

**Spec changes required:** (1) A new TLV in `channel_update` for the flat per-message onion forwarding fee. (2) A new TLV in the onion message per-hop payload (`encrypted_data_tlv`) carrying the fee for that hop. (3) A `channel_id` field in `onion_message` so the forwarder knows which channel to settle against.

**Limitations and tradeoffs:** A sufficiently funded attacker can still pay the fees, though at a much higher cost than today's free flooding. Coupling onion messages to the commitment dance means forwarding depends on channel liquidity and state machine availability, adding complexity to what is currently a stateless relay. This also makes the existing practice of only forwarding to channel peers a hard requirement, since peers without a channel cannot settle the fee. Per-message settlement also increases p2p overhead: today forwarding is a single message, but with upfront fees it becomes `onion_message` + `commitment_signed` + `revoke_and_ack` at every hop (there are actually two `commitment_signed` and two `revoke_and_ack` to complete the dance, but those are at least an order of magnitude smaller than onions). The bigger cost is latency. Without fees it is 0.5 round trips (`onion_message ->`). With upfront fees it is 1.5 round trips (`onion_message + commitment_signed ->`, `<- revoke_and_ack + commitment_signed`, `revoke_and_ack ->`). Under heavy load the last half trip can be combined with the next onion message (`revoke_and_ack + onion_message + commitment_signed ->`), bringing it down to ~1.0 round trips, a ~2x latency increase. Batching reduces the frequency of settlement but adds complexity.

[https://github.com/lightning/bolts/pull/1052](https://github.com/lightning/bolts/pull/1052)

[https://eprint.iacr.org/2022/1454.pdf](https://eprint.iacr.org/2022/1454.pdf)

[https://research.chaincode.com/2022/11/15/unjamming-lightning/](https://research.chaincode.com/2022/11/15/unjamming-lightning/)

### 2. 3-Hop Limit + Proof-of-Stake Based on Channel Balances (Hard/Soft Leash)

This approach, proposed by Bashiri and Khabbazian at the University of Alberta (Financial Cryptography 2024), has two components.

**Component 1, leashing the hop count:**

- **Hard leash:** A strict maximum hop count (e.g., 3 hops). The rationale draws from Tor, which achieves meaningful anonymity with just three hops, though it allows expanding up to 8 hops when needed. Since onion messages route through peers rather than channels, most nodes are reachable within a short hop distance. Requires changes to the onion message format to embed and verify the hop limit.

- **Soft leash:** Instead of a strict limit, the sender must solve a proof-of-work challenge whose difficulty scales exponentially with hop count. Each node announces a PoW difficulty target via gossip, adjusting dynamically based on incoming message rate. This preserves flexibility for longer paths while making high-volume long-path spam prohibitively expensive. Can be adopted without altering the onion message format.

**Component 2, proof-of-stake forwarding rules.** Instead of uniform rate limits, each node sets per-peer rate limits proportional to that peer's aggregate channel balance (as advertised via gossip): `αA × FB`, where `αA` is a tunable parameter and `FB` is the sum of capacities of channels owned by B. Well-capitalized nodes earn higher forwarding allowances, while underfunded attacker nodes get minimal throughput. The paper demonstrates that an adversary cannot meaningfully degrade the service unless they control a significant fraction of total network funds.

**Limitations and tradeoffs:** A 3-hop limit shrinks the sender's anonymity set, and the Lightning Network's hub-concentrated topology makes origin inference easier than in Tor. The proof-of-stake component advantages large established nodes and may create centralization pressure. An attacker could open large channels solely to inflate their gossip-visible balance, with capital lockup as the only cost. The soft leash adds computational overhead for honest senders needing longer paths. Nodes with only private (unannounced) channels would have zero gossip-visible capacity, yielding a rate limit of zero and shutting them out of onion message forwarding entirely, including privacy-conscious users and mobile nodes.

[https://ualberta.scholaris.ca/items/245a6a68-e1a6-481d-b219-ba8d0e640b5d](https://ualberta.scholaris.ca/items/245a6a68-e1a6-481d-b219-ba8d0e640b5d)

### 3. Bandwidth Metered Payment (Paid Onion Messaging Sessions)

Proposed by roasbeef, this approach also makes forwarding compensated and flooding expensive, like upfront fees. The key difference is the payment model: upfront fees require no application-layer state (no session IDs, no bandwidth counters) and settle per-message through the existing channel state machine, while bandwidth metered payment adds per-session state (~40 bytes per session for tracking session IDs, expiry, and remaining bandwidth) and settles once upfront via an AMP payment.

**How it works:** Inspired by HORNET's two-phase design, the sender first sends an AMP payment that drops off fees at each intermediate hop (priced via `sats_per_byte` and `sats_per_block` rates advertised in `node_announcement`) and delivers a 32-byte `onion_session_id` along with an expiry height. The receiver accepts by pulling the payment or rejects by failing any HTLC split. Once accepted, the sender includes the `onion_session_id` in the `encrypted_data_tlv` of subsequent onion messages. Forwarding nodes check session validity and remaining bandwidth before relaying, adding roughly 40 bytes of per-session state.

**Limitations and tradeoffs:** The sender can use distinct session IDs per hop (since they are inside the per-hop `encrypted_data_tlv`), so colluding nodes cannot correlate messages by session ID alone. However, at any single hop, all messages sharing the same session ID are linkable, unlike stateless onion messages where every message looks independent. Payment and forwarding are not atomic, so a node could take the payment and refuse to forward, though tit-for-tat (small sessions first) mitigates this. Requires a channel between every hop for AMP settlement, unlike base onion messages. Setting up a new session incurs a similar round trip overhead as upfront fees, since the AMP payment must complete the commitment dance at each hop. However, this cost is paid only once per session; subsequent messages within the session are forwarded without additional settlement until the prepaid bandwidth is exhausted.

[https://lists.linuxfoundation.org/pipermail/lightning-dev/2022-February/003498.html](https://lists.linuxfoundation.org/pipermail/lightning-dev/2022-February/003498.html)

### 4. Backpropagation-Based Rate Limiting (`onion_message_drop`)

Proposed by t-bast after discussions with Rusty Russell at the Oakland Dev Summit, this scheme uses a lightweight backpressure mechanism that statistically traces spam back to its source.

**How it works:** Nodes apply per-peer rate limits on incoming onion messages (e.g., 10/second for channel peers, 1/second for non-channel peers). When relaying, a node stores only the `node_id` of the last sender per outgoing connection.

When a message exceeds the rate limit, the receiver sends an `onion_message_drop` back to the sender, which identifies the last peer that forwarded on that link and relays the drop signal backward, halving that peer's rate limit. If the peer stops overflowing, the rate doubles every 30 seconds until it returns to the default.

The `onion_message_drop` includes a `shared_secret_hash` (BIP 340 tagged hash of the Sphinx shared secret), allowing the original sender to recognize when the drop propagates back to them and retry via a different path.

**Limitations and tradeoffs:** Since each node only stores the _last_ incoming `node_id` per outgoing connection, the drop signal may sometimes hit the wrong peer, though the correct sender is statistically penalized proportionally. The mechanism is reactive: legitimate users experience degraded service before backpressure takes effect. The attacker pays nothing beyond channel opens, so a persistent adversary can sustain low-grade degradation. A malicious node could also send fake `onion_message_drop` signals to artificially halve peers' rate limits and suppress legitimate forwarding without actual congestion. The bounce amplification attack described in the problem section is particularly effective against this scheme, as it directly weaponizes the backpressure mechanism against the victims.

[https://lists.linuxfoundation.org/pipermail/lightning-dev/2022-June/003623.html](https://lists.linuxfoundation.org/pipermail/lightning-dev/2022-June/003623.html)

[https://gist.github.com/t-bast/e37ee9249d9825e51d260335c94f0fcf](https://gist.github.com/t-bast/e37ee9249d9825e51d260335c94f0fcf)

## Conclusion

Each of the four mitigations addresses a different aspect of the problem. Upfront fees and bandwidth metered payment both make flooding expensive by compensating forwarding nodes, but differ in mechanism: upfront fees are stateless and per-message, while bandwidth metered payment uses stateful session-based bulk prepayment better suited for sustained communication. Hop leashing with proof-of-stake limits the attacker's reach and ties forwarding capacity to economic commitment. Backpropagation-based rate limiting offers a lightweight, reactive defense that requires no payment infrastructure. Each comes with meaningful tradeoffs and differences in implementation and deployment complexity.

LND has completed onion message forwarding as part of its BOLT 12 roadmap, with the final PR now merged and awaiting release. Once released, all major implementations will support the protocol, significantly expanding the attack surface. If BOLT 12 becomes the standard method for invoice requests, offers, refunds, and asynchronous payments, a sustained jamming attack would cause a severely degraded user experience.

Channel jamming illustrates how difficult it is to retrofit mitigations once a vulnerability is well established. Significant research and BOLTs proposals are underway, but reaching consensus and deploying a solution across all implementations takes time. Tor faced a similar challenge when a [prolonged DDoS attack](https://blog.torproject.org/tor-network-ddos-attack/) degraded its network for months in late 2022, requiring defenses to be built under pressure. With onion message support now reaching full network coverage, we have a window to design and ship mitigations early, and we should take it.

Feedback is welcome on which approach (or combination of approaches) is most viable, and whether there are alternative directions not considered here.

*Acknowledgments: Thanks to Matt Morehouse and Gijs van Dam for reviewing drafts of this post.*

---

## Annex A: Maximum Hop Count Derivation

Each intermediate hop in an onion message requires a minimum of **86 bytes** of payload, broken down as follows:

|Component|Bytes|Notes|
|---|---|---|
|BigSize length prefix|1|Length of the per-hop payload|
|`encrypted_recipient_data` TLV wrapper|2|1 byte type + 1 byte length|
|Encrypted blob|51|35 bytes ChaCha20-Poly1305 ciphertext (encoding the `encrypted_data_tlv` with the 33-byte `next_node_id`) + 16-byte Poly1305 authentication tag|
|HMAC|32||
|**Total**|**86**||

BOLT 4 suggests two onion message sizes (1,366 and 32,834 bytes) but does not enforce a maximum length. The actual upper bound is the **Noise Protocol's maximum message size of 65,535 bytes**. An attacker can craft arbitrarily large onion messages up to this limit, maximizing the number of hops and therefore the amplification of a single spam message.

However, the onion packet is nested inside a Lightning message structure. The full Noise message contains:

|Field|Bytes|Notes|
|---|---|---|
|Message type (513)|2|Lightning message type identifier|
|`blinding_point`|33|Route blinding point, separate from the onion packet|
|`onion_routing_packet` length|2|u16 length prefix|
|Packet version|1|Onion packet header|
|Packet `public_key`|33|Onion packet header|
|`hop_data`|N|Onion payload (raw bytes, no length prefix)|
|Packet HMAC|32|Onion packet header|
|**Total**|**103 + N**||

The onion packet header accounts for 66 bytes (1 version + 33 public key + 32 HMAC), and the enclosing Lightning message adds another 37 bytes (2 message type + 33 blinding point + 2 length prefix). The available hop data is therefore: 65,535 − 103 = **65,432 bytes**.

| Packet size | Hop data bytes | Intermediate hops | + Final hop | **Total hops** |
|---|---|---|---|---|
| 1,366 bytes (suggested) | 1,300 | 15 | 1 | **16** |
| 32,834 bytes (suggested) | 32,768 | 381 | 1 | **382** |
| **65,535 bytes (worst case)** | **65,432** | **760** | **1** | **761** |

In the worst case, a single onion message can fan out across **761 hops**, nearly doubling the amplification factor compared to the largest suggested packet size.

-------------------------

tnull | 2026-04-14 07:41:43 UTC | #2

Thank you for putting together the survey and keeping the discussion going! With LND finalizing onion message forwarding support I concur that it’s about time that we find a way forward and start implementing further mitigations for potential OM spam attacks. 

To me, Bastien’s backpressure proposal seems like a reasonable next step, in particular as 

1) it’s likely not a enormous lift (making it unlikely all implementations actually adopt it in a reasonable timeframe)

2) it’s somewhat backwards compatible and not super invasive to other parts of the protocol

3) it doesn’t seem to conflict with further mitigation steps. So, in case we some time down the road we don’t deem it sufficient, we can still add any of the other mitigation strategies on top

-------------------------

t-bast | 2026-04-14 08:33:03 UTC | #3

I agree, the other proposals have a much higher cost in terms of complexity and impact on honest users, so I’d prefer starting with the simple back-pressure algorithm and see what we learn from there. But it’s good to have other alternatives sketched out in case we need them!

-------------------------

erickcestari | 2026-04-14 13:41:24 UTC | #4

I agree that the other approaches are far more complex and that a simple mitigation like backpressure would be preferable. Unfortunately, as I noted in the limitations and tradeoffs, the mechanism can be weaponized by the attacker and actually leave the network worse off than having no backpressure at all. It's worth exploring ways to harden the backpressure mechanism so it can't be weaponized by attackers. However, I can't think of a way to prevent this without breaking onion routing's anonymity and unlinkability guarantees.

-------------------------

nothingmuch | 2026-04-15 20:04:57 UTC | #5

>  Coupling onion messages to the commitment dance means forwarding depends on channel liquidity and state machine availability, adding complexity to what is currently a stateless relay. This also makes the existing practice of only forwarding to channel peers a hard requirement, since peers without a channel cannot settle the fee.

fees for onion traffic with good privacy can be instantiated using an optional e-cash mechanism that can be used to bypass any rate limiting/caps.

the improvement over upfront fees as described here is that that would decouple onion forwarding from channel liquidity, and allow forwarding to/from non channel peers. it would still require stateful processing to detect double spending (though arguably some low probability of a false negative in double spend detection can be tolerated, relaxing the state consistency requirements).

the downside is that this increases complexity (cryptographic code, new type of node state, some engineering challenges). that said, it seems like this has potential to simplify as well, by removing some of the constraints on where fees can be used, so i thought it's worth describing here.

suppose alice has a channel to bob, bob has a channel to carol, carol has a channel to dave to whom alice wants to send an onion message.

alice makes a regular lightning payment to bob and in exchange he issues e-cash tokens redeemable only with him. note that in general bob doesn't have to share a channel with alice.

alice has to trust that bob will honor redemptions. the total balance represented in e-cash should therefore be minimized, on the order of a the expected fee cost of a handful of payments or some small amount of onion traffic. if this below the dust threshold arguably this doesn't change the threat model significantly.

with small amounts it's ok to assume these tokens cannot be converted back into channel funds, to avoid any custodial balances and the concerns they raise. that would make these tokens purely utility tokens, i.e. only usable for services with bob's node, like paying for onion bandwidth or locked up liquidity.

one such service would be purchasing of a neighboring node's e-cash tokens: alice redeems bob tokens in exchange for bob purchasing carol tokens on her behalf. this should incur some fee.

with tokens from both bob and carol, alice can now pay bob to forward an onion message to carol, which then pays carol to forward the nested message to dave. there can be more than one carol-like hop (no direct connection to alice).

there is no requirement to maintain a balance with more than a minimal set of nodes. tokens along a particular route can be obtained just in time. if some are left over, they can be used to purchase tokens in the backwards direction, so if alice has carol tokens left over, she could convert those back into bob tokens. note that this is more about efficiently managing liquidity and minimizing total outstanding token balance, it does not significantly reduce the trust assumption even though channel peers are likely to be more trustworthy (alice still trusts carol to forward traffic or allow purchase of bob tokens effectively refunding).

there are two concrete choices for the building blocks of such an e-cash protocol:

1. blind signature or blind Diffie-Hellman (like cashu / privacy pass). if a single denomination is used, where 1 token = 1 message, that is very simple cryptography and also imposes no latency overhead since each token presented is simply fully redeemed. the cryptography involved here is fairly straightforward, using only basic sigma protocols.

2. anonymous credentials and homomorphic value commitments. keyed verification anonymous credentials are the simplest choice, since the issuer is the verifier no publicly verifiable scheme is required. this increases latency, since "change" credentials must be issued. the cryptography needed for this is substantially higher in complexity due to the need for range proofs.

given the tradeoffs, a hybrid approach, where homomorphic value based e-cash denominated in msats could be used used to blind DH tokens that used more like stamps is also possible. this increase in complexity would allow both more flexible dynamic pricing, and minimum latency.

although using using multiple denominations for arbitrary balances with blind DH e-cash naively presents some challenges for privacy that homomorphic value commitments categorically avoid, these can be mostly be mitigated for this use case, so a hybrid approach could also be realized with just blind DH type e-cash (no scripting mechanism as in cashu is necessary or appropriate, nor publicly verifiable tokens based on blind signatures so this would be simpler than either cashu or privacy pass) by imposing some relatively simple constraints allowing the additional cryptographic complexity of anonymous credentials or homomorphic value commitments to be avoided. the downside of this variation is that it's more specialized for this use case, a homomorphic value commitment based (hybrid) approach could be reused (e.g. for encrypted storage, PIR blockchain queries, watchtowers, ...) so it may still be desirable.

-------------------------

morehouse | 2026-04-16 22:53:12 UTC | #6

## Central Planning vs Market Pricing

Fundamentally this is an economic problem.  Routing nodes expend resources (bandwidth, compute) to forward onion messages and currently are not compensated for those resources.

We could use *central planning* to decide who gets to consume those resources -- rate limits, backpropagation, and other attempts at "fair" resource allocation.  But as with all rationing systems, there are inevitably inefficiencies: legitimate usage gets throttled, edge cases are penalized, and attackers find ways to game the arbitrary quotas.

The alternative is to use *market pricing* to allocate these scarce resources -- upfront fees, blinded tokens, PoW, etc.  With this approach, protocol devs provide the mechanism for pricing and paying for onion forwarding and let the free market do the rest.

I am very reluctant to go further down the central planning route.  It's very difficult to address all attack vectors that way, and we'll just end up delaying the proper market solution.

## Dual Onion/Channel-Jamming Solution

Onion jamming is conceptually identical to the "fast" channel jamming attack coined by @carla and @ClaraShk.  In fast channel jamming, the attacker sends lots of quickly-failing payments across the network, preventing honest traffic from using the available HTLC slots.  In onion jamming, the attacker sends lots of onion messages across the network, preventing honest traffic from using the available message quotas.

For years now it's been widely accepted that upfront fees are the most promising mitigation for fast channel jamming.  Now we have another use case for them -- onion jamming.  Perhaps we finally have enough motivation to [implement](https://github.com/lightning/bolts/pull/1052) upfront fees and fix both of these problems!

-------------------------

Abdulkbk | 2026-04-17 11:37:45 UTC | #7

Excellent summary.

The backpressure proposal seems like a reasonable next step to me, but as you rightly pointed out, it can be weaponized against victims of the bounce amplification attack. 

One thing I kept coming back to is whether the `onion_message_drop` signal ever reaches the attacker in the bounce amplification scenario. I don’t think it does because each node stores only the last node_id per outgoing connection, and the bounce path is just V1 ↔ V2 repeating, the drop signal gets trapped in the same loop as the original message. V1 attributes the flood to V2, V2 attributes it to V1, and the signal never walks back to the attacker, who appears only at one of the very previous hops.

One idea worth discussing: a stricter, separate rate limit on `onion_message_drop` signals per peer (e.g., 1/sec). This would cap mutual blame escalation between victims and prevent the drop loop from spiraling. The limitation is that the real damage, victims rate-limiting each other’s onion messages, occurs before any drop signals are sent, and the attacker remains unaffected.

-------------------------

AdamISZ | 2026-04-21 16:05:46 UTC | #8

I'd just like to back this up.

The more I've looked at this problem in a number of vaguely similar contexts - anonymous or pseudonymous participants, with payment desired to present DOS *and* the snooping that can come from DOS (in LN case, probing etc.) - the more I've come round to the idea that, while this "2 layer payment" system looks ugly in complexity, it's somehow the *right* way to do it. The separation of the buying event from the payment event afforded by these ecash credentials is exactly what you want, I think. It actually gives you more flexibility to use cryptographic primitives that suit this exact problem, rather than the ones in e.g. bitcoin (which LN inherits) which are not particularly suited to this problem.

I do realize it looks heavy (per node economic relationship etc) but I suspect it's less so than it seems (as @nothingmuch already argued).

-------------------------

carla | 2026-04-21 19:07:44 UTC | #9

Dropping in to add perspective from HTLC jamming. I haven't deeply reviewed sections 2-4.

> avoiding separate anti-jamming systems for payments and messages.

* AFAIK, one of the original design considerations of onion messages is that we *don't* have to go to disk every time we send them, so attaching an unconditional fee to each one defeats the point? 
* *If* I'm right about that, we'd need to go with the batching approach you've suggested, which wouldn't be a good fit for HTLCs, where we'd just include the unconditional fee in `update_add_htlc`.
* In this case, I'm pretty confident we'd need separate protocol mechanisms - while the concept of pushing funds unconditionally is the same, `update_add_htlc` vs some batching scheme would have separate messages/exchanges/state tracking.

**On the relation to unconditional fees for HTLCs:**

While both are subject to "type 1" spam, onion messages and HTLCs are quite different domains. With HTLCs, nodes have the incentive to forward the payload on in the hope that they'll earn the remaining "success case" fee if the payment succeeds. For onion messages, your counterparty can just just drop the message and pocket the fee because there's no future benefit to be earned [1].

We also have a measurable loss when HTLCs are crowded out (other, fee paying payments) which makes it easier to understand how to set these policies. We don't have an "in-protocol" way to do this for OMs, so would likely have to shove this concern onto the end user and/or set a magical global default.

Also a meta note: I think it's pretty easy to get lost in the details of how we'd implement an unconditional payment (staking/ecash/tokens etc.). I personally found it helpful to start with asking whether this is the right tool for the job and how it would be used in the protocol, and _then_ look at whether all the fun and fancy options that _aren't_ native lightning offer us an improvement that justifies the complexity.

--- 

To be clear, I'm definitely in favor of *less* trivial DOS vectors in the protocol, not more! But I do think it's important to look at unconditional fees specifically in the context of onion messages, rather than porting over conclusions from the context of HTLCs. 

##### Footnotes
[1] In theory, forwarding an onion message _could_ mean that in future you'll forward a HTLC related to the onion message that you previously forwarded but there's no guarantee of this. I also don't see people feeding onion message routing into payment pathfinding anytime soon (other than an online check), as our primary concern is liquidity.

-------------------------

ariard | 2026-04-21 21:17:41 UTC | #10

See the Bitcoin Optech entry on staking credentials to solve jamming (https://bitcoinops.org/en/newsletters/2022/11/30/#reputation-credentials-proposal-to-mitigate-ln-jamming-attacks).The o’reilly p2p design book chapter on accountability (https://www.freehaven.net/doc/oreilly/accountability-ch16.html) is also a classic.

I’m not going to comment more on the subject, but some kind of monetary credential protocol would be very useful for the btc ecosystem, and given I’m interested in it for some bitcoin backbone experiments, I’ll likely go to implement one anyway.

-------------------------

morehouse | 2026-04-21 22:43:43 UTC | #11

[quote="carla, post:9, topic:2414"]
* AFAIK, one of the original design considerations of onion messages is that we *don’t* have to go to disk every time we send them, so attaching an unconditional fee to each one defeats the point?
[/quote]

Is that an essential part of the original design though, or was it a DoS-mitigation mechanism?  If we instead mitigate DoS/jamming via a monetary solution, perhaps we don't need onion messages to be as lightweight.

I'd also note that a completely jammed onion network also defeats the original point (to an even greater degree), so this may be a worthwhile tradeoff overall.

[quote]
* *If* I’m right about that, we’d need to go with the batching approach you’ve suggested, which wouldn’t be a good fit for HTLCs, where we’d just include the unconditional fee in `update_add_htlc`.
[/quote]

We *can* do batching, but I'd rather not if we don't have to.  There's less state to manage without batching, and the whole protocol becomes simpler (and overlaps more with up-front fees for HTLCs).

[quote]
**On the relation to unconditional fees for HTLCs:**

While both are subject to “type 1” spam, onion messages and HTLCs are quite different domains. With HTLCs, nodes have the incentive to forward the payload on in the hope that they’ll earn the remaining “success case” fee if the payment succeeds. For onion messages, your counterparty can just just drop the message and pocket the fee because there’s no future benefit to be earned [1].
[/quote]

The future benefit is getting to route *more* onion messages in the future.  Nodes that drop messages will be routed around, leading to less fees in the long run.

Also, as you mention in the footnote, sometimes the current onion message corresponds to a payment flow that will pay fees to the routing node in the *near* term.

[quote]
I also don’t see people feeding onion message routing into payment pathfinding anytime soon (other than an online check), as our primary concern is liquidity.
[/quote]

Why not?  If an onion message couldn't be routed across a path (tiny liquidity requirement, small resource usage), it seems highly unlikely that a payment could be routed on that path (larger liquidity requirement, larger resource usage, consumes a scarce HTLC slot).

[quote]
We also have a measurable loss when HTLCs are crowded out (other, fee paying payments) which makes it easier to understand how to set these policies. We don’t have an “in-protocol” way to do this for OMs, so would likely have to shove this concern onto the end user and/or set a magical global default.
[/quote]

I'd expect onion message fees to be adjusted automatically.  If the node is getting overloaded with onion messages, it raises its fees.  If it has unused capacity, it lowers its fees.  No need for static magic defaults or for the end user to do this manually.

-------------------------

carla | 2026-04-22 14:11:11 UTC | #12

[quote="morehouse, post:11, topic:2414"]
I’d expect onion message fees to be adjusted automatically. If the node is getting overloaded with onion messages, it raises its fees. If it has unused capacity, it lowers its fees. No need for static magic defaults or for the end user to do this manually.
[/quote]

How would we communicate these fees? Assuming that it won't be via gossip, as we're moving to one update per block in taproot gossip which wouldn't be nearly reactive enough to protect against a sudden influx of traffic. Will we add proper error propagation for onion messages to be able to communicate your latest fee policy?

This also sounds like something that could be abused by an attacker if we're not careful. I'd be interested in seeing an algorithm and some adversarial simulations that demonstrate how this would work!

[quote="morehouse, post:11, topic:2414"]
Why not? If an onion message couldn’t be routed across a path (tiny liquidity requirement, small resource usage), it seems highly unlikely that a payment could be routed on that path (larger liquidity requirement, larger resource usage, consumes a scarce HTLC slot).
[/quote]

I'd need to understand the details of the proposed scheme to answer this properly (specifically the above questions about dynamic fees). At the moment it's unclear to me whether an OM failure can always be interpreted as a liquidity failure, and if we allow different failure modes (like insufficient fees failures) how incentives for forwarding nodes to be honest play out. I'd also worry that feeding OM data into pathfinding could open up new channel jamming vectors.

-------------------------

