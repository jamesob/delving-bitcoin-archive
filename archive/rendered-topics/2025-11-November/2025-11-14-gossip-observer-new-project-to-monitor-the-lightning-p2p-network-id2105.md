# Gossip Observer: New project to monitor the Lightning P2P network

jonhbit | 2025-11-14 20:48:38 UTC | #1

For the past few months (since plebfi miami), I've been working on a project to monitor the Lightning gossip network by collecting gossip messages from numerous nodes. The repo with the code used, the raw data collected before lightning++, and the slides from my talk is here:

<https://github.com/jharveyb/gossip_observer>

## Observations:

- Convergence delay (time for a message to propagate around the network) has dropped a significant amount since similar measurements in 2022, from ~500 to ~200 seconds for 75% propagation. I suspect this is due to LN implementations defaulting to making more P2P connections.

- A significant subset of all messages received were sent to my node by less than 1/4th of its peers. This could be due to a poorly connected graph of P2P connections, or some filtering policy in the LN implementations.

- For channel_update messages, which were 60% of total messages, 20% of channels had more than 144 messages. This is relevant when compared to the proposed gossip rate limit for Taproot gossip / gossip 1.5(?):

<https://github.com/lightning/bolts/pull/1059>

- For node_announcement messages, which were 30% of total messages, 2.5% of nodes were announced more than 144 times. This seems like a unintended behavior of some implementation or node automation tool.

- The total size of all unique messages collected over that day was 103.2 MB, which is greater than I expected. This does not account for the bandwidth overhead of receiving the same message from multiple peers, so node bandwidth usage is likely much higher.

## Future Work

So far, I only collected data over a 24 hour period and analyzed that. In the near term I'll be setting up permanent infrastructure to receive gossip from multiple geographic regions and neighborhoods in the P2P graph. I'll also be adding functionality to broadcast gossip messages from these nodes, and observe their propagation. Work will continue in the gossip-observer repo linked above.

## Getting Involved

If you have feedback on interesting metrics to compute, a request to track gossip propagation from your node, or anything else, feel free to message me here!

I'm also investigating how this raw data could be published so others can perform their own analysis. I'm particularly interested in options for unsupervised anomaly detection, but I'm also definitely not a data scientist.

-------------------------

jonhbit | 2025-11-19 21:31:53 UTC | #2

A related subject is how we could switch the LN P2P network from message flooding to something closer to the design outlined in the Erlay paper and BIP. Some observations:

- Latency / propagation delay is much less important for LN gossip; implementations will allow slightly outdated fees to apply to their channel, for example.

- Gossip messages are signed by the sender, and are intended to be public. So the propagation method does not need to conceal information about the sender's position nor connections in the LN P2P graph.

There has been some existing work on this subject:

<https://endothermic.dev/p/magical-minisketch>

### Adapting Minisketch to LN Gossip

A key detail is how to map gossip messages to Minisketch set elements. This mapping should be collision-resistant, and ideally the elements are short, since reconciliation time is quadratic wrt. element length.

In the Erlay design, collisions are avoided by generating a salt for each peer connection, and using that and the TXID as input to a fast non-cryptographic collision-resistant hash like SipHash. This means that the total amount of hashing operations needed scales with the number of peer connections, but having a shorter (32-bit) set element greatly reduces the compute needed for set construction and reconciliation.

Alternatively, set elements could be computed from a channel's short channel ID (64 bits), plus the block height included in the gossip message. However, there are some issues with this approach.

One is that the components of a short channel ID (channel open block height, transaction index, and output index) don't have a lot of entropy. For example, in a full block, one could have ~36000 1-in 1-out P2TR TXs of 111 vB, or 1 TX with 1 input and 93020 outputs with weight ~3999900 vB. In either case, the entropy of the TX + output index is only ~17 bits. Mixing in block heights does not add much to the total entropy, since a message can only use a block height from the last two weeks (~2016 possible values, 11 bits of entropy).

