# ZK-gossip for lightning channel announcements

halseth | 2025-01-28 13:10:04 UTC | #1

Hi, I would like to propose an extension to the taproot gossip proposal, that
makes it possible to avoid revealing the channel outpoint.

It is based on Utreexo and zero-knowledge proofs, and is accompanied with a
proof-of-concept Rust implementation.

The proposal is created as an extension to the gossip v1.75 proposal for
taproot channel gossip and intended to be used as an optional feature for
privacy conscious users.

### Taproot gossip (gossip v1.75)
See Elle's deep dive here: [Updates to the Gossip 1.75 proposal post LN summit meeting](https://delvingbitcoin.org/t/updates-to-the-gossip-1-75-proposal-post-ln-summit-meeting/1202).

Tl;dr: a new `channel_announcement_2` message that carries a Musig2 signature
proving the two nodes control a certain UTXO.

An example `channel_announcement_2`:
```json
{
  "ChainHash": "000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f",
  "ShortChannelID": "1000:100:0",
  "Capacity": 100000,
  "NodeID1": "0246175dd633eaa1c1684a9e15d218d247e421b009642f3e82e9f138ad9d645e03",
  "NodeID2": "02b651f157c5cf7bfbf07d52855c21aca861d7e3b8711d84cb04149e0a67151e16",
  "BitcoinKey1": "027fc1e3f6f5a67b804cb86d813adf94f89ea1de63f630607d4c31242d58306043",
  "BitcoinKey2": "03fef5084b98aa37757acce81d25148cfdb9592142567c6265e39b082e73c4d54",
  "MerkleRootHash": null,
  "Signature": {
    "bytes": "5c14ad15b614c9f91fd5c66b7bfe3f3552427c6d5e6d598f5838c5d219cdd0b89c72ad6a3effe5d995387563b80dfb1b59da599c936c705ad8dfd6da8288b89b",
    "sigType": 1
  },
}
```

## ZK-gossip
What we propose is an extension to the taproot gossip proposal, that makes it
possible for the two channel parties to remove the link between the channel and
on-chain output.

In order to still be able to protect the network from channel DOS attacks, we
require the channel announcement message to include a ZK-proof that proves the
inclusion of the channel in the UTXO set, and that it is controlled by the two
nodes in the graph.

In order to create the ZK proof with these properties, we start with the data
already contained in the regular taproot gossip channel announcment:

1) `node_id_1`, `node_id_2`
2) `bitcoin_key_1`, `bitcoin_key_2`
3) `merkle_root_hash`
4) `signature`
5) `capacity`

(we'll ignore the `merkle_root_hash` for now).

In addition we assemble a Utreexo accumulator and a proof for the channel
output's inclusion in this accumulator.

Using these pieces of data we create a ZK-proof that validates the following:

1) `bitcoin_keys = MuSig2.KeySort(bitcoin_key_1, bitcoin_key_2)`
2) `P_agg_out = MuSig2.KeyAgg(bitcoin_keys)`
3) Check `capacity <= vout.value`
4) Check `P_agg_out = vout.script_pubkey`
3) Verify that `vout` is in the UTXO set using the utreexo accumulator and proof.
4) `P_agg = MuSig2.KeyAgg(MuSig2.KeySort(node_id_1, node_id_2, bitcoin_key_1, bitcoin_key_2))`
5) Verify the signaure against `P_agg`
6) `pk_hash = hash(bitcoin_keys[0] || bitcoin_keys[1])`

We then output (or make public) the two `node_ids`, the signed data, utreexo accumulator and `pk_hash`.

Now we can output a proof (ZK-SNARK, groth16) of 256 bytes, and assemble a new
`channel_announcement_zk` (since messages are TLV, this should really be
combined with the `channel_announcement_2` with appropriate fields set):

```json
{
  "ChainHash": "000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f",
  "Capacity": 100000,
  "NodeID1": "0246175dd633eaa1c1684a9e15d218d247e421b009642f3e82e9f138ad9d645e03",
  "NodeID2": "02b651f157c5cf7bfbf07d52855c21aca861d7e3b8711d84cb04149e0a67151e16",
  "UtreexoRoot": "4b07311e3e774f0db3bcedee47c56fe30da326d9271d1a2558a5975ab4214478",
  "ZKType": "9bce41211c9d71e1ed07a2a5244f95ab98b0ba3a6e95dda9c87ba071ff871418",
  "ZKProof": "6508d8476086238076bb673005f9ef3bfe7f0c198a1d4f6fcee65e19478b422c512aefd004f8f476d0ef5939dc4339e3e19347a6ab60fe5714e9d3e3e77417499dbf18da68dfd942d79c8bf4cf811f615334f4643befb267a189d8e6b05509760bfd7add9aa9ecbce38db277bf11b1b94e147b504e75be5405066421aad8e10b49d105a33241742bafe611b4025ffa35d066fc87e11df595030d18b962ad5917ef1f73c97d660c1e62c7e392d51821ec342b2faf763d2a9177d13471c8b2a829578fd401d76aa8ae5642937f48573e657a5af14fda5f7a39216dda05b183121913088d2d0e0c1902d1f656b5d769b95040a40ef5a9ffd87f550545b0a5bc2505",
  "PKHash": "be7d602934c5ce95000ee989748f6c892ce16fb4276389ec15bc0764fbc4bea5"
}
```

