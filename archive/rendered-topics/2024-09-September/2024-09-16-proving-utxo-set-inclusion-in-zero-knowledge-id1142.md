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