[Edit 19.11.25: I forgot about the max standard TX weight of 400k WU; so a full block could have 10 TXs with ~3600 outputs each. That doesn't really change the total entropy of (TX index + output index) I think.]

In addition, we would need a different mapping for node_announcement messages.

Given that LN nodes already store some per-peer state on connection (re)establishment, I'm in favor of borrowing the Erlay approach and using a per-peer salt + a hash like SipHash to generate short Minisketch set elements for each gossip message.

### Gossip v1.5 message format

Another detail is that the gossip 1.5 proposal allows for optional fields in gossip messages, which is not possible with current gossip messages:

<https://github.com/lightning/bolts/pull/1059>

However, I think peers can decide which fields to send to their peer based on their announced feature bits.

### Other considerations

Compared to the Erlay design, I don't think LN implementations would need to perform any message flooding at all. Combining limited flooding and set reconciliation is useful for reducing total propagation delay, but the LN gossip propagation delay is so large that it could be maintained through set reconciliation alone. This should greatly simplify implementation complexity as well.

With respect to implementation, one problem would be different nodes filtering messages with different policies. This would lead to persistent set differences, which would waste bandwidth and, in the worst case, cause reconciliations to fail. I suspect that these filtering policies are either legacy behavior, or adaptions to quirks in existing gossip behavior, and could largely be removed if we move to set reconciliation.

### Future Work

In parallel to the gossip monitoring work, I'd like to get feedback from LN implementers on which designs could work for them. Is the per-peer state a significant drawback or source of complexity? Would there be an issue with more frequent (every 30 seconds?) set reconciliation and gossip message handling? Are there other issues that haven't been considered?

I'll update this thread if/when I have a more concrete proposal or draft BIP.

-------------------------

gmaxwell | 2025-11-17 21:51:45 UTC | #3

[quote="jonhbit, post:2, topic:2105"]
Compared to the Erlay design, I don’t think LN implementations would need to perform any message flooding at all. Combining limited flooding and set reconciliation is useful for reducing total propagation delay,

[/quote]

Minisketch has superlinear decoding costs, so decoding huge sketches will burn up a lot of CPU.   One could reconcile more often to try to fight this, but every reconcile will imply some communication overheads since you’ll overshoot the unknown needed amount for a reconstruction.

The flooding in the erlay has the advantage that takes the bulk of the load off the sketch and lets the sketch fill in the small omissions which it’s good at doing.

If you were to use reconciliation only you might be better off using iblt instead of minisketch (maybe plus a very small minisketch to unjam stuck IBLT decodes).   The overheads of iblt are much worse but that may be less significant if you’re running all the traffic through it.

-------------------------

jonhbit | 2025-11-19 22:36:32 UTC | #4

[quote="gmaxwell, post:3, topic:2105, full:true"]
Minisketch has superlinear decoding costs, so decoding huge sketches will burn up a lot of CPU.   One could reconcile more often to try to fight this, but every reconcile will imply some communication overheads since you’ll overshoot the unknown needed amount for a reconstruction.

The flooding in the erlay has the advantage that takes the bulk of the load off the sketch and lets the sketch fill in the small omissions which it’s good at doing.
[/quote]

I missed that benefit of the flooding in Erlay on my previous reads of the paper; that makes sense.

[quote="gmaxwell, post:3, topic:2105, full:true"]
If you were to use reconciliation only you might be better off using iblt instead of minisketch (maybe plus a very small minisketch to unjam stuck IBLT decodes).   The overheads of iblt are much worse but that may be less significant if you’re running all the traffic through it.
[/quote]

This opened a rabbit hole that actually led me to a very recent paper proposing an IBLT-based set reconciliation protocol, that may be well-suited for this use case.

Firstly, the CPISync implementation used for benchmarks in the Minisketch repo has moved and expanded to include some new protocols:

https://github.com/nislab/gensync

There is a related paper that explores some of the tradeoffs of sync with Cuckoo filters vs. CPI vs. IBLT:

https://arxiv.org/abs/2303.17530

Cuckoo filters looked interesting, but IIUC don't fit our use case well since a participant learns which elements their counterparty is missing, not the opposite (which elements they should request). The bandwidth used also seems to have a high floor. Outside of that, the benchmarks focus on much larger sets than what's relevant in the LN Gossip (or Erlay) setting.

I never looked at the MET-IBLT paper; it seems like a better IBLT sync scheme, but with no benefits compared to RIBLT.

Looking into cuckoo filters eventually led me to this paper, Rateless IBLT (RIBLT):

https://arxiv.org/abs/2402.02668

Which seems very promising. They even used the Minisketch library in their benchmarks! _And_ there is a public implementation in Golang (+ impls in C++ and Rust linked in the README):

https://github.com/yangl1996/riblt

The main math & explanation is in Section 4, with comparison to alternatives in Section 7.

As a tl;dr:

- To make an IBLT rateless / an infinite stream of coded symbols, we can extend a mostly-normal IBLT s.t. later coded symbols map to fewer and fewer inputs. If we have an efficient function to compute the indices of coded symbols an input must contribute to, we can encode our table efficiently even as the set size grows.
- - We can also extend the table incrementally, and send extensions until decoding succeeds, so we don't need to estimate the set difference nor regenerate the IBLT if decode fails (similar to extending a Minisketch) (the paper doesn't comment much on partial recovery, but Figure 6 has some simulation results related to this).

- The bandwidth overhead seems to have a ceiling of ~1.75, even for sets with very few differences; it converges to ~1.35 as differences increase past 100, which seems significantly better than standard IBLT? (Figure 5).

- For both large and small set differences (1-1000), encoding cost grows linearly (Figure 8) This also holds for total set size (FIgure 10). Worth noting that the ratio (set_differences)/(set_size) is quite small in most of their benchmarks.

- Most of their benchmarks use an element size of 64 bits. IIUC the bandwidth overhead could be significantly reduced if an implementation was optimized for small sets and smaller elements; there are some relevant notes in Sections 7.1 and 7.2.

If these performance properties hold up, I agree that mixing frequent IBLT-based syncs + infrequent Minisketch usage is appealing. It may also be worth skipping Minisketch entirely and trade bandwidth overhead for the CPU savings; I'm not sure how LN implementations feel about that tradeoff tbh. 

