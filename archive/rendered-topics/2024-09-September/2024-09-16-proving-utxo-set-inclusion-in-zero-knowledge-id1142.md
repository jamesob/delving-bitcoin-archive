# Proving UTXO set inclusion in zero-knowledge

halseth | 2024-09-16 13:01:59 UTC | #1

I’ve been working on a proof-of-concept for proving UTXO set inclusion in zero knowledge, and I feel it is now ready for more eyes: https://github.com/halseth/utxozkp

The `utxozkp` tool was primarily made as a research effort to prototype a possible way of making Lightning channel announcements more private. With this in hand one can prove that a LN channel exists on-chain without revealing the specific UTXO.

It has various other applications as well of course - any setting where UTXO set inclusion can be used as an anti-DOS measure would benefit.

Note that in its current form one only proves that the prover can sign *some* UTXO, and there are no guarantees about the UTXO not being reused for multiple proofs. In the future the aim is to generalize this enough to make it possible to selectively reveal information like UTXO size, tapscript paths, output age etc.

**Proving time and proof size**

The tool is very much a rough draft at this point. On my laptop (Apple M1 Max, 32GB), proving time is around 6 min, and the result is a proof of UTXO ownership of 1.4 MB. Verification is much quicker at around 300 ms. Both timings assuming the UTXO set is already loaded into memory as a Utreexo data structure.

The running time and resulting proof size have intentionally not been optimized until the direction of the project is decided.

**Architecture**