`ZKType` is the unique identifier for the verification program, and is often a
unique hash of the binary representation of the verifier. This makes it easy to
move to a new proof type. `pk_hash` acts as a unique channel identifier.

(note: the `pk_hash` is not unique if the two nodes reuse their public keys for
a new output. Maybe this can be used to move the channel to a bigger UTXO
without closing it...)

### Handling received channel_announcement_zk
When a node receives a `channel_announcement_zk` message, it will first use
the `pk_hash` to check whether this is a channel already known to the node. The
`pk_hash` is deterministic and unique per channel. It will then verify the
proof if it has a type known to the node. Otherwise it will ignore it.

Since we can no longer detect channels closing on-chain, we must require
periodic refreshes of announcememnts, proving the channel is still in the utxo
set. It is an open problem what the requirement shoud be here, but I propose
setting this to around two weeks. With legacy channels we already have the
problem of not knowing whether a channel unspent on-chain is active, so some
kind of liveness signal is needed regardless.

### Proof-of-concept
I've preperad a branch with accompanying code and documents walking through the
process of creating a proof from the original channel announcment: [Musig2
example](https://github.com/halseth/output-zero/blob/15cfb6adcef11379c5601831a864e15fe09910dc/docs/musig2.md).

It is based on RiscZero, a versatile framework for creating proofs of execution
for RISC-V binaries. This means that is easy to add more contrainsts to the
verification of the UTXOs if useful.


### More details
More details regarding proof types, pros and cons and the proof-of-concept can
be found here: [Privacy preserving Lightning
gossip](https://github.com/halseth/output-zero/blob/15cfb6adcef11379c5601831a864e15fe09910dc/docs/ln_gossip.md)

See this thread for more info on the OutputZero project (formerly `utxozkp`): https://delvingbitcoin.org/t/proving-utxo-set-inclusion-in-zero-knowledge/1142 


Feedback much appreciated!

-------------------------

sanket1729 | 2025-01-28 22:28:00 UTC | #2

First off, awesome work with this. I have been brainstorming this specific issue for quite sometime now. Great to concrete progress here.

I will try out the code and ask specific questions in a followup reply, but here is the main one that I struggled with initially. 

How does this interact with lightning channel closures? I am not familiar with lighting routing messages for channel close, or if the node directly figures it out by observing the blockchain. 

For effective payment routing, the graph also needs to be updated with the closed channels, removing them network. Will the ZK benefits go away if the channel_id is leaked when we close the channel?

-------------------------

roasbeef | 2025-01-28 23:44:14 UTC | #3

> How does this interact with lightning channel closures?

IIUC, in this model, as nodes don't know which outpoint to watch, they won't be able to detect channel closures. Instead, they'll just drop _all_ channels every N weeks. This is a more agressive version of the graph pruning that nodes do today (dropping channels that you haven't received a channel update for in 2 weeks). As a result, nodes will now need to continually rebroadcast the announcement data for active channels periodically (today they only rebroadcast channel updates, which contain routing policy information). 

To my knowledge, some Lightning implementations today already implement such behavior: they won't watch for closes at all on-chain, and just drop channels after a period of time. When/if they encounter a channel that has been permanently closed on chain, they can use a unique error (non-temp) to penalize the channel to the point that the path finding model never uses it, or to just remove it all together. Proactive probing can help to discover such channels, enabling early pruning of the graph (vs at payment attempt time). 

As far as what's revealed on open, as formulated the scheme still has the channel capacity in plain text, with the verifier using that as public input to verify that the committed output has a value that matches the channel capacity. This leaks a bit of information as a verifier can scan on-chain using the utreexo checkpoint height as a starting point (new channel can only be created after the checkpoint). It's possible to omit the capacity field all together (or just allow it to float all together based on some multiple as suggested in earlier Gossip 2.0 proposals). Omitting it likely has some path finding implications (read: UX) as many path finding implementations input the capacity of a channel into a model to determine the conditional success probability of a payment given the capacity and past observations.

-------------------------

halseth | 2025-01-29 11:22:02 UTC | #4

[quote="sanket1729, post:2, topic:1407"]
How does this interact with lightning channel closures? I am not familiar with lighting routing messages for channel close, or if the node directly figures it out by observing the blockchain.
[/quote]

Currently, an LN node backed by a Bitcoin full node definitely will watch the chain and remove channels when their output is spent. Light clients can't effectively do this, and as roasbeef mentions fall back to using the `channel_updates` as a liveness signal (or use some trusted source for graph/routing information).


[quote="roasbeef, post:3, topic:1407"]
This is a more agressive version of the graph pruning that nodes do today (dropping channels that you haven’t received a channel update for in 2 weeks)
[/quote]

Exactly. A channel being unspent on-chain is no guarantee that it is still usable, so it is nice to get some kind of heartbeat from the nodes. This is why I think requiring a refreshed `channel_announcement` every few weeks is not a bad option.

[quote="sanket1729, post:2, topic:1407"]
Will the ZK benefits go away if the channel_id is leaked when we close the channel?
[/quote]
I definitely think that would be a big loss, since closing channels then will be discouraged since it hurts privacy.

[quote="roasbeef, post:3, topic:1407"]
As far as what’s revealed on open, as formulated the scheme still has the channel capacity in plain text, with the verifier using that as public input to verify that the committed output has a value that matches the channel capacity.
[/quote]
In my current implementation, I prove in ZK that the advertised capacity is _less or equal_ to the on-chain output. This leaves some obfuscation to what (taproot) outputs it could be. In a production implementation I would imagine a floating range could work, as roasbeef suggests. 

[quote="roasbeef, post:3, topic:1407"]
This leaks a bit of information as a verifier can scan on-chain using the utreexo checkpoint height as a starting point (new channel can only be created after the checkpoint).
[/quote]
It is worth mentioning that there's no way to tell whether the channel was open or not before the checkpoint. So a privacy conscious node could open the channel on-chain, but wait several weeks before announcing it (they could actually start using it as a private channel right away, then later make it public).

-------------------------

MattCorallo | 2025-01-29 15:25:42 UTC | #5

Why utreexo and groth16 vs much simpler UTXO snapshots and log-scale ring sigs? The ring sig approach has some overhead, but avoiding all the more complicated crypto seems well worth it (which we expect every lightning implementation to take as a dependency? bleh).

> As far as what’s revealed on open, as formulated the scheme still has the channel capacity in plain text, with the verifier using that as public input to verify that the committed output has a value that matches the channel capacity.

Right, if we're gonna go ZK gossip we should remove this leak. No one cares about having the *exact* channel capacity, we should limit the precision of the announcements if we're still gonna tie proofs to channels.

Really we should let people under-commit, though - commit to N BTC on chain, announce M*N BTC in channels (still probably with reduced precision).

-------------------------

halseth | 2025-01-30 13:34:13 UTC | #6

[quote="MattCorallo, post:5, topic:1407"]
Why utreexo and groth16 vs much simpler UTXO snapshots and log-scale ring sigs?
[/quote]

Utreexo is nice since it is deterministic and easy to maintain as a full-node (efficient updates to the accumulator). You need the accumulator both to create and verify proofs.

Groth16 was just used in this case since the proof-size is small and it is supported by the Risc0 framework. I made no other considerations for the POC. The proposal is meant to not be tied to any specific proof type. 

Regarding ring signatures as an alternative, I have not explored how you could use that in this setting. The nice thing about Risc0 is that you can selectively reveal what you want about the output and the Musig2 keys in play.

[quote="MattCorallo, post:5, topic:1407"]
Right, if we’re gonna go ZK gossip we should remove this leak. No one cares about having the *exact* channel capacity, we should limit the precision of the announcements if we’re still gonna tie proofs to channels.

Really we should let people under-commit, though - commit to N BTC on chain, announce M*N BTC in channels (still probably with reduced precision).
[/quote]
I agree. Luckily this is easy to do, by changing the proof assertion from 

`capacity <= vout.value` 

to 

`capacity <= M*vout.value`

-------------------------

MattCorallo | 2025-01-30 23:11:20 UTC | #7

[quote="halseth, post:6, topic:1407"]
Utreexo is nice since it is deterministic and easy to maintain as a full-node (efficient updates to the accumulator). You need the accumulator both to create and verify proofs.
[/quote]

Is this more efficient than, say, just building a fresh UTXO snapshot every 6 hours?

> Groth16 was just used in this case since the proof-size is small and it is supported by the Risc0 framework. I made no other considerations for the POC. The proposal is meant to not be tied to any specific proof type.

Sure, fair enough. We spent a long time discussing this a few years ago and basically the issue we ran into was that there was no suitable proof scheme that can be realistically included in lightning "consensus". That seems like the hardest part of all this, sadly.

> Regarding ring signatures as an alternative, I have not explored how you could use that in this setting. The nice thing about Risc0 is that you can selectively reveal what you want about the output and the Musig2 keys in play.

I'm not sure how useful that is here? We can instead split the UTXO snapshot into separate trees for all amounts in different amount ranges and prove against a single amount range.

-------------------------

halseth | 2025-01-31 13:25:10 UTC | #8

[quote="MattCorallo, post:7, topic:1407"]
Is this more efficient than, say, just building a fresh UTXO snapshot every 6 hours?
[/quote]

Depends on what you mean by "efficient". My first implementation did build a merkle tree from the full utxo set dump, and that took about 10 minutes every time if I remember correctly (not optimized in any way).

With utreexo you don't even need to hold the full utxo set to create proofs, so it is also compatible with light clients.

[quote="MattCorallo, post:7, topic:1407"]
I’m not sure how useful that is here?
[/quote]

It is useful in this setting since we can reveal the keys (node IDs) used to create the aggregated key. Maybe you could do that with a ring signature also, I will look into it.

[quote="MattCorallo, post:7, topic:1407"]
We can instead split the UTXO snapshot into separate trees for all amounts in different amount ranges and prove against a single amount range.
[/quote]
This is a great idea! I will look into that, thanks :)