Another consideration is that the elements we want to exchange are very small (average message size <275 bytes), so there isn't much room to add bandwidth overhead. These messages will still be small with gossip v2.

Being able to have larger (64+ bit) elements with only a small increased CPU cost (FIgure 11) may be a very significant benefit; I'll take a look at these implementations soon and report back.

-------------------------

gmaxwell | 2025-11-19 23:43:49 UTC | #5

Just be careful with figures from papers they tend to be rather asymptotic.  IBLT like schemes usually start with \~32-bits per difference in overhead for a checksum which is pretty bad when otherwise 30-bit members are fine >100% overhead before you even get to the overheads needed to achieve correct reconstruction.  :smiley:  Some papers simply leave this overhead out of their figures, though I don’t recall if the rateless paper did that.

(FWIW, you can use minisketch in a kind of quasi rateless way by just dynamically sending more until the other side could recover– almost all the computation from a partial one can be conserved, as you don’t get to the expensive and non-reusable root finding step until you’re almost certain to have a correct decode, so long as you’re willing to take one or two extra elements overhead)

-------------------------

jonhbit | 2025-11-20 16:18:51 UTC | #6

[quote="gmaxwell, post:5, topic:2105, full:true"]
Just be careful with figures from papers they tend to be rather asymptotic.  IBLT like schemes usually start with \~32-bits per difference in overhead for a checksum which is pretty bad when otherwise 30-bit members are fine
[/quote]

Fair; their main benchmarks are using 32-byte members and 8-byte checksums. I think the claimed overhead depends on the ratio between these two? Which would be worse for this application.

They do mention that you can shorten the checksum for small sets, but it definitely isn't benchmarked. I _think_ that is acceptable if the set members are already sufficiently collision resistant.

[quote="gmaxwell, post:5, topic:2105, full:true"]
(FWIW, you can use minisketch in a kind of quasi rateless way by just dynamically sending more until the other side could recover– almost all the computation from a partial one can be conserved, as you don’t get to the expensive and non-reusable root finding step until you’re almost certain to have a correct decode, so long as you’re willing to take one or two extra elements overhead)
[/quote]

Ah, I didn't fully realize that this was the tradeoff for bisection / sketch reuse from the original paper nor notes on the repo :sweat_smile: But I see it listed as a TODO. That seems like the best option then; having more than one communication round for bisection is also more acceptable for LN than Bitcoin TX broadcast.

I don't have a real estimate nor intuition for how many differences to expect; I may be able to estimate that by looking at real-world traffic, but it seems like this also depends on the flooding timer skew across all of your peers? Perhaps that is another option for reducing differences. Eventually I'll replay real-world traffic in simulation for a proper estimate.

Related, one simple adjustment to reduce differences would be to flood messages your node generates to your direct peers, instead of waiting for the reconciliation timer.

-------------------------

rustyrussell | 2025-11-22 22:48:46 UTC | #7

Sorry to join the conversation late, and I haven’t worked on this for years so my memory could be stale, but here’s a brain dump of our conclusions.

1. Short channel ids have a lot of bits to squeeze. Ultimate would be to refer to outputs by their index in the blockchain (i.e. the 123456th output), but you can easily use fewer than 64 bits for blocknum/txnum/outnum.
2. You maintain three minisketches. We tried a single, but it costs encoding bits, complexity and hits the O(N^2) harder.
3. Channel minisketch is just the scids. Importantly, you send your blockheight with the sketch, because that informs the failure case.
4. Channel update minisketch is compacted scid + direction bit + height. Note that height takes 10 or 11 bits, IIRC, since you don’t send expired ones. You can only reliably decode this once you have reconciled the channels sketch.
5. Node announce sketch is uses the same encoding, using the oldest channel attached to the node as its key. Again this assumes you reconciled the channels sketch first.

If you cannot encode an scid compactly, just send it as a series of  “raw" entries. IIRC creating such an scid requires an exceptional number of txs in a block or exceptional output count.

With this scheme, you simply send the sketches every 60 seconds (like now), and your peer sends you what you’re missing.

Note that you can truncate the set you send if you want to save bandwidth, but really the cost is in the set maintenance, so maybe this is silly. My memory is that minisketch is *fast* in practice though, even if you keep a 64k (8k element) set which is our max message size anyway.

If reconstruction fails, there are several things you can do:

1. If block height differs, ignore. Time will sort it. Maybe include block hash here?
2. Enlarge your own set (or, send more of your set). If this allows your peer to reconstruct, it will learn that you cannot reconstruct, and it knows to send its largest set if it wasn’t already.
3. Wait for other peers. You might close the gap.
4. Existing gossip queries for recent changes (assuming a pile of old changes haven’t suddenly appeared).  You know if you need announcements, updates or node anns.
5. Query for everything.

Oh, we added a “total entries" counter to each message, which gives a clue as well: if your peer has far fewer entries, it’s a cry for help :slight_smile:

-------------------------

rustyrussell | 2025-11-23 00:28:57 UTC | #8

(post deleted by author)

-------------------------

