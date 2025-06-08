# Latency and Privacy in Lightning

carla | 2025-05-23 17:44:47 UTC | #1

I was *ultimately* nerd sniped last LN spec meeting's discussion [0] of the privacy impact of surfacing granular HTLC hold times via attributable failures [1]. This post contains a recap of the discussion (as I understand it) and a summary of my sniping.

Recap of meeting discussion:
- The current version of the spec allows forwarding nodes to specify the time they held the HTLC in ms.
- It's likely that sending nodes will use this value in the future to pick low latency routes.
- Adding a random forwarding delay ((2], [3]) improves payment privacy.
- Surfacing hold times may dis-incentive this privacy-preserving delay as nodes race to the bottom to be the fastest.

The solution suggested in the meeting was to change the encoding to represent blocks time instead, so that the smallest encodable value still leaves time for processing and a random delay.
This can't be done by keeping ms encoding and enforcing some minimum, because nodes can always report smaller values; by changing the encoding, communicating a value under the smallest block of time becomes impractical [4].

Some questions that came up in the meeting:
- What value should we set this minimum to?
- How should we manage the UX/privacy tradeoff of fast payments vs forwarding delays?
- What happens if we need to increase forwarding delays in future? 

### Understanding Forwarding Delays + Privacy

To understand how these forwarding delays impact payment privacy, I took a look at a few research papers on the subject - summarized below. Of course, any inaccuracies are my own, I'd really recommend reading the papers to form your own opinion.

We are concerned about two different types of attackers:
1. **On path**: attacker creates channels, routes payments and attempted to deanonymize them.
2. **Off path**: attacker controls an AS, and is able to monitor messages at a network level

#### On Path Adversary

As outlined in [5]:
- Attacker probes the network to get latency estimates for nodes.
- Attacker opens up low-fee and low-expiry channels to attract channels.
- Recipient identification:
  - Record the time between `update_add_htlc` and `update_fulfill_htlc` 
  - Compare to latency estimates to calculate number of hops the HTLC took.
- Sender identification:
  - Only works if the sender retries along the same path.
  - Fail the first HTLC seen, and record time between `update_fail_htlc` and replacement `update_add_htlc`.
- Use amount and CLTV of HTLC to reduce set of possible senders/receivers.
- Use latency estimates to identify possible paths based on recorded time.

A *random* forwarding delay is helpful here because it interferes with the ability of the attacker to compare the time they've recorded with their latency estimates. In lay-carla's terms (give or take some noise), the delay is successful if it equals at least the processing time of a single hop, because this means that the attacker will be off by one hop and fail to identify the sender/receiver.

#### Off Path Adversary

As outlined in [6]:
- Attacker ICMP pings nodes in the network to get latency estimate.
- Attacker controls an AS and passively monitors network traffic.
- The commitment dance for a channel can be identified by message size and direction.
- With knowledge of the LN graph, an adversary can construct "partial paths" by tracing flow of `update_add_htlc` messages through the channels they observe.
  - This is timestamp based: if an incoming and outgoing `update_add_htlc` are processed within the estimated latency, they are assumed to be part of a partial path.