-------------------------

AdamISZ | 2025-02-01 17:06:17 UTC | #9

[quote="MattCorallo, post:5, topic:1407"]
Why utreexo and groth16 vs much simpler UTXO snapshots and log-scale ring sigs? The ring sig approach has some overhead, but avoiding all the more complicated crypto seems well worth it
[/quote]

IMHO the juice isn't worth the squeeze with the log scaled ring signature option, because the verification time (and compute too) is unavoidably linear in the set size. I just don't see much point in getting ring sizes of say a few thousand, which is still fragile anon-set-wise, and then also pay a lot in verifying time. To quote myself from an old blog post that [implemented](https://gist.github.com/AdamISZ/03a5112172807ae614ccc34c55ed686e) triptych:

> You might naturally think, "cool, let's do 100K keys or even more; it should still be pretty small". That's true. But the computation and verification costs, as discussed above w.r.t. Bootle15, are approximately linear in (N) and that means this will become impractical somewhere, depending on your coding efficiency. My guess from playing with the code in the gist above, is that with properly optimised code/algos, we can stay sub 1-3 seconds for these operations as long as we're in the 10K region or perhaps a little larger. But that's already a *huge* practical achievement, in my opinion (and .. I'm guessing).

(from [here](https://reyify.com/blog/little-riddle/))

Having said all of that, I still do get your point. It is at least conceivable. And on reflection maybe I'm wrong that verifying time is so important; for *this* use case, I guess it's fine if it's somewhat slower(?).

@halseth a couple of Qs, as I'm also very interested to delve into understanding this better; it does look promising. Your [gist](https://github.com/halseth/output-zero/blob/15cfb6adcef11379c5601831a864e15fe09910dc/docs/musig2.md) has an 80 second prove time for the un-wrapped, correct? Does the wrapped version take more time, if so how much more? Do we have a path to getting these numbers down, do you think?

Clearly verif. is going to be cheap, and proof size naturally is at the ideal small O(1) for groth16. I need to study utreexo again and better understand how it fits into your circuit construction here before making other comments.

I'm glad to see in this thread there's discussion that realistically, we can just take regular snapshots of utxo set (or better, subset), and though we can't "read" channel closes in plaintext, it doesn't have to be a killer of this idea. For this reason I don't think either @halseth 's work here, or aut-ct should be dismissed as a possibility based on "it only proves ownership at one time". Other possibilities (like timelocks?) would I guess be really awkward.

Re: 

> Right, if we’re gonna go ZK gossip we should remove this leak. No one cares about having the *exact* channel capacity, we should limit the precision of the announcements if we’re still gonna tie proofs to channels.
Really we should let people under-commit, though - commit to N BTC on chain, announce M*N BTC in channels (still probably with reduced precision).

Wanted to mention, I added this kind of functionality - amount ranges, support for multiple utxos, in my aut-ct repo (see the `auditor-docs` subdirectory for details). People can generate concise proofs of aggregated-over-several-utxos total amount in a range (i.e. there is a range proof included in the bulletproofs circuit). If anyone's confused about how amounts fit in, just understand that the curve tree is not only a tree of pubkeys, it can be a tree of composite values (pubkey, amount) using a Pedersen commitment to both: C = P + vH + rJ for example. This idea is developed way further in @kayabaNerve 's [FCMP++](https://github.com/kayabaNerve/fcmp-plus-plus/tree/develop) work.

"Concise" in the above para. : maybe 2-4kB is not concise! There is some analysis [here](https://github.com/AdamISZ/aut-ct/issues/19). I always assumed that was fine, but for LN .. perhaps it just isn't (?), even if there are other advantages like < 100ms verification and 1-2 seconds for proof.

Question for everybody:

Something I often wondered is, could we use pairings based (e.g. KZG commitments) type stuff *if* we just make the tradeoff like this: every participant in LN, starting out with a new utxo or several, say, and has the utxo's pubkey sign a key on a pairing friendly curve. Then we can start using substantially more efficient crypto/ZK techniques with O(1) scaling for proofs and very cheap verifiers, but with the obvious tradeoff: the anon set is *only* the participants of "this system" (what is "this"? People would opt-in to adding their utxo to a list anonymously, and perhaps some altruistic volunteers also provide their non LN utxos to this list; kind of a fun thought!). An example of such a much-better-scaling system might be [Caulk](https://eprint.iacr.org/2022/621.pdf) (just one example I happen to know is const. proof size and verif time, and log(N) proving compute .. may not be the best).

-------------------------

MattCorallo | 2025-02-01 19:18:18 UTC | #10

>  Depends on what you mean by “efficient”. My first implementation did build a merkle tree from the full utxo set dump, and that took about 10 minutes every time if I remember correctly (not optimized in any way).

This seems pretty slow. Hashing the full UTXO set on my node (with a networkfed filesystem! though it seems to have mostly been hitting memory) only took 1:30. If we do that once every 6 hours that seems totally fine, honestly, especially since we'd restrict to a subset of the UTXO set (eg taproot-only, outputs above value X, etc).

[quote="AdamISZ, post:9, topic:1407"]
IMHO the juice isn’t worth the squeeze with the log scaled ring signature option, because the verification time (and compute too) is unavoidably linear in the set size. I just don’t see much point in getting ring sizes of say a few thousand, which is still fragile anon-set-wise, and then also pay a lot in verifying time.
[/quote]

Shucks. Then it seems we're right back where we were three years ago - this is in theory a great idea, we know we want to do it, we even have some ideas for optimizations for it, but ultimately there simply is no suitable proof scheme with a mature implementation space.

> And on reflection maybe I’m wrong that verifying time is so important; for *this* use case, I guess it’s fine if it’s somewhat slower(?).

This was, sadly, not my point. Verification in the 1-3 second range is already probably too slow here (though if we can batch, verification in the 60+ second range may be totally fine!).

> Something I often wondered is, could we use pairings based (e.g. KZG commitments) type stuff *if* we just make the tradeoff like this: every participant in LN, starting out with a new utxo or several, say, and has the utxo’s pubkey sign a key on a pairing friendly curve...the anon set is *only* the participants of “this system”

I mean its better than what we have today, so...sure? The real issue that we've run into is that we can only use stuff where there's robust implementations that all lightning nodes are happy to take as a dependency (and/or is simple enough for there to be multiple implementations!). In our previous searches, that was far from the case...there's plenty of ZK implementations of various algorithms, but most of them are kinda meh on quality, and the ones that are better are often more generic than we need (we could optimize a lot by not trying to verify secp256k1 curve ops in RISC-V) or impractical to build our own proof system using. The state of things may be better here than it was three years ago when we last looked, so happy to be proven wrong!

-------------------------

AdamISZ | 2025-02-01 23:00:58 UTC | #11

[quote="MattCorallo, post:10, topic:1407"]
Shucks. Then it seems we’re right back where we were three years ago - this is in theory a great idea, we know we want to do it, we even have some ideas for optimizations for it, but ultimately there simply is no suitable proof scheme with a mature implementation space.
[/quote]

I mean, sure, possibly. But from *my* point of view it's certainly not "we're in the same place as 3 years ago" because that's when I was looking at ring sigs for this idea and they never get past O(N) compute and verify. Curve Trees are to me a very clearly better version of that, with as they say in the paper "concretely small proving, verifying times". And since it can all be built on secp/secq (including bulletproofs circuits) it isn't dragging in a new area of dependencies to bitcoin coders (I mean, OK, *kinda*). As you put it, indeed, "we could optimize a lot by not trying to verify secp256k1 curve ops in RISC-V". Of course my aut-ct repo is little more than a hobbyist project, as it caveats it's not something ready for production use, and nor will it be without other people contributing/analyzing it. And apart from the aforementioned FCMP++ project (which, worth mentioning, at least has an audit! - not to mention generally being a more substantial project), there is not as you put it a "mature implementation space", i.e. who else is even using Curve Trees? But, that's a high bar for a quite novel concept in play here.

But of course this thread is about @halseth 's scheme with zkSTARKs so I'll stop talking about an alternative idea, here (there is another thread about that!).

> The state of things may be better here than it was three years ago when we last looked, so happy to be proven wrong!

Indeed it may be. But, the main thought that I have in reading your analysis here is, that I perhaps (can't speak for others in the discussion) don't know the set of criteria you (and other lightning engineers) have in mind of what exactly is acceptable and/or needed to do a ZKP of utxo ownership that's actually workable in LN (gossip).

> Verification in the 1-3 second range is already probably too slow here

OK that's what I originally thought, though I had other use cases in mind too. Yeah, that's part of why the "juice not worth the squeeze" comment.

-------------------------

AdamISZ | 2025-02-02 17:00:25 UTC | #12

@halseth one more question, I don't immediately know whether your scheme enforces distinctness/uniqueness? Does it output something like a nullifier or key image such that you can't reuse the (channel or otherwise) utxo more than once?

as per the readme anyway, you don't seem to be mentioning it?:


    The prover has a valid signature for an arbitrary message for a public key P, where P = x * G. The message and hash(x)is shown to the verifier.
    The prover has a proof showing that the public key P is found in the Utreexo set. The Utreexo root is shown to the verifier.

(In aut-ct I did a key-image "add-on" to the curvetree part, for this)

Edit: Oh, I guess hash(x) satisfies this purpose, right?

-------------------------

halseth | 2025-02-03 14:03:57 UTC | #13

Thanks for the insights! (and for finding a bug :smile: )

[quote="AdamISZ, post:9, topic:1407"]
Your [gist](https://github.com/halseth/output-zero/blob/15cfb6adcef11379c5601831a864e15fe09910dc/docs/musig2.md) has an 80 second prove time for the un-wrapped, correct? Does the wrapped version take more time, if so how much more?
[/quote]

Yes, that's correct. On my laptop (M1 Max, 32GB) creating the STARK creates about 80 seconds. The wrapped (groth16) version is currently not possible to create on non-x86 hardware. 

RISC0 provides a (trusted) proof server one can use. Using that server the fully wrapped proof is created in less than 30 seconds. I must assume that it is sporting som beefy GPUs, but it gives you an idea what is possible. I am aiming to rent som GPUs myself to see what proving times I can get to for the groth16 proof.

An interesting thing to note is that wrapping the proof (STARK->SNARK) reveals nothing about the private inputs, so one can imagine outsourcing the wrapping to a non-trusted server without giving up privacy.

[quote="AdamISZ, post:9, topic:1407"]
Do we have a path to getting these numbers down, do you think?
[/quote]
I'm certain these numbers can be brought down quite a lot. As of now I have spent close to no time profiling the ZK application. RiscZero has an extensive guide on how one can do this: https://dev.risczero.com/api/zkvm/optimization

In addition there are alternative implementations of the zkVM that claims to have better performance, like [SP1](https://github.com/succinctlabs/sp1). It would be worthwhile spending some time comparing them.

[quote="AdamISZ, post:9, topic:1407"]
“Concise” in the above para. : maybe 2-4kB is not concise! There is some analysis [here](https://github.com/AdamISZ/aut-ct/issues/19). I always assumed that was fine, but for LN … perhaps it just isn’t (?), even if there are other advantages like < 100ms verification and 1-2 seconds for proof.
[/quote]
My hunch is that we need verification times to be at least sub-second on "normal" hardware, preferably sub-100ms. Proof times I am not too worried about, several minutes is not a problem since you are waiting several blocks anyway before announcing it. Proofs are gossiped around so they should not be too big (and maybe they have to be less than 65kB to fit the LN max message size?).

-------------------------

halseth | 2025-02-03 14:13:28 UTC | #14

[quote="AdamISZ, post:11, topic:1407"]
But of course this thread is about @halseth 's scheme with zkSTARKs so I’ll stop talking about an alternative idea, here (there is another thread about that!).
[/quote]

It would be great to chat and discuss the various alternatives (STARKs, SNARKs, Curve Trees, ring sigs) and how we could use them to get to the goals we aim for (verification time, proof size, crypto maturity) for the scheme to be practical :smile:

-------------------------

halseth | 2025-02-03 14:24:44 UTC | #15

[quote="AdamISZ, post:12, topic:1407"]
@halseth one more question, I don’t immediately know whether your scheme enforces distinctness/uniqueness? Does it output something like a nullifier or key image such that you can’t reuse the (channel or otherwise) utxo more than once?
[/quote]
In this context (Musig2 for LN gossip) you should look at the [musig2 branch](https://github.com/halseth/output-zero/blob/15cfb6adcef11379c5601831a864e15fe09910dc/docs/musig2.md) and the [LN gossip document](https://github.com/halseth/output-zero/blob/15cfb6adcef11379c5601831a864e15fe09910dc/docs/ln_gossip.md) there.

Here we enforce uniqueness by hashing the public keys before they are aggregated to a taproot key: 

```
pk_hash = hash(bitcoin_keys[0] || bitcoin_keys[1])
```

These individual Musig2 public keys never go on-chain, so the idea is that no observer is able to link this hash with an on-chain output or spend.

-------------------------

AdamISZ | 2025-02-03 21:19:50 UTC | #16

[quote="halseth, post:13, topic:1407"]
Proof times I am not too worried about, several minutes is not a problem since you are waiting several blocks anyway before announcing it.
[/quote]

I've vacillated on this one. While in principle, yes, proofs can be slow, and 100% for sure we can allow them to be far slower than verifies, I'm not convinced that times of 1min + will be viable. Consider that some hardware running LN is a lot weaker, but even if that concern can be sidestepped, there's something a bit impractical about *any* CPU intensive operation taking double digit seconds, no matter if it's non urgent. Still, I do get your argument, as per the docs:

> It should be noted that only nodes announcing public channels need to do this, and they usually require a certain level of hardware to be effective routers anyway.

I think it is a bit debatable though. Glad to hear substantial performance improvement is a real possibility as per your optimisation comments.

On to:


> Here we enforce uniqueness by hashing the public keys before they are aggregated to a taproot key:

```
pk_hash = hash(bitcoin_keys[0] || bitcoin_keys[1])
```

This feels a bit flaky. In the docs of the non-MuSig version I see you did hash(x) where x is privkey (for single control utxo), which to me is kind of "the" way to do it; a key image is, functionally, almost exactly the same as a hash of a private key.

But hashing public keys is kind of a violation of expectations that could screw up connected protocols; if the pk_hash value can leak whenever those keys are derivable or directly seen. (I'm considering the nuances of BIP32, but who knows how else they could leak - public keys are public!).

Here's the part that might be interesting to you (it was to me!): I didn't even *consider* the proof-creation-by-channel-parties-together idea. My thinking was (a) taproot + musig2 making channel utxos the same as other utxos and (b) 2-party construction of proof is a nightmare, so just make proofs over *other* utxos that happen not to be channels. In an ideal world it doesn't make a difference, but I neglected to consider: **utxos that are not channels are extra liquidity cost** for actual Lightning operators!

So while that's not a slam dunk argument for "supporting musig 2-party outputs is required", it's a pretty *good* argument, and if the *only* thing sacrificed is that slightly flaky version of a key image, it ... may be OK?

(Edited to add: another reason I forgot to mention, for *not* using channel utxos, is that updates to one's sybil-resistance ZKproof would not be correlated with channel opens/closes, and such correlation has the potential to remove the ZK-ness)

-------------------------

AdamISZ | 2025-02-03 17:01:05 UTC | #17

Hmm. Could the 2 counterparties maybe just ECDH to add a third term to pk_hash? Or just directly use ECDH(bitcoin key 1, bitcoin key 2), then use that as the key image instead? Would that be much extra work for the zkSTARK part considering it's EC operations?

-------------------------

halseth | 2025-02-04 14:45:08 UTC | #18

[quote="AdamISZ, post:16, topic:1407"]
I’m not convinced that times of 1min + will be viable.
[/quote]

Considering 
- opening channels is not something you do often AND
- you wait minimum 6 blocks before announcing the channel AND
- you should perhaps wait even longer to increase your anonymity set AND
- either channel party can create the proof (the one with the beefiest hw) AND
- it can likely be optimized quite a lot AND
- hardware acceleration tends to become more widespread

I think we can quickly get proving times down to something reasonable. 

[quote="AdamISZ, post:16, topic:1407"]
This feels a bit flaky. In the docs of the non-MuSig version I see you did hash(x) where x is privkey (for single control utxo), which to me is kind of “the” way to do it; a key image is, functionally, almost exactly the same as a hash of a private key.
[/quote]
I understand why you say that. I myself spent some time convincing myself this was safe (I'm happy to hear a counterargument). 

The thinking here is that these public keys are never revealed to anyone other than the two channel counterparties (and they have all the information to leak the existence of the channel anyway of course), hence it is "semi-private".

If they leak, then an observer can obviously link the channel to the output. But that is still not worse than today, where the link is already public and gossiped around.

I do get the confusion that a "public key that should be kept private" could bring though. And I believe you could indeed do a ECDH as an alternative. The problem with that as I see it, is that you need to give the prover access to the private key (or give it access to APIs that can perform the ECDH). With the hash of public keys we currently do  the LN implementation can just spit out a regular channel accnouncement, then this external tool can convert it to a ZK proof before it gets broadcasted.

-------------------------

halseth | 2025-02-04 14:47:45 UTC | #19

[quote="AdamISZ, post:16, topic:1407"]
Here’s the part that might be interesting to you (it was to me!): I didn’t even *consider* the proof-creation-by-channel-parties-together idea. My thinking was (a) taproot + musig2 making channel utxos the same as other utxos and (b) 2-party construction of proof is a nightmare, so just make proofs over *other* utxos that happen not to be channels. In an ideal world it doesn’t make a difference, but I neglected to consider: **utxos that are not channels are extra liquidity cost** for actual Lightning operators!

So while that’s not a slam dunk argument for “supporting musig 2-party outputs is required”, it’s a pretty *good* argument, and if the *only* thing sacrificed is that slightly flaky version of a key image, it … may be OK?
[/quote]

I must confess I'm not understanding what you are saying here :sweat_smile: Could you explain?

-------------------------

AdamISZ | 2025-02-04 15:03:32 UTC | #20

> I must confess I’m not understanding what you are saying here :sweat_smile: Could you explain?

Oh sure. Let me write it out carefully (though you probably already know most of it) because I'm not exactly sure which part didn't click with you:

 If the possession of a channel opening utxo (proved by signing) is sufficient to grant me access to X in the LN protocol, we want to replace that with "ZKP of possession of a channel opening utxo", for better privacy. But, why does that "gate" exist? Only because it imposes a cost on spamming. It isn't the case that, logically, the utxo *must* be a channel-related utxo, in fact. Because from an attacker's point of view, creating a channel is the same cost as creating a utxo. So far so good? This implies we don't *have* to use channel utxos and therefore MuSig2 negotiation. 

Further, because we're considering a taproot scenario the distinction between one type of utxo is not revealed, in the happy paths.

So in my work I went to the other extreme, considering only a single sig generated case.

Then we get into the capacity and cost consideration. As you yourself noted, we don't want to leak the exact channel amount, defeating the purpose. We can use multipliers or ranges. E.g. @MattCorallo above comments about M*N worth of channels for one N utxo. I was just noting that economically it seems a bit "off" to not make use of channel utxos in the proofs, but privacy wise it might be better.

So at the end there I'm trying to say we, maybe, do indeed need MuSig2 support in the ZKP scheme because it's not economical to not use channel utxos (but tbh I'm not entirely sure).

-------------------------

Davidson | 2025-02-05 00:35:23 UTC | #21

Hi @halseth thanks for the impressive work you've been putting on this zk prover.

One major advantage I can think of using this approach, is that it works out-of-the-box with extremely lightweight nodes like [floresta](https://github.com/vinteumorg/Floresta) and [utreexod](https://github.com/utreexo/utreexod), since the utreexo state is all context you need to verify the proof. This would be beneficial for resource-constrained lightning nodes, as they may not have full access to the UTXO set.

Just out of curiosity: have you benchmarked this using an algebraic hash function? `rustreexo` now (since October last year to be precise) lets you choose your custom hash function, and algebraic hashes are waaaayyy lighter when running inside a prover. I wonder how much proving time is due to utreexo proofs. 

I see you're using my bridge node for proof generation, it supports Poseidon 2 as hash function and puts everything in a nice json format intended for provers (this was developed for the folks at starkware for their bitcoin zk prover), just need to toggle the `shinigami` feature (it won't build the API tho).

-------------------------

halseth | 2025-02-05 16:46:45 UTC | #22

Aha, I get what you are saying. Using a different UTXO than the channel output as the "anti-spam cost".

However, I'm not sure if it give us anything (in a non-ZK scenario it obviously could):
- Since channel counterparties _have to_ create a multisig output in order to open a channel between them, you already have a utxo available for this purpose. (if we wanted to allow channels not backed by a real utxo this would not be the case of course...)
- Channels are often created between non-trusted parties. Who would put up a utxo in that scenario? If both had to that would be less scalable of course.

-------------------------

halseth | 2025-02-05 16:48:23 UTC | #23

Thanks!

I have not tried the algebraic functions, but it is on my list of optimizations to try!

Great to hear that rustreexo has it available already, that will make it a lot easier to test out :smile:

-------------------------

AdamISZ | 2025-02-05 23:33:28 UTC | #24

[quote="halseth, post:22, topic:1407"]
Channels are often created between non-trusted parties. Who would put up a utxo in that scenario? If both had to that would be less scalable of course.
[/quote]

I had been working with a mental model based on what I read a long time ago [here](https://diyhpl.us/~bryan/irc/bitcoin/bitcoin-dev/linuxfoundation-pipermail/lightning-dev/2022-February/003470.txt) (Rusty's gossip v2 proposal, I think the original one? Hard to keep track).

To crib from there:

> 1. Nodes send out weekly node_announcement_v2, proving they own some
   UTXOs.
> 2. This entitles them to broadcast channels, using channel_update_v2; a
   channel_update_v2 from both peers means the channel exists.
> 3. This uses UTXOs for anti-spam, but doesn't tie them to channels
   directly.
> 4. Future ZKP proofs are could be added.

He also has a concept of claiming the channel on open at the end of that post.
Obviously in that model, my perspective should make a lot more sense.

But it seems pretty clear you're looking at it differently (as per the OP, v1.75 but with zkp added in). Responding more directly to your Q, though, I think you could make this argument: single funder provides zkp, dual funders have the starting channel balance reflect the asymmetric cost of one funder making a single proof; but as you can see this is fudgy, and maybe I'm wrong. I personally find Rusty's model makes a lot more logical sense, taking anti-DOS as far away as possible from the blockchain and making capacity claims vague).

-------------------------

halseth | 2025-02-06 11:42:05 UTC | #25

[quote="AdamISZ, post:24, topic:1407"]
I personally find Rusty’s model makes a lot more logical sense, taking anti-DOS as far away as possible from the blockchain and making capacity claims vague).
[/quote]

I think that model makes a lot of sense in the non-ZKP scenario, since you have to reveal _some utxo_ but allow it to be a different one from the channel.

In the ZKP setting I think it is less efficient, as using the actual channel utxo as stake doesn't reveal it. 

Also, as you mention, this proposal doesn't change the trust model as much; each channel is still backed by a single UTXO, you just don't reveal which one.

-------------------------

halseth | 2025-02-06 13:44:44 UTC | #26

[quote="Davidson, post:21, topic:1407"]
Just out of curiosity: have you benchmarked this using an algebraic hash function? `rustreexo` now (since October last year to be precise) lets you choose your custom hash function, and algebraic hashes are waaaayyy lighter when running inside a prover. I wonder how much proving time is due to utreexo proofs.
[/quote]

![Screenshot 2025-02-06 at 14.41.45|690x306](upload://91VOuBcnHGDMFjAeKhmx4Yc1GUt.png)

From profiling looks like most of the proving time comes from key aggregation (to verify the Musig signature), while utreexo verification and SHA-256 play a much smaller part.

-------------------------

Davidson | 2025-02-06 17:11:33 UTC | #27

Oh, nvm then. Def not worth it.

It actually makes sense, once ECC enters the game, everything else becomes irrelevant performance-wise. (It's good to know my code isn't the bottleneck tho :joy: ).

-------------------------

