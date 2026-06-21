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

