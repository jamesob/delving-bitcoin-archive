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

AdamISZ | 2024-05-14 13:13:42 UTC | #3

Yes it's one of the *potential* use cases, i.e. a drop-in replacement for fidelity bonds in a coinjoin protocol without a single central coordinator. However, I do remember back when I proposed [RIDDLE](https://reyify.com/blog/riddle) that it *might* not be as great for that particular use case. So, "delving" into that ;) :

First, let's be clear on the threat being addressed: Sybil attacks that allow deanonymisation of the coinjoin by being ~ all the other participants in the round. A technique that imposes a cost on taking part can never make this impossible, so our chosen threat is an attacker bounded in their costs, and our defence is contingent/limited (you could try to quantify it for a particular case, but it's going to be very handwavy). The costs imposed by a timelocked utxo can be calculated explicitly in terms of [time value of money](https://gist.github.com/chris-belcher/87ebbcbb639686057a389acb9ab3e25b), but the costs imposed by simple utxo ownership at the time of a *recent* snapshot are clearly much less; principally just a utxo creation cost (arguably ameliorated, even). This leads one to think along the lines I did in the above blog, that: 

> By using age and BTC value requirements, a cost - in the form of BTC network fees - is imposed that is almost zero for slow, occasional use but becomes (nonlinearly) much larger for anyone attempting to use the service very frequently (and due to block arrival rate, *no* amount of money may be enough to achieve very high Sybilling rates).

Notice I say *age* and value requirements; "before the snapshot" can enforce that, or, I think a better idea, one can set the filter as $Q = a^x v^y$ where $a$ is age and $v$ is sats value. This can really bump up the cost to *burst attack* a system.

But with a decentralized coinjoin protocol rather than e.g. a website or a nostr relay or maybe a Lightning node, our main concern isn't *burst attack* as much as very slow Sybilling. I'm going to guess that, if we want to replace fidelity bonds direct money locking with this style of utxo proof, we need to set $x$ in the above formula much higher, so that a proof of the necessary value will require quite an old utxo (thus enforcing a similar timelocking but in the past, not pushed into the future). This doesn't sound very practical (locking way before using the service?), and will also reduce the anon set (though I think that won't be as big of a deal, most likely).

Finally, as well as these concerns (which can be summed up as "you can use this for coinjoins but it might end up being a weak defence unless you make it too impractical"), there's one last one that bothers me:

Structurally this aut-ct could be simplified to:

    Prover -> Verifier: [ZKP] [keyimage]

And that keyimage can't change unless you change the utxo (pubkey) (taproot). This means that every user of the decentralized protocol sees the same one, and if it's a token that lets them use the system for some considerable time (as it must be, as fidelity bonds are today), it by its nature allows linkage of different coinjoins.

The problem here is very deep: we want purely private participation in a protocol, *and* we want defence against Sybilling behaviour.

