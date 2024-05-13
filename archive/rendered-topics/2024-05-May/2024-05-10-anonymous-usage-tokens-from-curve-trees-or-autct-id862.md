# Anonymous usage tokens from curve trees or autct

AdamISZ | 2024-05-10 14:50:11 UTC | #1

Ever since whenever (I think when we introduced fidelity bonds in Joinmarket), I've been researching the best way to have private proof of pubkey ownership.

I did also think the same issue arises a lot in Lightning, potentially in solving jamming but also (as others pointed out to me), even more obviously in gossip of channels.

I think this may be a decent partial solution, to the problem "how can we gossip channels without revealing the utxos and without allowing people to sybil with fakes (too much)?".

What stopped earlier ideas I had from being really practical (see my earlier post on RIDDLE **) was that all the ring signature based stuff fails on sublinear verification. Crudely, it always requires the verifier to "read through" an object whose size is the size of the ring. So while an anon set of 1000 was fine and very compact in proof size, an anon set of 1M was wildly impractical.

But the [Curve Trees](https://eprint.iacr.org/2022/756.pdf) approach solves this: think of it as a merkle tree except algebraic (using points not hashes), which allows you to create ZKPs (in this case, arithmetic circuits using bulletproofs) about the object (key) that you're proving is in the tree. So here you get the statement "this pubkey is one of the 100K pubkeys in this set of taproot outputs, but you don't know which one, and this "key image" (think token) I'm providing, is tied to that un-revealed pubkey so I can't just use it again". I'll mention how that "tie"-ing works in a moment.

So the obvious application in Bitcoin is - make a proof of ownership of one taproot utxo pubkey out of N where N can be any large number up to the size of the utxo set. Given how many taproot utxos are dusty and let's say "uneconomic" you can just set a minimum value filter to get a sensible size. A couple months ago I got these values:

```
Amount in sats    Number of utxos(taproot only)
> 5 million         51674
> 2.5 million       81512
> 1 million         154130
> 500k              238060
> 250k              352235
> 100k              800843
> 50k               1043038
> 25k               1333547
> 10k               2853756
> 1000 sat          6084116
> 100 (i.e. ~all)   39034007
```

So as an example, a 500k sat threshold ends up with 240000 keys which is already practical for curve trees.

Re: "tie"-ing: Now the curve trees proof is a proof of *set membership* and doesn't enforce scarcity, since by its nature it can be repeated without revealing re-use. For this reason I added a DLEQ part, similar to Joinmarket's PODLE (think of this as like "linkable" in a ring signature), that creates an extra curve point that's provably the "key image" of the key whose membership you proved, without revealing that key. See [here](https://github.com/AdamISZ/aut-ct/blob/7e673b572a300fd43f0ba4b1839fe682be676d93/aut-ct.pdf) for the writeup of that algorithm; it's using completely standard sigma protocol techniques.

In practice I have done a lot of tests with keyset sizes from 50K up to 2.5M and seen verification time mostly sitting between 40 and 70ms although these aren't properly controlled scientific results; the point is that it remains fast! See Table 1 in the Curve Trees paper for their benchmarking results for the secp/secq cycle which are significantly better, even. The main point is that the logarithmic scaling coming from the tree, combined with how bulletproofs can be batch verified (a separate proof for each curve), gives extremely good scaling (it's a little difficult to write down a formula because of the arithmetic circuit; basically 1M keys are easily practical for verifying).

Working code at [my aut-ct github repo](https://github.com/AdamISZ/aut-ct). This is using as a basis the [benchmarking code](https://github.com/simonkamp/curve-trees/) of the authors of the curve trees paper.

It should be easy to run tests, I hope, following the README. There are a couple of toy small and medium size keysets you can use to play with if you want. Forgive my Rust :)

