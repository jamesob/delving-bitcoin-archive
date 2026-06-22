# Fountain Codes: a way to reduce blockchain storage costs

lucasdbr05 | 2026-06-20 10:33:17 UTC | #1

Over the past few months, I have been conducting research on the application of Fountain Codes in Bitcoin, together with Vinteum.

Fountain codes allow the Bitcoin blockchain to be split into encoded fragments called droplets, such that any node can reconstruct the full blockchain by collecting and decoding a sufficient number of them. Each node only needs to store a fraction of this encoded data, reducing the storage burden per node.

I wrote a blog post explaining the idea and covering the progress I've made in the research so far. You can read it at [https://lucasdbr05.com/posts/fountain-codes/](https://lucasdbr05.com/posts/fountain-codes/).

-------------------------

ajtowns | 2026-06-21 10:34:06 UTC | #2

Few comments:

> On the other hand, a larger k also requires the node to buffer more data before closing the epoch and generating the encoding.

I don't think that's true: if you have the droplet composition be deterministic based on a per-peer seed and the height, then you can construct the $s$ droplets as you go. I think the idea was that peers would share the seed they use for working out what droplets they'll select, both to save a little bit of data, but also to help guarantee unbiased random selection?

So with $k=1,000$ and $s=10$ you might decide your droplets for blocks 0-999 are going to be:

  * [18, 321, 688, 875, 890]
  * [36, 170]
  * [39, 178, 980]
  * [73, 104, 153, 167, 370, 577, 586, 864, 865, 978]
  * [91, 980]
  * [164, 359, 741, 842]
  * [208, 820]
  * [252, 330, 361, 399, 477, 478, 528, 667, 688, 964]
  * [351, 876]
  * [679]

But you can just build those up as you go, so at height 215, your droplets look like:

  * [18]
  * [36, 170]
  * [39, 178]
  * [73, 104, 153, 167]
  * [91]
  * [164]
  * [208]
  * empty
  * empty
  * empty

So I think your storage is just the fraction $s/k$ versus storing all blocks. I think it would be reasonable to paremeterize based on $k$ and $r=s/k$ instead, with the potential for $r$ being a per-node setting.

> So, if we choose `k = 1000` and `s = 10`, the target is approximately `100x` storage savings.

Choosing 100x savings per-node means that you need to connect to 100 nodes (or more) in order to actually reconstruct all the blocks, whereas you'd only need to connect to 1 full archival node. Reportedly there are currently 27k NODE_NETWORK_LIMITED listening nodes and 24k NODE_NETWORK listening nodes; so if all current pruned nodes adopted this, it would only act as a bump from 24,000 copies of the blockchain available for download to 24,030. So I think this is much more a "robust engineering for future scenarios" than making a tangible improvement today.

With a 100x savings, that's presumably an extra ~7GB on disk. A 20x savings (~35GB) would probably be pretty feasible for many node operators who aren't able to store the full ~700GB of old blocks and would boost the additional number of copies of the blockchain in the above scenario from +30 to +150.

If you have to connect to 100+ peers instead of 1 (or 10) to do IBD, that's a significantly increased connection load on pruned peers (who currently don't participate in helping IBD at all), so it might be sensible to have connections made during IBD be block-relay-only, and, once IBD is complete, drop and reconnect in order to start doing tx relay.

> the protocol also needs to define how to handle murky droplets.

My first thought is that you'd disconnect and ban a peer that gave you bad data (so not request any additional droplets), but might keep the remaining droplets they gave you on the offchance any of them turned out to be useful.

I wonder what the impact of different serialization sizes of blocks is here: if you have a droplet that's [208, 820] and 208 is full of inscriptions and 4MB while 820 is full of runestones and 1MB, I think the droplet has to be the full 4MB. That seems like it could result in a multiplier effect, though probably isn't too substantial unless $k$ is very large.

-------------------------

gmaxwell | 2026-06-21 16:51:32 UTC | #3

I made a prior development list post where I proposed using RS codes to divide partial nodes into a collection of flavors such that syncing required connecting to e.g. 8 distinct flavors.  I actively eschewed fountain codes in this proposal (in spite of the fact that we used them in Bitcoin Fibre and the satellite broadcast) for two reasons:

Bitcoin has to be robust against actively malicious peers. If you get data from a bunch of different peers and some are malicious there didn't seem have anything short of exptime algorithms to find the set of malicious peers and successfully reconstruct the data in spite of them.  For an RS code you can decode in polytime using berlekamp once you have enough data (e.g. two extra good peers for each malicious peer) to correct errors and not just erasures.  Decoding absent malicious peers can still be very fast (even for big codes e.g. https://github.com/catid/leopard/ ).