Probably the answer lies in token multi-issuance, which can be done [as described in this Issue](https://github.com/AdamISZ/aut-ct/issues/8) : 

Basically, when you do the DLEQ proof, you need an independent NUMS basepoint $J$ (and the "key image" is your private key * $J$), and you create that $J$ using a hash-to-curve operation acting on a string. Currently that string is "J,application-domain-label" (e.g. "J,hodlboardmainnet") but we could instead imagine it being "J,application-domain-label,1" and then 2,3 etc. This way you get multiple independent tokens per utxo. This is basically exactly what we do in PoDLE (which is *not* the same as fidelity bonds). I think this is probably the right way to go for a fully symmetric decentralized coinjoin protocol.

-------------------------

1440000bytes | 2024-05-14 17:39:45 UTC | #4

[quote="AdamISZ, post:3, topic:862"]
The costs imposed by a timelocked utxo can be calculated explicitly in terms of [time value of money](https://gist.github.com/chris-belcher/87ebbcbb639686057a389acb9ab3e25b), but the costs imposed by simple utxo ownership at the time of a *recent* snapshot are clearly much less; principally just a utxo creation cost (arguably ameliorated, even).
[/quote]

Is it possible that users can prove ownership of one of the timelocked UTXO?

-------------------------

AdamISZ | 2024-05-15 01:25:28 UTC | #5

Yep that's another possibility but it's a much more limited anon set and it's kind of a different model to the usual taproot one; with taproot the idea is that a utxo has (better, might have) a locking script that you don't see on chain.

That's in the category like "anon set of all the users of *this* protocol"; for that you might want to do things very differently. E.g. "all lightning network nodes that are choosing to engage in this protocol publish a key (somewhere, somehow) and a locking script for their utxo and then all the other members can accept a request to participate that attaches a ring sig or ZKP that proves that they own one of the keys+timelocked scripts that "registered".

The aut-ct thing is geared around the other extreme of "anon set of all taproot utxo pubkeys that satisfy a public filter" so much larger anon set but much more limited on the conditionality. Hope that makes sense.

-------------------------

kayabaNerve | 2024-05-22 09:44:21 UTC | #6

[quote="AdamISZ, post:1, topic:862"]
all the ring signature based stuff fails on sublinear verification
[/quote]

Correct. You need a succinct proof (which would then have linear proving) or to move the problem statement to one with sublinear complexity (a merkle tree).

[quote="AdamISZ, post:1, topic:862"]
But the [Curve Trees ](https://eprint.iacr.org/2022/756.pdf) approach solves this
[/quote]

Curve Trees is one approach, and great for a trustless setup discussion.

[quote="AdamISZ, post:1, topic:862"]
keyset sizes from 50K up to 2.5M and seen verification time mostly sitting between 40 and 70ms
[/quote]

You can do quite a bit better. My work has been applying Curve Trees to Monero and we're at 35ms for verifying one proof (using two curves without tailored field implementations yet crypto-bigint's Residue type) of 219b. With batch verification (n=10), it quickly gets down to 11ms.

[quote="AdamISZ, post:1, topic:862"]
The reason for this processing is to allow the xxx-coordinate to curve point transformation to be unambiguous (the well known “yyy-coordinate tiebreaker” issue), but using an algorithm that is actually efficient inside the arithmetic circuit . Since this only has to be done once for each new key, this time cost can be ameliorated in every case but a pure trustless bootstrap to do a proof from scratch. Discussed [here](https://github.com/AdamISZ/aut-ct/issues/10)
[/quote]

This isn't necessary if you define the linking tags as x coordinates. Then proving for a negative leaf may produce a negative linking tag, except the sign data is lost when you drop the tag's y coordinate.

You could still trivially produce a collision on the layers by hashing negative words. That's solved by using an initialization generator as a term with a constant coefficient of 1. Since that term can't be negated, you'd need to solve the DLP for the initialization generator and other generators.

[quote="AdamISZ, post:1, topic:862"]
Curve Trees being based on bulletproof ZKPs
[/quote]

Technically Generalized Bulletproofs (not Bulletproofs as published several years ago).

[quote="AdamISZ, post:1, topic:862"]
There is another idea, see Microsoft’s SPARTAN as applied [here](https://github.com/personaelabs/spartan-ecdsa) which also does not require pairings nor any assumption beyond ECDL.
[/quote]

The benefit of using Generalized Bulletproofs is the 'native' operations re: Pedersen Vector Commitments. Using Spartan would require manually building them on the towering curve (historically hundreds of multiplication constraints per word).

[quote="1440000bytes, post:4, topic:862"]
Is it possible that users can prove ownership
[/quote]

They'd just need to open the re-randomized output key. The existing DLEq proof does so (while additionally providing linkability).

-------------------------

kayabaNerve | 2024-05-22 03:32:17 UTC | #7

https://github.com/kayabaNerve/fcmp-plus-plus/tree/develop/crypto/fcmps for my own works for reference. If you redid the first layer (our first layer handles the key, a key image generator, and an amount commitment), it'd be applicable here.

Speaking of the key image generator, the above falls victim tor related-key attacks assuming the generator J is constant. This means people who create two outputs under a stealth address protocol (presumably such as Silent Payments) can then detect if those outputs are used in a protocol which requires publishing a linking tag. That's why Monero uses a per-output key image generator (which the linked work handles).

-------------------------

AdamISZ | 2024-05-22 12:09:33 UTC | #8

@kayabaNerve thanks for the feedback! I was only made aware of your work around the same time I wrote this post, but of course it's extremely useful to know that others are working on this same type of thing.

[quote="kayabaNerve, post:6, topic:862"]
You can do quite a bit better. My work has been applying Curve Trees to Monero and we’re at 35ms for verifying one proof (using two curves without tailored field implementations yet crypto-bigint’s Residue type) of 219b. With batch verification (n=10), it quickly gets down to 11ms.
[/quote]

Indeed, your numbers match up more closely with the benchmarking results quoted in the paper. I was trying to give indicative numbers that I actually achieved with my (crappy) code; the difference isn't too big anyway. And for batch verify, yes, I did note that in the OP, but thanks, it's useful to see what people have actually achieved.

Are your proving times also similar to what's claimed in the paper?

[quote]
one proof (using two curves without tailored field implementations yet crypto-bigint’s Residue type) of 219b. 
[/quote]

But on this part specifically, it's very interesting and very distinct from the CurveTrees paper - are you saying your proof size is 219 bytes? I believe
you're using Liam Eagan's work, and I also know you're working with different base structures because you don't have a 2-cycle with monero's curve (hence "towering" etc.), but in brief, would you say that a proof that much smaller than what is quoted in the orig. paper (like 2-3kB) is feasible here? That would be super useful in some use cases I think.

[quote]
The benefit of using Generalized Bulletproofs is the ‘native’ operations re: Pedersen Vector Commitments. Using Spartan would require manually building them on the towering curve (historically hundreds of multiplication constraints per word).
[/quote]

Probably I just misunderstood something, but I don't get what you're saying here; SPARTAN doesn't use a cycle of curves does it? Why are you referring to towering curve here? (my ignorance is duplicate here: I neither understand more about "towering curves" as you refer to in fcmp++ than "it's something you need because ed25519 doesn't have a 2-cycle", nor do I understand much at all about SPARTAN except that it uses sum-check protocols).

[quote="kayabaNerve, post:6, topic:862"]
This isn’t necessary if you define the linking tags as x coordinates. Then proving for a negative leaf may produce a negative linking tag, except the sign data is lost when you drop the tag’s y coordinate.
[/quote]

Oh, this is a tricky point but I think I *might* understand .. so, the algorithm for the ZKP needs each curve point to be represented as x-only for efficiency, but the process to go from point to x-coord needs a tiebreaker mechanism that can be calculated efficiently *in* the arithmetic circuit, and their "permissible points from a universal hash" does this. But is your point here that, for the *leaves* of the tree specifically, you could just not do that, as long as the linking tag is defined as x-coord only (i.e. ignoring sign)? I believe the algos. in their benchmarking code always required permissible points as inputs but that may just be a trivial detail that can be changed (hmm indeed come to think of it, I see no reason why the leaf points need this; it's only needed for the transition from one curve to the next...).

If so, that is a nice practical win to ditch that preprocessing.

-------------------------

kayabaNerve | 2024-05-22 17:51:39 UTC | #9

[quote="AdamISZ, post:8, topic:862"]
Indeed, your numbers match up more closely with the benchmarking results quoted in the paper.
[/quote]

It should do significantly better with proper arithmetic. It's a fraction of the size.

[quote="AdamISZ, post:8, topic:862"]
And for batch verify, yes, I did note that in the OP,
[/quote]

Just wanted to provide the actual number :) 

[quote="AdamISZ, post:8, topic:862"]
Are your proving times also similar to what’s claimed in the paper?
[/quote]

I haven't looked. They haven't been an issue and it isn't a priority of mine.

[quote="AdamISZ, post:8, topic:862"]
proof size is 219 bytes
[/quote]

Oh, no, 219 billion for the set size.

[quote="AdamISZ, post:8, topic:862"]
SPARTAN doesn’t use a cycle of curves does it?
[/quote]

Neither does Bulletproofs. The premise of Curve Trees is two proofs, one on each curve, each towering a child curve, as you've noted. Curve Trees with Spartan would require doing the Pedersen hash on the towering curve and lose a lot of the performance (though it'd still beat out Poseidon by a significant margin if I recall my estimates correctly). To be clear, secp256k1 towers secq256k1, secq256k1 towers secp256k1. They accordingly form a cycle, yet if secq256k1 didn't tower secp256k1, secp256k1 would still tower secq256k1.

[quote="AdamISZ, post:8, topic:862"]
as long as the linking tag is defined as x-coord only
[/quote]

Correct re: leafs. Re: branches, see initialization generator commentary.

-------------------------