I even went as far as to make a proof of concept website where you sign up for a forum using a proof of taproot key ownership, see [hodlboard](https://hodlboard.org). Note that the process of creating a proof here turned out to be way more convoluted as you have to be able to access a private key for a taproot utxo you own, which to say the least, is not supported by any wallet (except Core, with multiple steps on the command line!). However this practical annoyance will be irrelevant for any usage of this protocol in e.g. Lightning. It would only be relevant if we want interoperable creation of such proofs at user level.

As I understand it, the way in which this is "not a full solution" is pretty clearly that a Lightning node wants to (or perhaps, "has to" is better!?) advertise capacity, and this doesn't do this, except in the most primitive way: you could advertise "amount within a range" by choosing utxos from a range of values.

Filling in some more technical details:
* The proof size: it varies but not by much, in the 3-4kB range for keyset sizes oom $10^4$ to $10^6$. Varying the tree topology (branching factors 256 to 1024, depth from 2 to 6), doesn't seem to change it much; important to remember that we always end up with 2 bulletproofs because multiple proofs for one curve (secp or secq) can be aggregated.
* Points must be pre-processed before being added as leaves to the tree. This process can actually be quite computationally intensive. The reason for this processing is to allow the $x$-coordinate to curve point transformation to be unambiguous (the well known "$y$-coordinate tiebreaker" issue), but using an algorithm that is actually efficient inside the arithmetic circuit . Since this only has to be done once for each new key, this time cost can be ameliorated in every case but a pure trustless bootstrap to do a proof from scratch. Discussed [here](https://github.com/AdamISZ/aut-ct/issues/10)
* Verifications can be batched for bigger speedup (I believe also proofs, but I'm not sure exactly how that works); these are bulletproofs arithmetic circuit ZKPs, so the properties follow from that.
* Curve Trees being based on bulletproof ZKPs means we do not require pairings nor any assumption beyond ECDL (not that one would dismiss a proposal which required other assumptions out of hand, but rather the issue is that with secp256k1 as our base curve we *really* don't want to deal with pairings *** )

One thing that you might (should) wonder is, are there other techniques that can also provide near-constant fast verify for this Bitcoin use case? The most powerful techniques, e.g. Groth16 and its descendants, all require pairings; see note above. There is another idea, see Microsoft's SPARTAN as applied [here](https://github.com/personaelabs/spartan-ecdsa) which also does not require pairings nor any assumption beyond ECDL. I briefly investigated SPARTAN but I don't *think* it's quite as powerful in getting super-fast verification. I haven't investigated STARKs in this context, though, perhaps there is a direction there.

Extensions of this idea? From what I understand, we may be able to combine this Curve Trees construction with the idea of credentials, like KVAC as used in e.g. Wabisabi. If we could do this in a way that allowed a person to *independently* construct a single combined credential "this is 1.7 btc worth" such that (a) reuse of value is not possible, and (b) this could be done without a central coordinating server, then .. would we be getting very close to what might be needed for "good" Lightning gossip? I'm not very sure about anything I wrote in this paragraph :)


** While I am sort-of dismissing that earlier RIDDLE idea, it's only the mathematical construction I'm dismissing as too slow - the ideas there about tokens from (u)txo ownership I think are still interesting. [riddle blog](https://reyify.com/blog/riddle)

*** It's worth expanding on that ... in a *closed* system of peers you can obviously just have everyone sign a key of any exotic type you like, certainly including one for use in a pairings based system. But this drastically reduces the "spontaneity" of formation of anonymity sets, e.g. you can't include taproot utxos that aren't connected to running Lightning nodes etc. Tradeoff might be that you get *very* compact proofs etc. I tended to think that 3kB is already compact enough, since by design, there won't be millions of these proofs!

-------------------------

1440000bytes | 2024-05-13 12:04:04 UTC | #2

Thanks for writing this post.

This will make joinstr pools sybil resistant and we [discussed the idea](https://t.me/Joinstr/521) a few days back. It would work exactly like the hodlboard proof of concept where some peers in a round will be required to prove ownership of a UTXO greater than some amount of sats. This requirement can either be added by pool creator or nostr relay used for the pool.

If some nostr relays start adding this requirement apart from one time payment, it can make them DoS and sybil resistant for any use case.

I hope autct can also be used in [pathcoin](https://github.com/AdamISZ/pathcoin-poc) instead of fidelity bonds which simplifies the idea of self custodial eCash if covenants ever get implemented on bitcoin.

-------------------------