`utxozkp` is based on the [RISC Zero](https://github.com/risc0/risc0) STARK platform and [Utreexo](https://dci.mit.edu/utreexo). For more information check out the [README](https://github.com/halseth/utxozkp/blob/c3402a72005d8d1058f758ed277df9e6cafcac72/README.md).

Any feedback and suggestions for the path forward are welcome!

-------------------------

1440000bytes | 2024-09-16 22:06:50 UTC | #2

[quote="halseth, post:1, topic:1142"]
**Proving time and proof size**

The tool is very much a rough draft at this point. On my laptop (Apple M1 Max, 32GB), proving time is around 6 min, and the result is a proof of UTXO ownership of 1.4 MB. Verification is much quicker at around 300 ms. Both timings assuming the UTXO set is already loaded into memory as a Utreexo data structure.
[/quote]

How does this compare with [aut-ct](https://github.com/AdamISZ/aut-ct)?

-------------------------

ajtowns | 2024-09-17 02:00:23 UTC | #3

I think in comparison to [taproot ring signatures](https://github.com/jonasnick/taproot-ringsig) this doesn't require that the verifier have the full utxo set, just the corresponding "utreexo" root, which seems like a win.

I don't think it's quite accurate to call this "utreexo":

> The tool works with the UTXO set dump from Bitcoin Core. It uses this dump to create a [Utreexo](https://dci.mit.edu/utreexo) representation of the UTXO set

The normal utreexo construction depends on the entire history of the utxo set, and can't just be reconstructed from the current utxo set (otherwise you'd often need to rearrange utxos that weren't touched by a block when figuring out the new utreexo structure after a block comes in). So I think this is just a regular merkle tree of the utxo set, not really a "utreexo" thing per se.

I don't see how this really works for lightning channel announcements -- can't you just give a proof for a utxo as at block X, then spend it in block X+1? ie, at best, isn't this just proof-of-use-of-blockspace?

Also, can't you use the same utxo multiple times with different blinding factors to advertise multiple lightning channels? If you make the "verifier's public key" (`P'` in `P' = P+bG`) be the channel's advertised public key (`musig(A,B)`?) that might be good enough to prevent selling utxos.

-------------------------

ariard | 2024-09-17 07:35:03 UTC | #4

[quote="halseth, post:1, topic:1142"]
and there are no guarantees about the UTXO not being reused for multiple proofs.
[/quote]

So no guarantee that the utxo set inclusion proof cannot be double-spend, and as such a `channel_announcement` being unboundedly replay to dos lightning nodes.

-------------------------

halseth | 2024-09-17 08:34:23 UTC | #5

aut-ct is much faster and succint AFAIK (~kB proof instead of ~MB proof, seconds to prove instead of minutes).

The difference is that aut-ct only works on sets of public keys, so you cannot selectively reveal anything else about the output.

utxozkp is based on a more general proof of computation model, so you can choose to reveal more about the output (e.g. that the output holds at least 1 BTC) (note: not yet implemented).

-------------------------

halseth | 2024-09-17 08:50:38 UTC | #6

> So I think this is just a regular merkle tree of the utxo set, not really a “utreexo” thing per se.

Yes, you are right. This could have been done with just a regular merkle tree over the UTXO set, but I chose to use the utreexo data structure since it it is pretty straight forward to convert it to the "proper" utreexo representation that is per block deterministic.

> I don’t see how this really works for lightning channel announcements – can’t you just give a proof for a utxo as at block X, then spend it in block X+1? ie, at best, isn’t this just proof-of-use-of-blockspace?

This is a very good point. I can prove that a channel was opened, but there is no guarantee that it will stay open. Maybe one could require the proofs to be "refreshed" every X blocks? Each proof could also contain a proof of channel-age.

> Also, can’t you use the same utxo multiple times with different blinding factors to advertise multiple lightning channels? If you make the “verifier’s public key” (`P'` in `P' = P+bG` ) be the channel’s advertised public key (`musig(A,B)` ?) that might be good enough to prevent selling utxos.

The current setup is proving that the proper public key (not the blinded one) is commited to in the utreexo structure, so the blinding factor doesn't matter. But there is currently no way of telling whether a public key has been reused for multiple proofs, so that should be added for the LN use case.

-------------------------

halseth | 2024-09-17 08:55:29 UTC | #7

Correct, but fixing this should be doable:

A simple (naive?) solution could be to publish a hash of the private key along with the proof, and include in the proof that it corresponds to the non-blinded public key.

I'm sure there are better ways.

-------------------------

Davidson | 2024-09-18 17:19:19 UTC | #8

Pretty cool to see someone looking at this. I did [start](https://github.com/Davidson-Souza/zktreexo) something similar, but never finished. My goal was to commit the private key by hashing it at the end, so it couldn't be replayed. I also commit to the current accumulator, so you can know exactly up to which height it is valid.

As already pointed out, the `utreexo` accumulator is dynamic and needs to be computed from all blocks. Just as a heads-up, you can use [utreexod](https://github.com/utreexo/utreexod) or [floresta](https://github.com/Davidson-Souza/Floresta) if you want to get the current roots and prove a given utxo. Furthermore, the commitment scheme used in utreexo is [a bit different](https://github.com/vinteumorg/Floresta/blob/5fa8c546ef4aa25eda8a929c443a189d49e04530/crates/floresta-chain/src/pruned_utreexo/udata.rs#L50-L58) (I keep pondering if I should add this to `rustreexo` or keep it as "we only have the accumulator algorithms").

-------------------------

AdamISZ | 2024-09-18 17:34:41 UTC | #9

Hi @halseth ;

First thanks for publishing this, super interesting. I'll read up on it.

Second, on: 

> The difference is that aut-ct only works on sets of public keys, so you cannot selectively reveal anything else about the output.

The structure is more flexible than that; indeed, I've just written a [blog post](https://reyify.com/blog/privacy-preserving-proof-of-taproot-assets) (code impl. and paper are linked in there), concretely illustrating how you can use the same techniques to not only prove statements about individual utxos with certain properties (age, size) but also, statements about aggregates over groups (so that one is "proof that N utxos have total sats value in range x to y").

To state the perhaps obvious, the way to do that is just to add other constraints into the bulletproofs that you are using to proof membership in the curve tree.

Obviously that work is very speculative in its details, but I think the basic principle is pretty clear.

-------------------------

AdamISZ | 2024-09-18 19:18:58 UTC | #10

[quote="ariard, post:4, topic:1142"]
So no guarantee that the utxo set inclusion proof cannot be double-spend, and as such a `channel_announcement` being unboundedly replay to dos lightning nodes.
[/quote]

[quote="ajtowns, post:3, topic:1142"]
Also, can’t you use the same utxo multiple times with different blinding factors to advertise multiple lightning channels? If you make the “verifier’s public key” (`P'` in `P' = P+bG`) be the channel’s advertised public key (`musig(A,B)`?) that might be good enough to prevent selling utxos.
[/quote]

This is already addressed in the aut-ct construction with key images; the application stores a flat file database of those key images. The same  could also be done here afaik. You attach a trivial sigma-protocol type proof of DLEQ between a key image $I$ and the non-blinded part of the blinded commitment to $P$, i.e. proof that $C = xG + bH$ AND $I = xJ$.

[quote="ajtowns, post:3, topic:1142"]
I think in comparison to [taproot ring signatures ](https://github.com/jonasnick/taproot-ringsig) this doesn’t require that the verifier have the full utxo set, just the corresponding “utreexo” root, which seems like a win.
[/quote]

Yes; also in Curve Trees the verifier can verify just against the root, it's the same principle.

But (maybe stating the obvious?) basic AOS style ring signatures never really felt viable for these tasks, since they scale linearly in the anonymity set, so you can only get quite trivially small anonymity sets, which are probably too fragile to claim any real anonymity. Moreover, in some kind of flat network structure of interaction (like Lightning) you need verification of others' proofs to be fast, so that you can't get DOS just with *claims* of ownership etc. That's why I gravitated towards the Curve Tree structure. This alternative STARK direction could well be viable too.

[quote="ajtowns, post:3, topic:1142"]
I don’t see how this really works for lightning channel announcements – can’t you just give a proof for a utxo as at block X, then spend it in block X+1? ie, at best, isn’t this just proof-of-use-of-blockspace?
[/quote]

This feels like the most important question. Currently announcements are made publically of channel utxos. Assuming the key image (no double spend) thing mentioned above, how much worse is it to announce privately a utxo of the same size, for DOS resistance?

My (slightly woolly) thinking on this was always, while simply announcing money owned, not having to spend it, is obviously a vastly smaller cost, you can filter and control this to some extent: filters by age and by value can be included in your merkle/curve tree setup to make it that only "higher quality" utxos are allowed (e.g. amount $=T^aV^b$ for age $T$, size $V$, perhaps). But .. how is this defence really worse than the advertisement of "real" channels (which after all is not a meaningful distinction, in taproot/musig land, right?).

-------------------------

halseth | 2024-09-24 20:08:00 UTC | #11

> As already pointed out, the `utreexo` accumulator is dynamic and needs to be computed from all blocks. Just as a heads-up, you can use [utreexod ](https://github.com/utreexo/utreexod) or [floresta ](https://github.com/Davidson-Souza/Floresta) if you want to get the current roots and prove a given utxo. Furthermore, the commitment scheme used in utreexo is [a bit different ](https://github.com/vinteumorg/Floresta/blob/5fa8c546ef4aa25eda8a929c443a189d49e04530/crates/floresta-chain/src/pruned_utreexo/udata.rs#L50-L58) (I keep pondering if I should add this to `rustreexo` or keep it as “we only have the accumulator algorithms”).

Thanks, I'll update it to use the dynamic accumulator :slight_smile:

> Pretty cool to see someone looking at this. I did [start ](https://github.com/Davidson-Souza/zktreexo) something similar, but never finished. My goal was to commit the private key by hashing it at the end, so it couldn’t be replayed. I also commit to the current accumulator, so you can know exactly up to which height it is valid.

I added the private key hash at the end as well, as an attempt to create a private identifier for the output: https://github.com/halseth/utxozkp/commit/6fd00e2077dc82a46074a81b6f01ede316631a39

This should work since we are checking that the private key used as a preimage to the hash function also gives us the public key that is proven to be in the UTXO set.

-------------------------

halseth | 2024-09-24 20:08:42 UTC | #12

> The structure is more flexible than that; indeed, I’ve just written a [blog post ](https://reyify.com/blog/privacy-preserving-proof-of-taproot-assets) (code impl. and paper are linked in there), concretely illustrating how you can use the same techniques to not only prove statements about individual utxos with certain properties (age, size) but also, statements about aggregates over groups (so that one is “proof that N utxos have total sats value in range x to y”).

Very cool, did not know that was possible! :star_struck: 

What are the limitations to what you can prove about the UTXO using aut-ct? (at the extreme, can you prove something like "I know a witness that let me execute the script to this utxo" ?)

-------------------------

halseth | 2024-09-24 20:28:57 UTC | #13

> This feels like the most important question. Currently announcements are made publically of channel utxos. Assuming the key image (no double spend) thing mentioned above, how much worse is it to announce privately a utxo of the same size, for DOS resistance?

The way current channels are announced, they prove that a certain UTXO on-chain opens a channel, such that nodes watching the chain can know when that UTXO is spent and prune the channel from their graph accordingly.

In the zero-knowledge setting we can still prove that a UTXO was created ("I opened a channel between block height `X` and `X+10`"), but as a LN node maintaining a channel graph you cannot know when channels close. 

To a certain degree this problem exists already, in case a channel "dies" without being closed on-chain (nodes go offline, key loss etc). So as a good steward of your channel graph, you should probably be maintaining it and pruning inactive channels from your graph regardless, so maybe a heuristic like this can make up for not knowing whether it was closed on-chain.

As the public channel graph grows it will most likely be infeasible to keep track of every channel in any case, so one needs to be picky about which ones to keep.

Light client LN nodes are already using certain heuristics AFAIK to prune the graph, for example by removing channels where a channel update hasn't been sent recently.

-------------------------

AdamISZ | 2024-09-24 20:53:35 UTC | #14

[quote="halseth, post:12, topic:1142"]
What are the limitations to what you can prove about the UTXO using aut-ct? (at the extreme, can you prove something like “I know a witness that let me execute the script to this utxo” ?)
[/quote]

At a rough assessment, without delving into detail, it will be definitely straightforwardly possible for sPKs that are just logical conjunctions of keys or otherwise use EC arithmetic (thinking taproot style tweaks), since that's in EC-land so to speak; for sPKs that are based on hash locks, then clearly it's problematic for the usual reason: proofs of non-algebraic hashes (like SHA2) in such a system are extremely unwieldy. (which is of course why i use only taproot anon sets in these examples).

-------------------------

AdamISZ | 2024-09-24 20:57:13 UTC | #15

[quote="halseth, post:13, topic:1142"]
In the zero-knowledge setting we can still prove that a UTXO was created (“I opened a channel between block height `X` and `X+10`”), but as a LN node maintaining a channel graph you cannot know when channels close.
[/quote]

Yes after writing my note I did realize that that's the obvious answer to "isn't a private version kinda the same?" is: It isn't the same, for that reason.

But there is a (slightly weak) answer to that: if proofs have to be generated at intervals, e.g. let's say we are optimistic about performance of this approach and imagine it's OK to have a 24 hour update cycle, you could require everyone to reproduce proofs based on utxo snaphots on that cycle. Maybe that's unrealistic, I wouldn't be at all surprised, for a very large network like Lightning. But there are presumably all kinds of finesses/details here, like what you mention in the remainder of your post.

-------------------------

halseth | 2024-10-14 13:03:27 UTC | #16

The repo has been updated to do the schnorr signature verification in the ZK environment. This lets us avoid the blinding of the public key (since we are just proving that we have a valid signature for some public key that exists in the UTXO set, without revealing the key).

Together with an optimized library for schnorr signatures, this reduces proving time to ~50 sec on my machine: https://github.com/halseth/utxozkp?tab=readme-ov-file#benchmarks-apple-m1-max

-------------------------