The other reason was that fingerprinting is a concern-- when a node changes network identity it is preferable if it is difficult to distinguish from other nodes, and many protocol decisions have been shaped by this, e.g. the fact that there is a single prune height advertised for limited nodes.  Because of this I think it makes sense to limit the number of distinct shards.  It does mean that distribution wouldn't be as perfectly uniform where you need literally *any* N peers to reconstruct but I think not making every partial node perfectly tracable over its whole lifetime is probably worth that limit.

Sticking to a simpler code also has some potential IPR benefits.  ::shrugs::

-------------------------

ajtowns | 2026-06-22 03:00:05 UTC | #4


[quote="gmaxwell, post:3, topic:2624"]
I made a prior development list post where I proposed using RS codes
[/quote]

I think this is that post:

https://gnusha.org/pi/bitcoindev/CAAS2fgT5pJh68xufv_81+N8K0asxH16WdX7PLLXGjRPmJOkYFQ@mail.gmail.com/

[quote="gmaxwell, post:3, topic:2624"]
Bitcoin has to be robust against actively malicious peers. If you get data from a bunch of different peers and some are malicious there didn’t seem have anything short of exptime algorithms to find the set of malicious peers and successfully reconstruct the data in spite of them.
[/quote]

The blog post linked from the original message in this thread points at a [2019 paper](https://arxiv.org/pdf/1906.12140) that uses the header chain for solving this: you download a selection of "droplets" each represent an xor combination of 1-to-many blocks. You resolve the 1-block droplets immediately, and verify they're correctness versus the header-chain, then xor-those blocks out of the other droplets, reducing some of them down from an n-block droplet to an (n-1)-block droplet. Because you're only using validated data to affect droplets from other nodes, punishment is straightforward.

[quote="gmaxwell, post:3, topic:2624"]
The other reason was that fingerprinting is a concern-- when a node changes network identity it is preferable if it is difficult to distinguish from other nodes
[/quote]

That seems a fair criticism, particularly if the goal here is that every node (including non-listening ones) will eventually assist new nodes doing IBD. 

[quote="gmaxwell, post:3, topic:2624"]
Because of this I think it makes sense to limit the number of distinct shards. It does mean that distribution wouldn’t be as perfectly uniform where you need literally *any* N peers to reconstruct but I think not making every partial node perfectly tracable over its whole lifetime is probably worth that limit.
[/quote]

If you're just dividing non-archival nodes into N groups, then I think you'd just have each node in group k of N store any block whose height $h \equiv k \pmod{N}$, rather than doing anything more complex? With N=128, then that would mean storing ~6GB per node, and nodes who are happy to store ~44GB could store/serve $k, k+16, k+32, k+48, k+64, k+80, k+96, k+112$, while remaining indistinguishable amongst the 6.25% of nodes willing to store 44GB, ie the ones who chose the same k.

-------------------------

gmaxwell | 2026-06-22 03:15:46 UTC | #5

IIRC the figures you get for share of nodes you need to DOS to block synchronization is much worse than if you use a code.

-------------------------

ajtowns | 2026-06-22 03:32:51 UTC | #6

Presumably that only works if you have N anonymity groups, but only need to connect to $rN$ groups, with $r < 1$ (and you need a code to achieve that)? Then you need to DoS every node in $(1-r)N+1$ groups instead of just 1 group; though in either case you also need to DoS all the remaining full archival nodes as well, assuming there are any.

-------------------------

lucasdbr05 | 2026-06-22 04:14:00 UTC | #7

> I think the idea was that peers would share the seed they use for working out what droplets they'll select, both to save a little bit of data, but also to help guarantee unbiased random selection?

Yes. The peers share the seed used to generate the droplets in the respective epoch. In the post, where I talk about:

> Each droplet carries information about which original blocks were combined to produce it. Using this information, the decoder builds a bipartite graph connecting droplets to their source blocks.

this information might include the seed needed to rebuild the same bipartite graph on both nodes.

---
> So I think your storage is just the fraction 𝑠/𝑘 versus storing all blocks.

Actually it's approximately $s/k$, because blocks don't have exactly the same size, so different encodings produce droplets with variable size.
In the last paragraph you say that:

> I wonder what the impact of different serialization sizes of blocks is here: if you have a droplet that's [208, 820] and 208 is full of inscriptions and 4MB while 820 is full of runestones and 1MB, I think the droplet has to be the full 4MB.

That's one of the reasons the storage isn't exactly $s/k$ of the total blocks' data size.
In a case where a difference like the one you showed occurs, could this be optimized by concatenating blocks up to a target size threshold, to avoid XORing a large block against a much smaller one, where most of the operation would just be canceling out the zero-padding on the smaller block.

---
> I think it would be reasonable to parameterize based on 𝑘 and 𝑟 =𝑠/𝑘 instead, with the potential for 𝑟 being a per-node setting.

I agree with the above statement. The parameter $s$ being variable per node's storage capacity is really interesting. But I think using a fixed $k$ is important for node coordination.

---
> My first thought is that you'd disconnect and ban a peer that gave you bad data (so not request any additional droplets), but might keep the remaining droplets they gave you on the offchance any of them turned out to be useful.

I also agree with this statement. The remaining droplets from a murky peer should still be processed by the decoder, which will naturally accept them if they pass the header and merkle root verification when they become singletons, or reject them otherwise.

-------------------------

gmaxwell | 2026-06-22 06:00:50 UTC | #8

[quote="ajtowns, post:6, topic:2624"]
Presumably that only works if you have N anonymity groups, but only need to connect to rN groups, with r < 1 (and you need a code to achieve that)? Then you need to DoS every node in (1-r)N+1 groups instead of just 1 group; though in either case you also need to DoS all the remaining full archival nodes as well, assuming there are any.

[/quote]

Right, and slicing is on each block, so it doesn't matter that blocks are different in size.  Each block is encoded in say, 246 parts, such that you need any 10 parts to recover the block.  So to block reconstruction you must block reaching at least 96% of nodes (and all the archival nodes, if any exist).  And each node would need 1/10th the storage.  Discovery is not a big deal because just random selecting likely gives you 86% chance of getting all distinct selections even not having any idea what lane peers are in.

I hadn't seen the 2019 paper, but a pure peeling code has some unattractive recovery properties, e.g. 30%+ overhead in terms of symbol counts. And the scheme you describe has poor locality... so I don't see how storage would be handled during recovery on a pruning node. I guess I need to look at some concrete numbers.  But I don't see the advantage vs striping at a block level.

-------------------------

ajtowns | 2026-06-22 06:15:11 UTC | #9

[quote="ajtowns, post:6, topic:2624, full:true"]
Presumably that only works if you have N anonymity groups, but only need to connect to rN groups, with r < 1 (and you need a code to achieve that)? Then you need to DoS every node in (1-r)N+1 groups instead of just 1 group; though in either case you also need to DoS all the remaining full archival nodes as well, assuming there are any.
[/quote]

So I think a way of combining these ideas might be something like this:

 * Pick k=1008 blocks matching half a retarget interval
 * Pick N=100 as your anonymity groups
 * Pick 1.6% (1/63) as your storage reduction factor (~11GB total) for 16 droplets per interval (~32MB additional storage per week rather than ~2GB)
 * Calculate in advance (hardcode) 1600 unique droplet compositions according to the Robust Soliton distribution with $c=0.06 , \delta=0.1$ (versus $c=0.1, \delta=0.01$) using some deterministic pseudo-random scheme. Order these by degree, so $\deg(d_1) = 1$,  $\deg(d_{1600}) = 57$
 * Each node picks its anonymity set once from $\alpha \in 1..N$, and keeps droplet $d_i$ in each interval whenever $i \equiv \alpha \pmod{N}$.

With that setup, I believe you need to connect to 76-80 honest peers across different anonymity groups to be ~certain of recovering all data, but you can calculate exactly 1008 droplets to download and peel to obtain all the blocks you want, so you don't pay too much of a bandwidth cose. Meanwhile you detect a dishonest peer essentially as soon as you download a single bad droplet from them.

If you want to make $r$ be a per node parameter, then you could select $m \in \{1,2,5,10,20\}$ and multiplying your storage used by $m$ and storing droplet $d_i$ whenever $i \equiv \alpha \pmod{\frac{N}{m}}$.

With a hardcoded set of 1600 droplet compositions you might be able to do something better than random, but I'm not sure what that would look like.

I believe that needing 80 honest peers with distinct sets rather than 63 is the tradeoff between using an optimal encoding like Reed-Solomon versus being able to precisely detect who's responsible for introducing errors via the SeF-peeling scheme.

In practice this would mean during IBD you're downloading 1008 blocks at once, then processing them all, then downloading the next set. Picking a smaller size for k might be appropriate if that means you can keep the sequence of resolved blocks in memory for peeling, rather than having to reread it repeatedly off disk; after all, if you're interleaving processing the last set of 1008 while downloading the next, then the block cache for peeling would compete for memory with the leveldb utxo cache which might be annoying for (very) low-memory systems. Unfortunately dropping from $k = 1008$ significantly increases the number of peers you need or the amount they need to store afaics.

-------------------------