- Set limits for the possible payment amounts:
  - Minimum: largest `htlc_minimum_msat` along the partial path (can't be smaller than the biggest minimum). 
  - Maximum: smallest `htlc_maximum_msat` or `capacity` along the partial path (can't be bigger than the smallest channel).
- Perform a binary search to get a payment amount range:
  - Find the path from first to last node in the partial path for an amount.
  - If the computed path differs from the partial path, the amount is discarded.
- Remove channels that can't support the estimated payment amount.
- Identify sender and receiver:
  - Nodes that remain connected to the first/last hop in the partial path are candidate sender/receivers
  - Check payment path between each possible pair for the payment amount.
  - If the path uses the partial path, then the pair is a possible sender/receiver.

A forwarding delay is helpful here because it interferes with the ability of the attacker to construct partial paths. Notably, once these paths are constructed the attacker still has a large anonymity set to deal with, and the attack relies heavily on deterministic pathfinding at several stages to reduce this set.

[7] also examines how a malicious AS can identify nodes roles in a route with the goal of selective censorship:
- Senders: `update_add_htlc` messages sent "out of the blue" indicate that the node is the original sender.
- Intermediaries: timing analysis is used to connect an incoming `revoke_and_ack` with an outgoing `update_add_htlc` to identify forwarding nodes.
- Recipient: sending a `update_fulfill_htlc` message after receiving a `revoke_and_ack` message identifies the recipient, independent of timing.

Note that senders and receivers are identified based on the size of messages, without needing to rely on any timing information. Here, a forwarding delay isn't helping sender/receiver privacy at all - per the suggestions in the paper, it seems like message padding and possibly cover traffic are the most promising defenses.

### Incentives

While reading through all of this, it stood out to me that we're relying on forwarding nodes to preserve the privacy of senders and receivers. This doesn't seem particularly incentive aligned. Attributable failures and hold times aside, a profit driven node is incentivized to clear out payments as fast as it can to make efficient use of its capital. This seems sub-optimal on both ends:
- Senders and receivers who care about privacy can't hold forwarding nodes accountable for adding a delay, because these values *must* be random to be effective. If you see that nobody delayed your payment, it may have just happened to get a very low delay on each hop. 
- Forwarding nodes don't know how long a HTLC's payment route is, so they can't easily pick a good delay time that they're certain will help with privacy (unless they over-estimate, adding an additional hop's latency) [8].

Is there something better that we can do?

#### On Path Adversary

In this attack, the attacker depends on the time between `update_add_htlc` and `update_fulfill_htlc` to make inferences about the number of hops between itself and the recipient to deanonymize the recipient. It doesn't matter where the delay happens, just that there is enough delay for it to be ambiguous to the attacker how many hops there are to the recipient. It seems reasonable that we could implement delays on the recipient, instead of with the forwarding nodes. This puts the decision in the hands of the party whose privacy is actually impacted. It also works reasonably well with other hold-time aware systems like jamming mitigations and latency-aware routing, because we have to accommodate the MPP case where the recipient can hold HTLCs anyway.

For sender de-anonymization, the attacker needs to fail a payment and be on-path for the retry. This is more trivially addressable by adding a cool down between attempts and using more diverse retry paths. This is within the control of the sender, so it is nicely incentive aligned.

#### Off Path Adversary

While timing information is used in this attack, my impression from [6] was that predictable routing algorithms are what makes reducing the anonymity set feasible for the attacking node. This is again a level that we could provide the sender to toggle as they see fit rather than relying on forwarding nodes. Without the ability to prune the network, the anonymity set for this attack remains infeasibly large.

This attack also gets significantly easier for larger payments, as the attacker can prune more channels (that wouldn't be able to facilitate the amount). So more aggressive payment splitting is another option for privacy conscious sending nodes that does not rely on forwarding nodes for protection.

### What to do for attributable failures?

Practically in today's network, we don't have any privacy preserving forwarding delays deployed:
- LND (80-90% of public network): has a 50ms commitment ticker to batch updates, but it is not randomized so can trivially be accounted for in the attacks listed above [9].
- Eclair (major router): does not implement forward delays.

So we do not currently have any defenses against the above listed attacks implemented. And we should fix that!

My opinion is:

If we truly believe that forwarding delays are the best mitigation:
- We should all implement and deploy them.
- We should change encoding in attributable failures hold times to enforce minimum value.

If that's not the case (which I don't necessarily think it is):
- We should investigate and implement some of the suggestions listed above.
- It's fine to leave the attributable failures hold times encoded with millisecond granularity.

### Footnotes
[0] https://github.com/lightning/bolts/issues/1258#issuecomment-2892255704

[1] https://github.com/lightning/bolts/pull/1044

[2] https://github.com/lightning/bolts/blob/011bf84d74d130c2972becca97c87f297b9d4a92/04-onion-routing.md?plain=1#L301

[3] https://github.com/lightning/bolts/blob/011bf84d74d130c2972becca97c87f297b9d4a92/02-peer-protocol.md?plain=1#L2394

[4] Forwarding nodes could flip a high bit to indicate that they're using ms, but this would require sender cooperation and lead to devastating penalization if senders aren't modified (because it would lead to their hold time being interpreted as massive).

[5] https://arxiv.org/pdf/2006.12143

[6] https://ieeexplore.ieee.org/document/10190502

[7] https://drops.dagstuhl.de/entities/document/10.4230/LIPIcs.AFT.2024.12

[8] Yes, we may very well have an incredibly privacy conscious and altruistic routing layer. Even if that's the case (quite probably, since there isn't much money to be made with it), we shouldn't be relying on it to make privacy promises.

[9] Heavily emphasized across all papers is that this delay needs to be random to be impactful.

-------------------------

brh28 | 2025-05-23 20:28:07 UTC | #2

Thank you for taking the time to summarize this discussion.

As you mention, there is a fundamental trade-off between performance (latency) and privacy. While there *may* be a slight privacy improvement, I'm more concerned that this amplifies performance issues that already exist in Lightning today. Adding a forwarding delay effects every hop of every payment attempt. Not only does this slow the delivery of successful payments by `delay * hop_count`, but perhaps more concerning is it also delays failing payment attempts, both legitimate and malicious. As routing relies on trial-and-error, failed payments are expected. Worse yet, probing is a common trick to improve future reliability, which means we can expect the number of failed payments to grow exponentially with the number of nodes on the network. Adding delays to these failed attempts compounds the problem of locked liquidity and HTLC slots for routing nodes. 

So I agree, routing nodes have no incentive to follow these rules, other than as an optional feature to attract privacy-focused nodes. 

I like your idea of applying delays at the source and destinations:
1. It's opt-in and can be tuned to the user's preference
2. It's doesn't require protocol changes
3. For the receiver, it's safe to consider the payment successful, even while delaying HTLC fulfillment

-------------------------

t-bast | 2025-05-26 07:53:54 UTC | #3

Thanks for the detailed post and the insights! It does make a lot of sense: I was personally mostly worried about the AS case, where it's currently somewhat simple to match incoming `update_add_htlc` with the corresponding outgoing `update_add_htlc` based on timing and message identification. But as you mention, having cover traffic and padding messages to be indistinguishable by just looking at their size is probably a better (and more general) solution than delays for this kind of adversary.

We've known that padding messages was something we needed to do for a long time, and it became particularly useful since we introduced the `path_key` TLV to `update_add_htlc` messages, making them distinguishable from `update_add_htlc`s outside of blinded paths. The downside is of course that it uses more bandwidth, but we can't have our cake and eat it too. The 65kB limit ensures that we're still within a single TCP packet, which hopefully shouldn'd degrade performance too much. It would be a loss though if padding all messages to 65kB would actually degrade performance more than delaying HTLC messages! It could be interesting to do some simulations on real nodes (by turning on and off the message padding feature for various time periods) to figure this out.

Let's see what others think after reading your analysis, but to me it's a good enough argument to keep reporting the exact hold time in attributable failures.

-------------------------

ZmnSCPxj | 2025-05-26 20:23:54 UTC | #4

Last month I had a discussion about this what a few people.  Somebody pointed out that we deployed "HTTPS Everywhere" to improve the privacy of everyone on the web.

My counterpoint at the time was that "HTTPS Everywhere" could be imposed by user-agents and their operators, but there is nothing that would force forwarding nodes to create randomized forwarding times; senders and receivers cannot force forwarding nodes to perform the randomization.  This is equivalent to the observation by carla that it is the senders and receivers who have an incentive to randomize, not forwarding nodes.

My counterproposal was:

* Make ***batching of HTLCs*** the primitive, not individual `update_add_htlc`s.
* Create a new forwarding "receiver-enforced forwarding randomization" protocol:
  * New message `you_have_incoming_htlcs`.  This is sent if a node wants to *eventually* `update_add_htlc` ***one or more*** HTLCs.  The message has no body, and is replayed on reconnection.
  * New response `gimme_the_incoming_htlcs`.  This is sent after receiving `you_have_incoming_htlcs`.
  * New rules for `update_add_htlc`:
    *  it is an error for a node to send `update_add_htlc` unless it has received `gimme_the_incoming_htlcs`. (because it is an error, you should `error` if you receive an `update_add_htlc` without having sent `gimme_the_incoming_htlcs` first and drop all channels with that peer onchain)
    * A "batch" of `update_add_htlc`s MUST be sent in response to `gimme_the_incoming_htlcs`.  The batch is ended by a `commitment_signed`.  After sending `commitment_signed`, it is once again an error for the node to send `update_add_htlc` until it has received a new `gimme_the_incoming_htlcs`.

The above adds increased latency to the forwarding protocol, due to the additional `you_have_incoming_htlcs`/`gimme_the_incoming_htlcs` exchange.  A counter to this is that this protocol can be restricted to use only on endpoint receivers (i.e. receivers can use an even feature bit to enforce that this protocol is used in an "HTTPS Everywhere"-style campaign, while forwarders can provide an odd feature bit to indicate to new peers that they support this protocol, and if both of you use the odd feature bit you don't follow this protocol after all), and pure forwarders can use the original low-latency forwarding protocol with each other.

A receiver can, on receiving a `you_have_incoming_htlcs` message, then randomize the delay before sending `gimme_the_incoming_htlcs`.  This also allows the LSP of the receiver to batch multiple HTLCs to the receiver (e.g. probably helpful to improve throughput for multipath payments, which carla also noted would probably also help privacy in practice).

-------------------------

tnull | 2025-05-27 10:09:24 UTC | #5

Thank you Carla for this great write-up of the discussion! I agree with your analysis in general, here are just a few points I want to add:

- Yes, adding forwarding delays could be perceived to be unaligned with forwarding nodes' incentives, however, they already do this and also gain some efficiency and performance from batching HTLCs. Of course, there is a latency trade-off here, but generally it holds that the longer you wait, the higher are the chances that you can benefit from the reduced IO and network-latency overhead of batching HTLCs. IIUC, this would become even more relevant if we were to implement  `option_simplified_update` in the future.
- As always, "privacy loves company". So requiring individual nodes who think they need additional privacy protections to add random delays might help with the particular on-path adversary model in mind, but it could actually have them stand out more in the general case. I.e., depending on the adversary it could even put a crosshair on their back, at least if it doesn't become a best practice to add reasonable random delays before claiming (receiver-side) / retrying (sender-side) payments. So, if we agree sender/receiver side delays are the way to go, it would make sense to actually document best-practices that any implementation should stick to, just as we already do for the CLTV expiry delta in the [BOLT #7 "Recommendations for Routing"](https://github.com/lightning/bolts/blob/011bf84d74d130c2972becca97c87f297b9d4a92/07-routing-gossip.md#recommendations-for-routing) section.
- Note that in our research paper ([5]), we still assumed a purely BOLT11 world in which sender's receiver anonymity was non-existing, i.e., the sender would always know the full path and hence the identity of the receiver anyways. However, in a post-BOLT12/blinded path world, the receiver's identity can be actually hidden from the sender, and now the *sender* could be considered an on-path adversary. If we now report exact hold times of each intermediate hop to the sender, it might allow them to re-identify the receiver,  breaking the BOLT12 privacy gains we just finally introduced. But of course, if we consider this case, we'd also need to think about mitigations for the onion message parts of the protocol.

TLDR: Yes, I agree that receiver/sender-side delays could be an option, if they were documented as (~binding) best practices for implementations. That's mod the concerns regarding breaking blinded path privacy.

-------------------------

GeorgeTsagk | 2025-05-27 13:31:37 UTC | #6

Thanks for the write-up!

### Attributable Failures

I think changing the attr failure encoding to enforce hold_time related attributes isn't that absolute, routing nodes could manipulate the "protected" encoding to signal lower delays, e.g if we used uint8 hold times then `10001000` could be some slang for `16ms`, and this could break a theoretical floor of `100ms`. The sender of the payment also has to opt-in to this custom value interpretation.

We need to keep in mind that the reporting of the hold times as part of the attributable failures upgrade was just a placeholder that can prove useful in the future. It's not precise and certainly not reliable. Routing nodes can choose to lie or trim their hold times to make themselves look more attractive, and this inaccuracy would definitely be factored into the sender's pathfinding/scoring algorithm.

Seems as if we rushed ahead to assume that hold times are going to be the primary attribute to score a node by? This is definitely not covered by attr failure spec and I'm not sure if any discussion has started around how the values would be incorporated into pathfinding feedback.

We could have senders interpret all values below a threshold as if they were the same, so `87ms / 42ms / 99ms` would all be considered as `100ms / 100ms / 100ms`. Routing nodes are free to race to the bottom, but for the majority of the network which defaults to the above behavior it wouldn't make a difference.

### Off path adversary

Doing the LND-style commitment batching (maybe with greater & randomized intervals?) is attractive, but would definitely contribute towards slower payment attempts.

Since the cost of having timing related defenses is equally paid by payment senders, it's wiser to focus on the data/traffic obfuscation vector. Cover traffic sounds very promising, and can definitely help with muddying the waters for the adversary. This could also be an opt-in feature, controllable by the sender.

A sender-controlled approach could be having a mock path which doesn't change the commitment tx and follows an onion route which only triggers a `mock_add_htlc` message. This way for every real payment there would be X "mock payments" travelling over a somewhat related route, solely for the purpose of misleading the network-level adversary. A node receiving a `mock_add_htlc` knows that the only purpose of the message is to forward it to another peer.

Another simple approach could be having empty `mock_add_htlc` messages (no onion layers of any kind) with a `TTL` field, that nodes along the real payment path optimistically transmit (random receiver and delay included). A node receiving the mock message forwards to another random peer if TTL>0, decreasing the TTL by 1. The longest possible route here could also be a `mock_add_htlc` chain triggered by the last hop before the receiver.

All mock/obfuscation related messages should of course have their own processing budget and not interfere with channel related messages that are of higher priority.

### On path adversary

I don't have much to add here, the sender/receiver controlled delays seem to be a very nice angle to tackle the issue.

-------------------------

GeorgeTsagk | 2025-05-27 13:36:05 UTC | #7

> If we now report exact hold times of each intermediate hop to the sender, it might allow them to re-identify the receiver, breaking the BOLT12 privacy gains we just finally introduced. But of course, if we consider this case, we’d also need to think about mitigations for the onion message parts of the protocol.

For the blinded part of the path we don't have to report hold times (the forwarding node knows if it's part of one). Also the sender does not know which nodes make up the blinded path, so cannot assign blame / penalties anyway.

-------------------------

tnull | 2025-05-27 14:46:44 UTC | #8

[quote="GeorgeTsagk, post:7, topic:1723"]
For the blinded part of the path we don’t have to report hold times (the forwarding node knows if it’s part of one). Also the sender does not know which nodes make up the blinded path, so cannot assign blame / penalties anyway.
[/quote]

Yes, indeed I just discussed this offline with Joost. Here are a few conclusions as a follow-up:

- My concerns regarding BOLT12 privacy are indeed invalid as the introduction point would strip the attribution data. This in turn means that the next node upstream would report a huge latency measurement (as it would cover the entire blinded path's latency), which is of course bogus and would need to be disregarded during scoring.
- Similarly, any trampoline or legacy node in the path would also lead to stripped attribution data, meaning we'd only receive attribution data for hops before we encounter a blinded path, trampoline, legacy node in the path. 
- As the attribution happens on a per-edge basis there is no way to discern the second-to-last hop and the final recipient. This means that if we want to exempt the recipient from the blame to incentivize receiver-side delays, it would always need be the last two nodes on the path. This essentially also results in a similar rule for the sender-side scoring to disregard/throw away whatever the last attribution data entry it receives for any given the path.

Especially given that BOLT12/blinded paths (maybe even 2-hop?) might eventually become the default payment protocol, it seems hold time reporting will be limited to a prefix of any given path either way. This limits its usefulness, but also the impact it might have on privacy. 

So my personal conclusion is that we might be fine with hold times reporting, as long as we establish best practices around receiver-side delays and their exemption from sender-side scoring as mentioned above.

-------------------------

carla | 2025-05-27 19:25:31 UTC | #9

[quote="tnull, post:5, topic:1723"]
Yes, adding forwarding delays could be perceived to be unaligned with forwarding nodes’ incentives, however, they already do this and also gain some efficiency and performance from batching HTLCs. Of course, there is a latency trade-off here, but generally it holds that the longer you wait, the higher are the chances that you can benefit from the reduced IO and network-latency overhead of batching HTLCs
[/quote]

Indeed. I think being able to toggle these delays to decide on your IO/latency tradeoff makes a lot of sense :+1: afaik LND already allows this.

[quote="tnull, post:5, topic:1723"]
So my personal conclusion is that we might be fine with hold times reporting, as long as we establish best practices around receiver-side delays
[/quote]

Seems like a reasonable idea to me - and something that could be turned on by default to make sure that privacy gets some company!

[quote="tnull, post:5, topic:1723"]
 and their exemption from sender-side scoring as mentioned above.
[/quote]

Agree. Timing based scoring will already need to take into account the MPP timeout, so would already need a code path to allow this (just needs to be turned on for single-HTLC payments as well).

[quote="tnull, post:5, topic:1723"]
My concerns regarding BOLT12 privacy are indeed invalid as the introduction point would strip the attribution data. This in turn means that the next node upstream would report a huge latency measurement (as it would cover the entire blinded path’s latency), which is of course bogus and would need to be disregarded during scoring.
[/quote]

I think I'm missing the concern for BOLT 12 a bit here - could you spell it out for me why the introduction point needs to strip attribution data? 

I would have thought that the receiving node would just add its own delay (possibly breaking this delay up between any fake hops it added to the blinded route) and then report them back?

-------------------------

Crypt-iQ | 2025-05-30 16:26:04 UTC | #10

[quote="t-bast, post:3, topic:1723"]
The 65kB limit ensures that we’re still within a single TCP packet, which hopefully shouldn’d degrade performance too much. It would be a loss though if padding all messages to 65kB would actually degrade performance more than delaying HTLC messages!
[/quote]

I was looking into this recently to jog my memory -- TCP packets are fragmented based on the path's minimum MTU (PMTU) and then reassembly of TCP packets occurs. This means in practice TCP packets are limited to ~1500 bytes. See https://datatracker.ietf.org/doc/html/rfc8900#name-ip-fragmentation for more information if you have some time. That RFC links to another RFC (https://datatracker.ietf.org/doc/html/rfc4963) that describes an IPv4 fragmentation attack where a 3rd-party can spoof the 16-bit ID counter in the IP header and cause IP reassembly to fail (when validating the checksum) or pass with corrupted data (randomly passes the checksum which is also 16-bit).

If this is just a typo and you meant Lightning packet then please disregard. I am not sure of the overhead of reassembly of TCP packets, but from what I've read fragmentation seems to have some issues. Mistakes my own.

-------------------------

tnull | 2025-05-30 12:40:55 UTC | #11

[quote="carla, post:9, topic:1723"]
Seems like a reasonable idea to me - and something that could be turned on by default to make sure that privacy gets some company!

Agree. Timing based scoring will already need to take into account the MPP timeout, so would already need a code path to allow this (just needs to be turned on for single-HTLC payments as well).
[/quote]

Now opened https://github.com/lightning/bolts/pull/1263 to propose adding such recommendations to the BOLTs.

[quote="carla, post:9, topic:1723"]
I think I’m missing the concern for BOLT 12 a bit here - could you spell it out for me why the introduction point needs to strip attribution data?
[/quote]
It is my understanding that this is this is the currently proposed way how attributable failures would work in conjunction with blinded paths. Essentially, the sender can't learn anything about the blinded path, in particular not it length, let alone timing measurements per hop. For the sender the blinded path that it takes from the offer is opaque and acts it terms of scoring like a single monolithic hop itself. Maybe this should be made more explicit in https://github.com/lightning/bolts/pull/1044 (cc @joostjager)?

-------------------------

t-bast | 2025-05-30 12:51:12 UTC | #12

@Crypt-iQ you're completely right, thanks for highlighting this: I did mean TCP packet and assumed the best-case where it isn't fragmented by intermediate routers. But it was part of my open-ended question about possible performance degradation: I have no idea whether in practice 65kB TCP packets often get fragmented or not, how much overhead it adds, and couldn't find public data about this...if a lot of fragmentation happens, then padding could indeed lead to degraded performance (on top of the additional bandwidth usage increase).

[quote="Crypt-iQ, post:10, topic:1723"]
This means in practice TCP packets are limited to ~1500 bytes.
[/quote]

Is this a default OS behavior? Or is it just a recommendation? I'll read the links you provided when I have some time, but would love a TL;DR for now :slight_smile: 

I have no idea where to find actual data of what happens on the internet nowadays. I was thinking that we could do A/B testing on mainnet nodes: pad all packets to 65kB for a few days, then remove padding for a few days, and repeat, while measuring latency. This could give some indication of the overhead, even though it wouldn't let us accurately predict higher percentiles since there probably isn't enough payment volume today to provide meaningful statistics, but it's a start.

-------------------------

Crypt-iQ | 2025-05-30 15:05:54 UTC | #13

[quote="t-bast, post:12, topic:1723"]
Is this a default OS behavior? Or is it just a recommendation? I’ll read the links you provided when I have some time, but would love a TL;DR for now
[/quote]

I'm syncing a bitcoind node on my macbook and I can see in Wireshark that it is both fragmenting and reassembling packets greater than 1500 bytes (specifically `headers` messages). You can run `ifconfig` or similar on your machine and it will tell you MTU. Packets larger than 1500 bytes can be transmitted, but I believe this requires every router to handle this. I believe the 1500 byte limitation is a legacy thing and may vary with OS but seems to be pretty consistent from what I've seen. Hope I'm not link spamming too much but this post gives some history into the 1500 byte limitation (https://blog.benjojo.co.uk/post/why-is-ethernet-mtu-1500).

RFC8900:
- This was written recently (in 2020) and describes all of the different issues with fragmentation and reassembly of IP packets.
  - Some senders use something called Path MTU Discovery where ICMP packets are sent back to the sender so they can update their MTU estimate for a path. Usage of ICMP is not great because there is no authentication, can be rate-limited, black-holed, etc and I believe this means that in adversarial cases or even during regular operation, the sender may have to retry the send.
  - IPv6 has different fragmentation rules than IPv4 which seems to have some upsides but also may introduce some complications. It is less vulnerable to 3rd party IP reassembly attacks.
  - It notes that RFC 4443 recommends strict rate limiting of ICMPv6 traffic which may come into play during congestion.
  - Ultimately recommends that higher-layer protocols not rely on IP fragmentation as it's fragile.

RFC4963:
- This was written in 2007 and describes how IP reassembly works.
  - IPv4 uses a 16-bit ID field. The implementation "assembling fragments judges fragments to belong to the same datagram if they have the same source, destination, protocol, and Identifier". In the RFC, it gives an example time that the packet can be alive as 30 seconds. I'm not sure whether this is a TCP Maximum Segment Lifetime (MSL) value (depends on OS, defaults to 30 seconds in Linux) or an IP-related timeout. This has implications on a senders data rate as _technically_ only 65,535 1500-byte packets are valid in a 30-second window or whatever the time limit is.
  - IPv4 receivers store fragments in a reassembly buffer until all fragments are received or a reassembly timeout is reached. Configuring the reassembly timeout to be less has issues for slow senders but is better for fast senders. The opposite is also true when increasing the reassembly timeout.
  - The RFC describes a situation that can occur either maliciously or under high data-rates called "mis-association". This is where overlapping, unrelated IP packets are spliced together and then passed to the higher layer. Typically this will get caught by the TCP or UDP checksum, however it's only a 16-bit checksum and can occasionally be bypassed. Because of this, the RFC ultimately recommends the application layer to implement cryptographic integrity checks (which we do thankfully in both Bitcoin and Lightning).
  - Over UDP with 10TB of "random" data being sent, there were 8,847,668 UDP checksum errors and 121 corruptions due to mis-associated fragments (i.e. the UDP checksum was bypassed) and passed to the higher-layer.
  - From what I can tell (I have yet to test this), just because we have integrity checks in both Bitcoin and Lightning doesn't preclude an attacker from messing with our reassembly and causing delays even if they are not an AS and are just guessing two people are connected. The LN graph is public also which is a bit concerning.

[quote="t-bast, post:12, topic:1723"]
I have no idea where to find actual data of what happens on the internet nowadays. I was thinking that we could do A/B testing on mainnet nodes: pad all packets to 65kB for a few days, then remove padding for a few days, and repeat, while measuring latency.
[/quote]

Data is pretty hard to come by. I think testing on mainnet and observing traffic is probably your best bet. I think fragmentation can be pretty costly in the presence of errors since retransmission and reassembly has to occur again. But again I don't have hard data for this. It would be very interesting to see what other applications like Tor or something do when trying to send or receive large amounts of data at once.

-------------------------

tnull | 2025-05-30 14:19:33 UTC | #14

[quote="Crypt-iQ, post:13, topic:1723"]
I’m syncing a bitcoind node on my macbook and I can see in Wireshark that it is both fragmenting and reassembling packets greater than 1500 bytes (specifically `headers` messages). You can run `ifconfig` or similar on your machine and it will tell you MTU. Packets larger than 1500 bytes can be transmitted, but I believe this requires every router to handle this. I believe the 1500 byte limitation is a legacy thing and may vary with OS but seems to be pretty consistent from what I’ve seen. Hope I’m not link spamming too much but this post gives some history into the 1500 byte limitation ([How 1500 bytes became the MTU of the internet](https://blog.benjojo.co.uk/post/why-is-ethernet-mtu-1500)).
[/quote]

Might be worth noting that the actual payload size that can be transmitted without fragmentation is more like 1400 bytes to 1460 bytes   (1500 B - 20B (IPv4 header) or 40B (IPv6 header) - 20 to 60 B (TCP header)).

-------------------------

t-bast | 2025-05-30 15:06:54 UTC | #15

Thanks for the additional details, that's really helpful! I remember now that this is why onion packets have been chosen to be 1300 bytes, so that an `update_add_htlc` would fit inside the 1500 bytes MTU.

So the conclusion is that we should assume that TCP packets will be fragmented in 1500 bytes chunks, so I'm curious to see the impact of sending 65kB lightning packets, which will require quite a reassembly...It would be interesting to run this in simulations (SimLN can probably help here?) or try it out with A/B testing on mainnet.

-------------------------

Crypt-iQ | 2025-05-30 16:49:41 UTC | #16

Sorry, I think I've conflated TCP reassembly with IP reassembly. When the bitcoind node was syncing, I was seeing reassembled TCP segments. I don't think the IP fragmentation stuff applies here as I think most routers don't fragment IP packets. However, the limit @tnull pointed out still applies as TCP will break up packets into MTU size chunks to transmit as IP packets.

-------------------------

joostjager | 2025-06-02 09:00:06 UTC | #17

[quote="tnull, post:11, topic:1723"]
Maybe this should be made more explicit in https://github.com/lightning/bolts/pull/1044 (cc @joostjager)?
[/quote]

Added sentence to the bolt spec PR

-------------------------

MattCorallo | 2025-06-02 14:19:11 UTC | #18

[quote="carla, post:1, topic:1723"]
Intermediaries: timing analysis is used to connect an incoming `revoke_and_ack` with an outgoing `update_add_htlc` to identify forwarding nodes.
[/quote]

[quote="carla, post:1, topic:1723"]
Note that senders and receivers are identified based on the size of messages, without needing to rely on any timing information. Here, a forwarding delay isn’t helping sender/receiver privacy at all
[/quote]

These two statements appear to contradict each other - the point of the forwarding delay is to create a batch such that a passive network monitor can't trivially trace payments. Indeed, in some specific attacks in the literature they use message sizes and other issues in LN that we should address separately, but unless we want to switch to full CBR streams (at least for HTLCs themselves, in theory we could segment out our traffic in our TCP streams such that we send exactly one 1400-byte packet every 100ms or whatever even though we let gossip use whatever rate it wants), moving to no forwarding delay would leave payment tracing from someone doing network monitoring absolutely trivial.

Even if we switch to CBR (is there even appetite for doing this?), having no forwarding delays would mean that someone trying to get privacy by adding extra hops would be absolutely wrecked by an adversary running multiple nodes (post-PTLCs).

ISTM delays when forwarding (to the point of getting batching, at least insofar as we have enough traffic that we can get there without killing UX) is very important for privacy, especially as we fix low-hanging fruit like message padding.

-------------------------

carla | 2025-06-02 15:29:37 UTC | #19

[quote="MattCorallo, post:18, topic:1723"]
These two statements appear to contradict each other
[/quote]

Indeed - statements are meant to apply in the context of each described attack, as they're the two types of attacks I could find in the literature. Just meant to indicate that in [6] timing matters, in [7] it doesn't.

[quote="MattCorallo, post:18, topic:1723"]
the point of the forwarding delay is to create a batch such that a passive network monitor can’t trivially trace payments
[/quote]

My understanding of [6] is that just being able to construct these partial paths is insufficient to deanonymize senders and receivers (if we add some privacy-awareness to pathfinding).

Two questions here:
* Is there an off-path attack which does not require the attacker to run a ton of pathfinding to complete the path?
* Are you talking about the case where the attacker sees network messages for the full path (they're all in the same malicious AS group)?

[quote="MattCorallo, post:18, topic:1723"]
having no forwarding delays would mean that someone trying to get privacy by adding extra hops would be absolutely wrecked by an adversary running multiple nodes (post-PTLCs).
[/quote]

Could you explain this further? Based on the PTLC reference, I assume we're talking about the on-path attack? It's unclear to me why a receiver-side delay doesn't help with a multi-node on-path attacker.

-------------------------

MattCorallo | 2025-06-02 19:01:52 UTC | #20

[quote="carla, post:19, topic:1723"]
Is there an off-path attack which does not require the attacker to run a ton of pathfinding to complete the path?
[/quote]

Given many payments are only a few hops, I assume most of them, honestly.

[quote="carla, post:19, topic:1723"]
Are you talking about the case where the attacker sees network messages for the full path (they’re all in the same malicious AS group)?
[/quote]

I assume that in many payments an attacker can see network messages for much of the path, or at a minimum the first hop and last hop of a path, which suffices to figure out a payment based on timing (assuming we don't have any delays and only a moderate flow of payments along the path, which is probably common-ish, at least today, but if you only add one or two intermediate hops I assume its still pretty doable).

[quote="carla, post:19, topic:1723"]
Could you explain this further? Based on the PTLC reference, I assume we’re talking about the on-path attack? It’s unclear to me why a receiver-side delay doesn’t help with a multi-node on-path attacker.
[/quote]

My point was that being an on-path attacker is pretty similar in principle to a network attacker who can see only a subset of the hops, but with additional information, that lets you remove some false-positives in the classifier. Obviously pre-PTLCs you just know from the payment hash, but post-PTLCs the amount is pretty valuable, just not perfect - at that point we really want some delays so that the amount stays a bad classifier rather than a perfect one when combined with time.

-------------------------

joostjager | 2025-06-03 09:38:47 UTC | #21

Following up from yesterday's spec meet. This is my stance:

We can create friction by adding granularity or enforcing a minimum hold time, but this feels more like avoiding the problem than solving it. If users want fast routes, they will find them regardless. Even without explicit hold time data, latency can be inferred from sender observations over multiple payments. A scoring system that distributes observed delays across route hops is likely sufficient to identify slow nodes over time.

Rather than relying on filtering or encoding tricks that can be bypassed or become outdated, we should embrace the reality that the network will be exposed to performance pressures. This transparency can motivate stronger and more effective privacy solutions aligned with actual user incentives.

In my view, it is more sustainable to design the network to be resilient and private even when nodes compete on latency, rather than relying on weak measures that only obscure the problem and leave critical questions unanswered.

-------------------------

t-bast | 2025-06-03 10:05:02 UTC | #22

[quote="joostjager, post:21, topic:1723"]
Following up from yesterday’s spec meet.
[/quote]

Can you please summarize the options proposed? I don't think there's any written form of it anywhere, which makes it hard to form a good opinion for readers. If I understood correctly, the options were:

* Do nothing and keep a millisecond granularity on hold times reported in attributable failures
* Change the encoding of the hold time in attributable failures to have a granularity of Xms (with X to be defined): advantages/drawbacks of this option?
* Keep the millisecond granularity but ask nodes to subtract a hard-coded threshold value: we've discussed several variations of it, and to be honest I'm not sure exactly how that would work and would like to see it written down for analysis

-------------------------

joostjager | 2025-06-03 10:39:16 UTC | #23

[quote="t-bast, post:22, topic:1723"]
Can you please summarize the options proposed? I don’t think there’s any written form of it 
[/quote]

I’ll leave it to those proposing the alternative options to write up the details, pros, and cons. Personally, I’m in favor of simply keeping millisecond resolution. I don’t think we should postpone real problems by trying to obscure them. If users care about latency, they’ll find ways to measure it regardless. I also don’t think it’s wise to constrain the system today in order to avoid scenarios that are still hypothetical.

-------------------------

roasbeef | 2025-06-05 01:37:44 UTC | #24

[quote="t-bast, post:22, topic:1723"]
Can you please summarize the options proposed? I don’t think there’s any written form of it
[/quote]

Here's my attempt at summarizing the options. 

Skip the first two sections here for the options. 

### Background

Attributable errors provides a way for a sender to receive a tamper-evident error. This means that the sender can pinpoint _which_ node in the route inadvertently or purposefully garbled the error.  Today any node can flip a bit in the onion errors, which renders the entire error undecryptable, in a manner where no blame can be ascribed. 

The path finding of most implementations today has some component that will attempt to penalize a given node, or set of nodes for an unsuccessful route. The opposite is also useful as the path finder is able to _reward_ nodes for enabling the forwarding attempt to succeed up until a certain point. 

Without a way to attribute onion error garbling to a particular node, path finders either need to penalize the entire route, or do nothing. Both aren't great options. 

As a way to incentive the uptake of attributable errors by implementations, Joost proposed that the "hold time" be encoded in the errors for failed payments. In theory, this would allow path finders to pinpoint which nodes are persistently slow (bad network, faulty hardware, slow disk, etc) and penalize them in their path finding implementation. This rests on the assumption that users want fast payments, as poor payments are very bad UX (depending on the app, can appear to be stuck if no visual feedback is given). 

FWIW, I don't think any path finding implementation has _yet_ be updated to take this hold time information into account. Even today, path finders can bias towards geographically colocated nodes to reduce e2e latency (eg: no need to bounce to Tokyo, then LA, if the payment is going to Mexico). 

### Problem Statement

If we want to encode these hold times in the onion error, then a question that naturally arises is: **what encoding granularity should be used**? By this I mean, do we encode the latency numbers out right, or some less precise value, that may still be useful. 

A related question is if we encode this hold time: **to what degree does this decay privacy**? This question is what motivated this post to begin with. 

Before proceeding to the encoding options, I think it's important to emphasize that: **the sender always knows how much time the attempt took**. They can also further bisect the reported values, either by iteratively probing nodes in the path, or connecting out to them to measure ping latency. This brings forth a related third question: **what do we gain by encoding less precise values**? 

One other aspect as mentioned above is that a forwarding node can themselves become a sender. Even ignoring the latency encoding, log the resolution times of HTLCs they forward. For each of those HTLCs (similar amount, CLTV budget, etc), they can launch probes to attempt to correlate the destination. As mentioned above, variable receiver settlement delays mitigates this somewhat. 

### Latency Encoding Options 

I missed some of the initial discussion in the last spec meeting, but IIUC we have the following encoding options:
  * **Precise Values**:
     * **Rationale**: The sender already knows how long the route takes, and can measure how long it takes each node to forward as mentioned above. 
     * **Encoding**: Encode the actual value in milliseconds. 
  * **Bucketed Values**:
     * **Rationale**: We don't want to make it trivial to keep track of what the true per-hop latency is, so we should reduce the precision. 
     * **Encoding**: Given a bucket size (another parameter), report the bucket that a value falls in. So if we have buckets of 100 ms, and the actual latency is 120 ms, then 100 ms is reported. 
   * **Threshold Values**:
      * **Rationale**: Payments already take 1.5 RTTs per hop to extend, then half a round trip per hop (assuming pipelining) to settle. Therefore we can just extrapolate based no common geographical latencies, and pick a min/threshold value. IMO, this only viable if were space constrained, and want to encode the latency value in a single byte. 
      * **Encoding**: The value encoded isn't the actual latency, but the latency subtracted (floor of zero) or divided by some threshold. In the examples below, we assume this threshold is 200 ms, and the actual payment latency was 225 ms. 
         * **Subtracting**: A value of _25_ is encoded. If the value is below the threshold, then zero is reported. 
         * **Dividing**: A value of _1_ is encoded. Again if the value is below the threshold, zero is reported.

As we know LN is geographically distributed, so the actual latency depends on exactly where all the nodes are located. [Sites like this can be used](https://wondernetwork.com/pings) to get an idea of what types of latencies one would see in the real world. 

Both the threshold and bucket options need some parameter selected for an initial deployment. How do should we come up with such a parameter? During the discussion it was suggested that we just use a relatively high value like 300 ms, as it takes 1.5 RTT even in the direct hop case. Ofc payments can definitely be faster than 300 ms (small amount of hops, well connected merchant, etc), but anything around ~200-500 ms _feels_ instant. 

### Flexibility Concerns

One concern brought up during the discussion was flexibility: if we aren't encoding the actual value, then we need to pick some parameter for either the bucket, or the threshold value. The param is yet another value to bikeshed over. 

Changing this value in the future, may mean another long update cycle, as the senders need to upgrade to know how to parse/interpret the new value and it isn't _really_ useful until all the forwarding nodes also start to set these new values. 

### Flexibility Middle Ground 

One way to partially address this concern would be to: prefix the latency encoding with the type and parameter. So the final value on the wire would be `encoding_type || encode_param || encoding_value`. This would:
  1. Let nodes choose if they wanted to give granular information or not (hey! I'm fast, pick me!). 
      * Does one node choosing the precise excessively leak information? I'm not sure, as the sender knows what the real latency is. 
  2. Avoid hard coding the bucket/threshold param. As a result, we/nodes/implementations have a path to change it in the future. 

With this an open question is: if all, or just one of the encoding modes is specified (with the expectation that senders can interpret them all).

----- 

Personally, I favor: the self identifying encoding, with either just the actual value, or buckets (100ms?).

-------------------------

t-bast | 2025-06-05 07:42:57 UTC | #25

Thanks for the write-up @roasbeef :folded_hands: 

[quote="roasbeef, post:24, topic:1723"]
Personally, I favor: the self identifying encoding, with either just the actual value, or buckets (100ms?).
[/quote]

What do you mean by "self identifying encoding"? The encoding you're describing in your "Flexibility Middle Ground" section?

My personal opinion is that privacy is very important and needs to be protected. We chose from the beginning of the project to use onion routing because this was important to us ("us" meaning developers of the network). I strongly favor privacy over speed (within reasonable latency bounds of course): the goal of the lightning network in my opinion is to provide properties that you won't find in traditional payment rails, and privacy and censorship resistance are properties we absolutely don't have in traditional payment rails. Sacrificing those to get performance gains for which we don't have any concrete use-case today doesn't make any sense to me.

As we've highlighted several times during spec meetings, privacy loves company: you can only achieve good enough privacy if that is the default behavior of the network. Most users don't care about privacy (for them, or for other users of the network): but more importantly, most users *don't know that they care about privacy until something happens to them*, because of decades of telling people that if they haven't done anything wrong, they shouldn't have anything to hide. But I don't think this is fine, and I'm afraid of what a society without any privacy looks like - and don't want to live in one. So I'd rather have privacy built-in by default, and address the need for high performance separately, for example using direct channels (but again, since we don't have any use-case today that *requires* very low latency, we can't really design the right knobs for it).

If lightning succeeds at providing an open access to electronic payments for individuals worldwide, that offers good privacy, censorship resistance, low fees and reasonable speed (even 1-2 seconds is IMO reasonable speed compared to credit card payments), I would count it as an amazing success. I personally think that's where it provides the most real-world utility and I don't think we need to do "more" than that, I'd rather be 100% focused on making this use-case work well. I'm not at all interested in just building a faster VISA network.

Obviously, this is a "philosophical" question of what the developers of the network want: it always has been in every choice we make (feature we work on, implementation details, configuration knobs, etc). We're not impartial and I don't think we should try to be? We must be open to discussion and listen to our users, but in the end we all make a personal choice of what we want to spend time working on. And it's great that different contributors value different properties!

I'm sorry if it looks like we're slowing down progress on the attributable failures PR because of those discussions, but I think they're important (more important than just shipping a feature without which the network has been doing fine for years). I understand the frustration (I have pending spec PRs that have been waiting for much longer than attributable failures), but I think this is part of the trade-offs of working on an open, decentralized network?

-------------------------

GeorgeTsagk | 2025-06-05 12:17:42 UTC | #26

Would like to add some comments to a few of the points made in previous messages:

> @joostjager: In my view, it is more sustainable to design the network to be resilient and private even when nodes compete on latency, rather than relying on weak measures that only obscure the problem and leave critical questions unanswered.

It's also important to note that there are more crucial things to address that directly improve privacy over lightning, like forwarding and sender/receiver delays to mitigate the on/off path adversaries as described previously in this thread. Without addressing these, applying a privacy-dressing on the sender's feedback is only a partial illusion of privacy.

On top of that the sender may be considered the only actor that deserves to know as much as possible w.r.t what's going on with their payment, they're the ones piloting it at the end of the day.

> @roasbeef: One way to partially address this concern would be to: prefix the latency encoding with the type and parameter. So the final value on the wire would be `encoding_type || encode_param || encoding_value`. 

I don't really see the value of having the prefix. If you want to round up or down to buckets of 100ms then you're free to do so in the simple `uint64` encoding. Why would the sender want to know whether you're doing it or not?

> @t-bast: Sacrificing those to get performance gains for which we don’t have any concrete use-case today doesn’t make any sense to me.

A 2-part reply:

a) We're not really **sacrificing** privacy, it's more like being honest about the current situation. Similarly, we are not gaining any privacy by obfuscating the latency that is reported to the sender. Adversaries face no extra difficulties in analyzing traffic.

b) Focusing on the performance of LN is a fundamental use-case. Every day someone pays for their food or ticket with LN and the payment resolves fast is a small win for the whole network. I'm not saying we need 100ms instead of 1s, but we definitely need to treat performance as a fundamental requirement of the network.

> @t-bast: As we’ve highlighted several times during spec meetings, privacy loves company: you can only achieve good enough privacy if that is the default behavior of the network.

I still don't understand whether we're assuming that most people are willing to strip away parts of the software for the sake of speed. If that's the assumption, then the real mitigations for privacy (forwarding & sender/receiver delays) are really questionable, as anyone can just set the delays to 0s for a faster experience.

As you mentioned above "you can only achieve good enough privacy if that is the **default** behavior of the network", I believe there's a reason we don't want to say "the enforced behavior of the network", as in we still want to allow people to tweak their node **if they really want to**.

If all implementations have forwarding batching, sender & receiver delays and many other enhancements for privacy turned on (maybe some not even configurable), then would the majority of the network choose to nullify them? All we can really do is guide the user into making the right choices by controlling what's configurable, what the default values are, and by making sure to highlight the privacy impact of certain configuration choices in documentation.

### Final personal note

Attributable failures are very good at defining the hop that gets the blame in a route, not retrieving the latencies. The reported latencies are by nature "handwavy", it's what the node **claims** to have held the HTLC for. Any (sane) pathfinding algorithm would normalize that value first before incorporating it.

One of the arguments against expressive latency is that we're guiding nodes into picking the super fast chosen few routing nodes for payments and channel opens, weakening the topology of the lightning network. That's already happening today by having websites score nodes for you and then guiding you into choosing the best ones, which is already a worse version of the problem being played out right now.

Instead I'd like to see a future where we don't have to rely on external websites to source metrics and data for the network from and what your own node provides locally would be more than sufficient. The very existence of these websites is an indicator that people desire performance, and it's better to satisfy performance locally and on a p2p level (alongside privacy) rather than by submitting or fetching stats.

#### Proposed way forward

Keep the latency encoding as a `uint64`, where the value is the milliseconds the HTLC was held for. Each implementation is free to do bucketing or thresholds on the values they report when forwarding, without the need to report it. Similarly, the sender is free to apply thresholds or bucketing on the reported values, without signalling if they're doing it or not. Whether the former are going to be configurable is also up to the implementations (personal opinion is to not have it configurable, i.e someone has to build their own binary to change it).


If we ever have a more solid foundation on changing the encoding to be **something specific** then we can always upgrade to a more strict encoding (use a new TLV field and eventually deprecate the old one).

-------------------------

t-bast | 2025-06-05 12:24:39 UTC | #27

[quote="GeorgeTsagk, post:26, topic:1723"]
a) We’re not really **sacrificing** privacy, it’s more like being honest about the current situation. Similarly, we are not gaining any privacy by obfuscating the latency that is reported to the sender. Adversaries face no extra difficulties in analyzing traffic.
[/quote]

We are giving away more information than before, so this is definitely adding privacy risks. I don't buy the argument that adversaries have no extra difficulty obtaining that information elsewhere. It will keep getting harder and harder for adversaries to collect such information when we have random delays, message padding and jamming protection (which will include upfront fees, making probing costly).

 The issue is also mostly that incentivizing nodes to favor latency over privacy is not the direction I'd like to see the network take, as this is something we cannot come back from.

[quote="GeorgeTsagk, post:26, topic:1723"]
b) Focusing on the performance of LN is a fundamental use-case. Every day someone pays for their food or ticket with LN and the payment resolves fast is a small win for the whole network. I’m not saying we need 100ms instead of 1s, but we definitely need to treat performance as a fundamental requirement of the network.
[/quote]

I never said that we shouldn't treat performance as important, I said that it doesn't make sense to me to chase performance gains outside of the bounds of what really matters for payment UX if it hurts privacy. Having 1ms precision on payment forwarding latency is absolutely not crucial to get a decent payment UX.

[quote="GeorgeTsagk, post:26, topic:1723"]
I still don’t understand whether we’re assuming that most people are willing to strip away parts of the software for the sake of speed.
[/quote]

Of course some people will modify their software. But I'm ready to bet that it is going to be a very small minority of nodes, so it's fine. I'm convinced that the vast majority of the network will run standard lightning implementations, with most of the default configuration values.

[quote="GeorgeTsagk, post:26, topic:1723"]
That’s already happening today by having websites score nodes for you and then guiding you into choosing the best ones, which is already a worse version of the problem being played out right now
[/quote]

Nobody is using those to make their payments outside of a small niche of developers / tinkerers who are mostly playing around with running a node because they want to do something with their bitcoin: they're not the users who make the majority of the payment volume.

I think there's a big gap between real users (mostly non technical, who just need a payment app that works) and technical people who are spending a lot of their free time tinkering with lightning mostly for experimentation or to be part of a community because they own some bitcoin. It may be harsh, but I don't think the latter is what we should be focusing on when we design features.

On top of that, as Matt pointed out, the data exposed by those websites is mostly garbage. I don't believe that it has any significant impact on the network, or ever will.

-------------------------

t-bast | 2025-06-05 12:56:07 UTC | #28

To be clear: I'm not saying I'm absolutely against the 1ms precision in attributable failures. I'm only highlighting that I want to have a better understanding of the privacy risks before making a decision, and how important random delays are for privacy at different stages of the payment path, to ensure that attributable data doesn't incentivize in the wrong direction.

I do think that a 1ms precision isn't at all important for payment UX, and 100ms precision per-hop would be enough.

-------------------------

GeorgeTsagk | 2025-06-05 13:19:16 UTC | #29

[quote="t-bast, post:27, topic:1723"]
We are giving away more information than before, so this is definitely adding privacy risks.
[/quote]

Only the sender gets these latency values. An on/off path adversary does not extract additional data.

[quote="t-bast, post:27, topic:1723"]
It will keep getting harder and harder for adversaries to collect such information when we have random delays, message padding and jamming protection (which will include upfront fees, making probing costly).
[/quote]

As long as all of the above are not implemented it doesn't matter if the sender sees low or high resolution values. And when the above do get implemented, the number will have a natural threshold anyway.

[quote="t-bast, post:27, topic:1723"]
Nobody is using those to make their payments outside of a small niche of developers / tinkerers who are mostly playing around with running a node because they want to do something with their bitcoin: they’re not the users who make the majority of the payment volume.
[/quote]

If we don't build for node runners too then won't LN end up being a handful of centralized routing nodes owned by companies, who will probably submit traffic data directly anyway?

[quote="t-bast, post:28, topic:1723"]
I do think that a 1ms precision isn’t at all important for payment UX, and 100ms precision per-hop would be enough.
[/quote]

Agree, but instead of binding the protocol today with fixed numbers that we come up with we can just let it be `uint64` and implementations will enforce the X ms resolution or threshold over that field. This is only bad in the case where we assume everyone in the network to wake up one day and strip away the rounding/thresholds. It also allows us to change the way we report hold times in the future without having to change the way it appears on the wire.

-------------------------

MattCorallo | 2025-06-05 13:52:50 UTC | #30

[quote="roasbeef, post:24, topic:1723"]
They can also further bisect the reported values, either by iteratively probing nodes in the path, or connecting out to them to measure ping latency. This brings forth a related third question: **what do we gain by encoding less precise values**?
[/quote]

With randomized delays (plus randomization in I/O latency), doing iterative probing may require a very nontrivial number of tries to map the whole network. With future upfront fees, this may be impractical. In practice, we see people who try this largely fail to provide reasonably accurate data today (at least in the sense that we care about here - they can certainly provide "this node is terrible/on tor" vs "this node seems reasonable"-type data, which is obviously a thing we want to provide to all nodes here).

[quote="roasbeef, post:24, topic:1723"]
One way to partially address this concern would be to: prefix the latency encoding with the type and parameter. So the final value on the wire would be `encoding_type || encode_param || encoding_value`. This would:
[/quote]

I don't buy that its worth us all implementing a configuration knob and multiple encoding options for this. Sometimes more flexibility isn't the right answer :)

[quote="roasbeef, post:24, topic:1723"]
Let nodes choose if they wanted to give granular information or not (hey! I’m fast, pick me!).
[/quote]

This would defeat the whole purpose of attempting to reduce the information provided -  privacy loves company - the whole point of the discussion here was to ensure that nodes *cannot* (in a standardized way) communicate granular latency information as it incentivizes them to do so as senders would (presumably) use that information to (strongly) prefer nodes with even marginal decreases in latency.

[quote="GeorgeTsagk, post:26, topic:1723"]
It’s also important to note that there are more crucial things to address that directly improve privacy over lightning, like forwarding and sender/receiver delays to mitigate the on/off path adversaries as described previously in this thread. Without addressing these, applying a privacy-dressing on the sender’s feedback is only a partial illusion of privacy.
[/quote]

Indeed, but providing fine-grained latency information strongly incentivizes nodes to remove any forwarding/batching delays, effectively limiting our options later to improve privacy (at least by ensuring everyone has a batching delay, at least by default). I don't think this conversation was ever about whether this, itself, provides privacy, but rather whether it closes off privacy features we want.

[quote="GeorgeTsagk, post:26, topic:1723"]
b) Focusing on the performance of LN is a fundamental use-case. Every day someone pays for their food or ticket with LN and the payment resolves fast is a small win for the whole network. I’m not saying we need 100ms instead of 1s, but we definitely need to treat performance as a fundamental requirement of the network.
[/quote]

Its important that we be clear about what this means - lightning has some fundamental limits, and will (in its current protocol) certainly never achieve reliable payment latencies below a second or so (and is already achieving such latencies in practice!). Indeed, changes in payment latency by an order of magnitude absolutely changes what lightning can be used for and opens up new use-cases which we should want. However, in Carla's analysis in the OP it seems like forwarding/batching delays round an RTT or less can result in reasonable privacy improvements (assuming enough traffic on nodes and some other network-level improvements we want to make).

Thus, declining to offer fine-grained latency information and encouraging nodes to always batch forwards/failures on the order of 100ms will not change the set of use-cases LN is usable for, nor change the user experience of a lightning payment, nor materially change the "performance of LN". Given this, and the fact that privacy is also a critical feature of LN (whether we have it today or not), I'm not entirely clear on why its worth providing fine-grained latency information here. We don't currently envision a serious use-case for that kind of data, and there's some (even if marginal) risk in making it available.

-------------------------

t-bast | 2025-06-05 14:55:14 UTC | #31

[quote="GeorgeTsagk, post:29, topic:1723"]
Only the sender gets these latency values. An on/off path adversary does not extract additional data.
[/quote]

But it's not a matter of whether only the sender gets these latency values, the issue is that it incentivizes routing nodes to be as fast as possible to be picked up by senders and thus removes our ability to introduce random delays in the future, because the incentives will be against them.

It's true that we haven't implemented yet those privacy features (random delays / message padding), but we've made sure that nothing prevented us from adding them whenever we had time to work on them. Creating an incentive against them would be a big change that I'm afraid we wouldn't be able to fix afterwards.

[quote="GeorgeTsagk, post:29, topic:1723"]
If we don’t build for node runners too then won’t LN end up being a handful of centralized routing nodes owned by companies, who will probably submit traffic data directly anyway?
[/quote]

I believe you misunderstood my comment here. We are of course building software for routing nodes, but it doesn't make sense to build features they only use for fun that don't have a useful impact on the network. There are a lot of things that routing nodes think they want (most of the time the small ones that are mostly tinkering) that are either useless or harmful, and I don't think we should provide them, even though some people are asking for it. They're free to implement them anyway and document them in bLIPs, but I'm pretty sure they'll realize it's not worth the effort. But it's a good thing that they can do it because it's open-source software, and then they can prove me wrong!

[quote="MattCorallo, post:30, topic:1723"]
I don’t think this conversation was ever about whether this, itself, provides privacy, but rather whether it closes off privacy features we want.
[/quote]

I completely agree with that.

-------------------------

carla | 2025-06-05 19:01:49 UTC | #32


It seemed to be broadly agreed on in the spec meeting that in a world with receiver-side delays, we no longer need to rely forwarding delays for privacy against an on-path attacker, yay.
We should implement and deploy [#1263](https://github.com/lightning/bolts/pull/1263), and we'll get a nice improvement against a known attack.

Once we're in a receiver-delay world, the main privacy purpose for forwarding delays is to make it more difficult for an off-path adversary to trace payments through the network. While [6] outlines a case that relies on running repeated pathfinding attempts to reduce the anonymity set and identify the sender/receiver, there could be other possible ways to try deanonymize senders/receivers in these traced paths. IMO we don't really have a good understanding of what these are; seems like a good direction for future research (and not something that we're going to know in the near term). I agree with the intuition that an adversary that's able to make these partial paths can start to mess with privacy is correct.

An _actually effective_ forwarding delay that aims to have more than one HTLC per batch seems like a very difficult number to pick. It's highly dependent on a node's traffic, which is likely quite variable during the day, and will change as channels open and close. Interested to hear whether anyone's aware of research that we could look at to inform picking such a value?

It seems to me that message padding and some degree of cover traffic (dandelion for LN?) offer better privacy protections than forwarding delays because:
1. They'd offer protection to nodes on the edges of the graph that have less traffic (unlikely to be able to batch otherwise).
2. A bandwidth/privacy tradeoff is probably more palatable to forwarding nodes than a latency/privacy one (adding 30 GB egress to AWS is like $3), so they're more likely to adhere to this measure.
3. An attacker can probe latency without attributable failures (albeit at a cost, in an unconditional fee world)

All that to say, it's not obvious to me that we're closing ourselves off to future privacy improvements because (IMO) forwarding delays seem to be the least promising of the mitigations available to us.

Would folks consider leaving ms encoding and adding a sender-side advisory in the specification not to penalize hops under some threshold (300ms)? That provides a default that doesn't put downward pressure on forwarding delays that's more flexible in the future - if we find out that there's no privacy problem, we can remove the sender instruction, and if we find out there is one we can ship a new default pretty easily.

I think it's reasonable to believe that the majority of the network will run with this default (likely, without even knowing it's there).

-------------------------

brh28 | 2025-06-05 22:17:37 UTC | #33

Could someone elaborate on why the self-reported hold times are necessary? Specifically, I'm wondering:

1. What are the reasons for routing nodes holding onto HTLCs? Is this happening today?
2. If a routing node is delaying an HTLC, why would they self-report it?

[Edit] To my understanding, if all nodes in a path are honest, this informs the payment sender of the latency at each hop in a route. If a node lies, the difference in reported times by the node and and its peer will localize the delay to those two nodes; in which case, the sender likely avoids both peers.

-------------------------

joostjager | 2025-06-06 07:55:38 UTC | #35

Routing nodes may not deliberately delay HTLCs, but slow hardware, poor network conditions, or privacy-related strategies can cause latency. These nodes aren’t required to disclose intentional delays. However, the sender can measure the total end-to-end latency and may assign penalties to nodes involved. Without hold time information, the sender is likely to distribute this penalty evenly across the route.

Nodes that report their hold times can influence how the penalty is allocated. Including the delay in their report shifts the blame toward the next (outgoing) hop, while omitting it shifts blame toward the previous (incoming) hop. This trade-off allows each node to strategically decide what to report based on their own interests.

-------------------------

joostjager | 2025-06-06 08:11:34 UTC | #36

I’d like to offer another angle on this issue that ties back to the incentive mismatch raised earlier.

Routing nodes are naturally incentivized to minimize latency. Holding an HTLC slot longer than necessary can reduce their capacity to forward other payments and may lead to lost fee revenue. Nodes that don’t see heavy traffic may also see little reason to batch, unless it's adaptive and only used when demand is high. On top of that, longer-held HTLCs increase commitment transaction weight and could raise the cost of force closures. This has all been mentioned already.

From their perspective, there’s no benefit in adding delay to HTLCs. So why would they, given these downsides?

Even if we try to encourage delay-based privacy features, the economically dominant nodes may strip them out to keep latency as low as possible. This is rational behavior under current incentives.

Now suppose the network converges on coarse-grained hold times, say all nodes advertise 300 ms.

In that environment, a node that wants to introduce privacy-preserving delays at its own cost has no way to stand out. The coarseness of the hold time field makes it impossible to signal variability or randomness to the sender. Every node appears the same, even if some are doing extra work.

In that sense, coarse-grained hold times may actually discourage privacy improvements. They flatten the signal space and remove the ability for privacy-oriented nodes to differentiate themselves from latency-optimized ones.

-------------------------

t-bast | 2025-06-06 09:03:28 UTC | #37

[quote="joostjager, post:36, topic:1723"]
Routing nodes are naturally incentivized to minimize latency.
[/quote]

I don't think that this is correct, or as clear-cut as you think: can you detail what makes you think that? The main performance bottleneck for routing nodes is by far disk IO (database operations) that happens when `commitment_signed` is sent. Nodes are thus incentivized to batch HTLCs to minimize this disk IO, especially when they route a lot of payments. Even if they don't route a lot of payments, they cannot know when the next HTLC will come: so it is a sound strategy to wait for a small random duration before signing, in case another HTLC comes in.

Interestingly, since such batching reduces the frequency of disk IO, it provides more stable latency. The end result is a higher median latency (ie not chasing every millisecond) but a smaller standard deviation.

The batching interval really depends on the expected frequency of payments relative to the performance of the DB. But I believe that if lightning is successful at being a payment network, it doesn't have to be a huge value? I think that we can use a value that provides a good enough payment UX while providing good node performance.

[quote="carla, post:32, topic:1723"]
It seems to me that message padding and some degree of cover traffic (dandelion for LN?) offer better privacy protections than forwarding delays because:
[/quote]

I agree with you. Based on my current understanding, my preferred choice would be:

- receiver-side random delays
- sender-side random delay on retries
- small randomized batching interval at intermediate nodes (mostly for performance, but also to add a small amount of noise of relay latency)
- random message padding / cover traffic (which I think doesn't have to be CBR to be effective)

As you say, this doesn't rule out the current 1ms encoding for attributable failures. But I'd be curious to have @MattCorallo and @tnull's thoughts here: they mentioned during the spec meeting that intermediate forwarding delays are important for privacy even when we have random message padding and cover traffic, and I don't understand why. So I may be missing something important.

-------------------------

joostjager | 2025-06-06 09:19:40 UTC | #38

I think I've mentioned reasons for routing nodes to minimize latency. In line with that, if disk IO increases latency, they'd indeed want to batch. But that doesn't necessarily mean that they'd use batching always blindly. For less busy nodes, there may be no latency improvement. And indeed, HTLC busy times may be unpredictable, but probably not completely random. Overall it can still be better for latency to use an adaptive batching strategy. Even if a batch is sometimes missed, on average latency can still be better than always batching.

-------------------------

tnull | 2025-06-06 09:35:30 UTC | #39

[quote="t-bast, post:37, topic:1723"]
Based on my current understanding, my preferred choice would be:

* receiver-side random delays
* sender-side random delay on retries
* small randomized batching interval at intermediate nodes (mostly for performance, but also to add a small amount of noise of relay latency)
* random message padding / cover traffic (which I think doesn’t have to be CBR to be effective)
[/quote]

Yes, totally agree in generally here. 

Although I don't think we need to add sender-side retry delays if we wouldn't retry over exactly the same route. And, AFAIK, only LND currently does this under certain circumstances where they give a node a 'second chance' if they report back one of a certain set of failure codes.

[quote="t-bast, post:37, topic:1723"]
As you say, this doesn’t rule out the current 1ms encoding for attributable failures. But I’d be curious to have @MattCorallo and @tnull’s thoughts here: they mentioned during the spec meeting that intermediate forwarding delays are important for privacy even when we have random message padding and cover traffic, and I don’t understand why. So I may be missing something important.
[/quote]

So, while the attack described in the Revelio paper is mostly based on a heuristic that exploits the distinct packet sizes, they still group the streams of IP packages / 3-tuple (sender IP, receiver IP, message length, basically) that an adversary might observe at different points by their timing. Basically, the adversary would be able to observe the HTLC dance at one point in time / in the network, and then an HTLC dance at a later point / a different point in the network. To correlate these two observations and reconstruct that it was in fact the same payment they use timing information. The same approach could be utilized by an on-path adversary with multiple vantage points in the network post-PTLCs, but of course currently they can simply match the payment hash to ensure that two observations are indeed the same payment. 

So yes, the more entropy/noise we add/maintain to/in the forwarding process the harder we make the adversary's job of coming up with reliable models, which is why we likely wouldn't want to drop the forwarding delay entirely (although note it's mostly about maximizing *uncertainty* not the added net delay necessarily).

-------------------------

carla | 2025-06-06 12:47:57 UTC | #40

[quote="t-bast, post:37, topic:1723"]
Based on my current understanding, my preferred choice would be:

* receiver-side random delays
* sender-side random delay on retries
* small randomized batching interval at intermediate nodes (mostly for performance, but also to add a small amount of noise of relay latency)
* random message padding / cover traffic (which I think doesn’t have to be CBR to be effective)
[/quote]

[quote="tnull, post:39, topic:1723"]
I don’t think we need to add sender-side retry delays if we wouldn’t retry over exactly the same route.
[/quote]

I'm in agreement with all of the above.

[quote="t-bast, post:37, topic:1723"]
they mentioned during the spec meeting that intermediate forwarding delays are important for privacy even when we have random message padding and cover traffic,
[/quote]

[quote="tnull, post:39, topic:1723"]
Basically, the adversary would be able to observe the HTLC dance at one point in time / in the network, and then an HTLC dance at a later point / a different point in the network. To correlate these two observations and reconstruct that it was in fact the same payment they use timing information.
[/quote]

Just to re-iterate @t-bast's question. Wouldn't the whole point of cover traffic be that we have "fake" HTLCs propagating through the network at the same time as real ones so that an attacker gets false positives on this type of surveillance?

-------------------------

