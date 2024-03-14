# Unilateral Exit

ZmnSCPxj | 2024-03-11 09:07:34 UTC | #1

Introduction
========

In order to implement shared **non**-custodial UTXOs, individual users need the capability to perform a unilateral exit.  A unilateral exit is the ability of a single participant to leave the shared UTXO without permission from any other participant. Without a unilateral exit capability, the other participants can hold the funds of a single participant hostage, thereby effectively having custody of their funds, and failing to be non-custodial.

We can observe that existing custodial systems already implicitly share a small number of UTXOs amongst their captive clients.  The improvement sought is to remove the custodial part while retaining the sharing part.

For simple shared non-custodial 2-party UTXOs like Lightning Network channels, unilateral exit simply requires publishing the shares of both parties.

However, we should also notice that even in Lightning, often the channel will contain other contracts than simply "the fund of Alice" and "the fund of Bob" --- there will often be HTLCs offerred by one participant to another.  In the case of a unilateral exit, all contracts --- N of them --- need to be published onchain.

This is problematic as sometimes you may want to unilaterally exit only one of the promised offchain transaction outputs.  For instance, if only one of the HTLCs is about to expire, it would be best if we could publish only *that* specific HTLC, and forgo publishing the rest, in the hope that the counterparty can come back online and settle the rest of the HTLCs offchain and continue transacting.

For HTLCs, the timelock can be used as a strict ordering, such that HTLCs with earlier timelocks can be made easier to exit while retaining the rest of the funds as latently capable of update.  This allows for effectively O(1) publication of the nearest-timeout HTLC.  However, for the general case where *any* participant may unilaterally exit *any* of the offchain UTXOs it has, the best seems to be the use of Merkle trees, which require O(log N) data to be published onchain to exit a single offchain UTXO.

As a concrete example, `OP_CTV`-trees can be created to allow unilateral exit of some offchain UTXO by publishing the tree path from the root to the specific offchain UTXO being published.

Of note is that an entire Merkle tree with N leaves has a total data size O(N log N).  If the probability that ***all*** N offchain UTXOs will end up having to exit is high, then producing the entire tree requires O(N log N) data, whereas a simple single-input multi-output transaction would be O(N), which is one of the objections to `OP_CTV`-trees and mechanisms based on it.  That is, an `OP_CTV` tree would take up more blockspace ***if*** all the offchain transaction outputs ***will*** be published anyway.

Obviously the hope is that ***not*** all offchain transaction outputs will be published ever even if some subset does want to unilaterally exit, and instead the remaining funds can be used in further offchain interactions to functionally "cut through" and reduce blockspace utilization total, i.e. a tradeoff between the theoretical danger of all participants exiting, vs. the theoretical advantage of most participants remaining in the shared UTXO and transacting offchain.  However, when using `OP_CTV` trees, the problem is that a single unilateral exit creates O(log N) new transaction outputs that contain the remaining funds, and re-unifying those funds to a single output requires even more data to be published onchain.

So, I present two problems:

* Can we create a commitment scheme that commits to a set, where we can show that an item is a member of that set in O(1) space instead of O(log N)?
  * By solving this problem, we can create a unilateral exit scheme that takes O(N) space if **all** outputs exit.
* Can we create an alternative to `OP_CTV`-trees which does not force creation of O(log N) actual UTXOs?

Accumulators
========

A cryptographic accumulator is a kind of cryptographic commitment scheme, such that:

* The accumulator can commit to a set of items.
* It is possible to insert and delete items to the set:
  * Insertion of items creates a "witness" as well as a new version of the accumulator.  The "witness" is ""succinct", i.e. "short" for some definition of "short".
  * Removal of items requires the "witness" constructed from insertion, and creates a new version of the accumulator.
  * In most accumulators, insertion and removal will require that existing witnesses already issued be modified as well, but the revelation of the witness of the insertion / removal is sufficient to modify the existing witnesses.
* It is possible for a third party to get an accumulator and a witness, and be convinced that the item was inserted and not yet removed from that version of the accumulator (i.e. proof of set membership).

The concept of "cryptographic accumulator" is an abstraction and is not a specific thing.  In cryptography, any number of schemes may match it (with various caveats like "with high probability" etc.).

Generally, Merkle trees are considered a form of cryptographic accumulator.  To insert into the Merkle tree, you generate a new path to a new leaf, and the witness is the Merkle tree path to the new leaf (which is sufficient to prove that the leaf exists in the tree).  Existing witnesses need to be modified as well, but the Merkle tree path to the new leaf is sufficient to modify the Merkle tree paths of the other leaves.   Removal means taking the Merkle tree path and removing the leaf, and recomputing the Merkle tree root hash (and again, modifying the Merkle tree paths of other non-removed leaves).  The issue with Merkle trees is that the witness (Merkle tree path) is "short" in the sense of requiring only O(log N), but it is not "short" in the sense of requiring only O(1).

RSA-based accumulators also exist.  The RSA assumption is that: it is easy to verify if a number is divislble by a partricular prime number, but difficult to actually factor a number to all its prime number factors.  A simple RSA accumulator would have items be (or be represented by) prime numbers, and the accumulator itself would simply be the product of all the prime number members.  To prove membership, show the item (a prime number) and try to divide the accumulator (total product) by it and see if it is divisible.  Insertion is multiplication and removal is division.  This has O(1) witness, but the accumulator itself is O(N) --- the representation of a product generally requires a number of bits equal to the sum of the bits representing its factors.  Thus, if we are using Bitcoin and fixed-size `scriptPubKey` we would feel obligated to hash the accumulator to a fixed-size hash, and revealing the accumulator would be O(N) data, meaning in effect that the witness is O(N).  There are RSA-based schemes which have the accumulator itself be O(1), which I am studying right now, but those frikkin mathists like 1-character variable names too much, often with characters being in a different font being different things, which is crazy, like this is the first thing programmers get taught, do not use single-character variable names, what is wrong with mathists?????

Another observation we can make is that Taproot script alternatives are really a use of Merkle tree accumulators.  ***If*** a nice accumulator with O(1) accumulator and O(1) witness actually existed, whose constant factors and validation times were acceptable, *then* we should have modified Taproot to use that accumulator instead of Merkle tree accumulators.  Since we did not, and Taproot uses Merkle trees, I hereby conclude that there currently exists **no** accumulator with O(1) accumulator size and O(1) witness with acceptable constant factors and validation times.

`OP_EXIT`
========

Let us assume that we have some kind of accumulator scheme. This can be any accumulator scheme with the desired properties of O(1) accumulator size, O(1) witness size, and nice constant factors and validation times.  Or it could just be Merkle trees if we cannot find anything better.

Now, suppose there are N different participants sharing a single UTXO.  These N participants have N funds inside the UTXO, i.e. they are offchain UTXOs that can be made real on unilateral exit, or kept in the shared UTXO where some other mechanism (Decker-Wattenhofer decrementing `nSequence`, or timebound Spilman sets a la timeout-trees, Decker-Russell-Osuntokun) can be used to cut through transfers between participants within the same shared UTXO.

We start with some "empty" accumulator, then insert a commitment to pairs `(point, value)`, where `point` is a standard SECP256K1 denoting the (possibly MuSig or FROST-composed) owner of a particular number of satoshis, with `value` being that number of satoshis.  If necessary we can add some `salt` as well, depending on the needs of the accumulator.  We insert a commitment for each owned value.  Ideally, we would want an accumulator where the insertion operation can be done by any party, not just the particular participant with an interest in a particular `(point, value)`.

We then use the accumulator itself (or a commitment to the accumulator) as the internal public key of a Taproot output. This allows us to add any number of additional Tapscript alternatives, but does prevent the use of the keyspend path.  The idea is that the output itself would be inside an offchain mechanism like Decker-Russell-Osuntokun, and thus would only appear during unilateral cases, where a keyspend path would be unlikely in the first place.

We have a Tapscript which contains an `OP_EXIT` opcode.  This opcode takes the accumulator from the internal public key (or validates an on-stack accumulator versus the commitment in the internal public key) and then allows for an exit of one of the `(point, value)` commitments.  The stack must provide the witness to the `(point, value)` being committed, as well as a signature from `point`.

`OP_EXIT` performs the following validation:

* The given witness does actually assert that the given `(point, value)` is committed in the accumulator.
* It removes the `(point, value)` item from the accumulator.  Then:
  * If the resulting accumulator is non-empty:
    * Recalculate a new Taproot address, with the accumulator minus the `(point, value)` being exited as the internal public key, and the same script path/taptweak.
    * Validate that the corresponding output (the output at the same index as this input) has the new Taproot address with updated accumulator, and the amount is the same as the input amount minus the `value`.
* The given signature signs the transaction with the public key `point`, *but* with this input removed, and if the resulting accumulator was non-empty, with the corresponding output removed.

The above validation allows a participant to exit with its `point` key signing for the exit, and paying for onchain fees from the `value` in its committed `(point, value)`.

A more elaborate form would be that the `point` itself could be a Taproot key, and we would also have a repeated, nested form of Taproot with keyspend or tapscript path alternatives.

When one participant exits, it has to publish the witness that attests the existence of its `(point, value)` in the set.  Then another participant which *also* wishes to exit can modify its witness, if the accumulator requires the witness to be modified on removal of an existing entry (e.g. Merkle tree accumulator), with the blockchain as a common log of such witnesses.

`OP_TLUV` Implements `OP_EXIT`
-------------

The `OP_TLUV` by ajtowns fundamentally implements much of `OP_EXIT`, as has been noted in its original writeup.

Indeed, the observation that the Taproot script alternatives are simply a use of an accumulator feeds into `OP_TLUV` being an implementation of `OP_EXIT`.

-------------------------

ursuscamp | 2024-03-11 11:18:29 UTC | #2

An idle thought: I wonder if sacrificing flexibility by standardizing the accumulator value for all participants could allow for a more efficient accumulator construction?

Instead of someone having 1 accumulator UTXO with 1.5 BTC, she might have 1 BTC in an accumulator where everyone has 1 BTC and 0.5 in a second, similar accumulator.

-------------------------

ZmnSCPxj | 2024-03-11 13:06:35 UTC | #3

You need to somehow identify the user, and the most general way of pseudonymously identifying users is for the user to give a public key, which is a ***lot*** of entropy you need to pack into the accumulator. i.e. the amount is a tiny amount of entropy compared to the identifier.

-------------------------

ursuscamp | 2024-03-11 14:57:53 UTC | #4

I think you are saying what I meant!

If everyone in the accumulator had the same amount, then you only need to accumulate the public key and could make any potential proof scheme simpler or more efficient.

edit: Or not! Either way, a cool idea. Thanks for showing.

-------------------------

ProofOfKeags | 2024-03-11 22:02:04 UTC | #5

Depending on how much pre-computation you want to do. You can embed all the different possibilities of which participant unilaterally exits as sibling tapleaves where the transaction creates utxos, one where the individual's share is ejected entirely, and another where the remaining participants share the utxo:

Let's say you have a set of N parties, P, indexed from [0, N). You would construct N presigned transactions, with two outputs each, one that pays the ejected party their share, the other pays the remaining set their combined share. From there you have a recursively smaller problem, and you can do this all the way to 0. At first this looks like it has factorial complexity since every sequence of unilateral exits would create a different sequence of shared utxo outpoints. With APO or CTV you can cut it down to $ \sum_{i=2}^{N} nCi  = O(N^3) $ in offchain computation by sharing common scriptPubKeys since ejections commute. On chain, you get $ O(N) $ I think.

Regarding accumulators, it seems like the key requirement is that the consensus rules are capable of either computing or verifying the removal operation. Having an operation that could do this is probably a really useful general opcode.

> but those frikkin mathists like 1-character variable names too much, often with characters being in a different font being different things, which is crazy, like this is the first thing programmers get taught, do not use single-character variable names, what is wrong with mathists???

It can be useful to do this because sometimes the shape is more important than the name when trying to grapple with the idea. By using single letter variable names it forces you to contend with the structure of the idea directly. Sometimes any name you give it necessarily forces you to look at it through an overly reductive lens. They could also just be dicks and be doing it unnecessarily, though 🤷🏻‍♂️

-------------------------

ZmnSCPxj | 2024-03-11 22:05:05 UTC | #6

Any putative commitment scheme that can only commit to `point`s for efficiency, can be modified to commit to `(point, value)` by the standard pay-to-contract scheme where `committed_point = point + hash(point || value) * G`, i.e. the same scheme used in Taproot, which commits to a point onchain.

-------------------------

stevenroose | 2024-03-11 22:42:46 UTC | #7

Using MAST for this scheme is pretty broken though. It requires all possible exit orders to be pre-calculated which is factorial. It doesn't work with more than maybe 10 or with specialized hardware 15 participants, if I remember my math from some months ago correctly.

When trying to do large sizes, like for Ark, we need an actual accumulator so that the calculation of the remainder is inside the opcode, not pre-calculated. I've had discussions with Salvatore about such accumulators.

Some kind of append-only merke forest would work in a fraud-proof (interactive) setting. In such a scheme, every exit would take the leaf index as input and the resulting new accumulator would be the same tree with an "exit leaf" appended. If it is malicious, it can be disputed by proving either (1) the exit is not located at the given leaf index, (2) the exit was already performed and there is an exit leaf for it or (3) the resulting new accumulator is not equal the old one plus the exit leaf.

Something like this would work and can be implemented using MATT or CATT.

-------------------------

ZmnSCPxj | 2024-03-11 23:20:34 UTC | #8

[quote="ProofOfKeags, post:5, topic:664"]
It can be useful to do this because sometimes the shape is more important than the name when trying to grapple with the idea. By using single letter variable names it forces you to contend with the structure of the idea directly.
[/quote]

which is my problem precisely, it **looks like** some existing idea I know of, but then the mathist turns around and says "you were expecting this existing idea, but it was me, Dio!!!!"

-------------------------

ZmnSCPxj | 2024-03-12 23:42:13 UTC | #9

[quote="ZmnSCPxj, post:1, topic:664"]
Of note is that an entire Merkle tree with N leaves has a total data size O(N log N).
[/quote]

***OOOPS THIS IS WRONG***.

Turns out that any tree structure with N leaves is total data size O(N), not O(N log N).  However, there is a constant multiplier on N relative to an array of N entries.

For a binary tree, the multiplier is 2.0.  For a tree where nodes have 4 children, the multiplier is 4/3, which honestly is pretty dang close to 1 (this is the same as mipmaps for 2d textures, they *only* add 33.33% more vram use; every 4 pixels in the original texture is 1 pixel in the next-smaller mipmap, which is equivalent to grouping 4 leaves in a single parent node, have I ever mentioned that I am an indie game dev wannabe who got suckered into buying Bitcoin).

So yes a `CTV` tree is more expensive in data size due to the above constant multiplier, plus the necessary overhead of using P2SH/P2WSH/P2TR, but is O(N) on the number of leaves, and increasing the fanout gets it closer to the case of "publish all the outputs".  Hmmm.

-------------------------

ZmnSCPxj | 2024-03-14 06:07:31 UTC | #10

Looking into this more --- it seems that practical accumulators with O(1) witness sizes tend to require trapdoors.  In other words, you have some party that knows of the trapdoor and is able to forge commitments.  I am unaware of any cryptographic accumulator with O(1) witness size that is trapdoorless, they seem to require O(log N) witness size (which in a all-N-unilaterally-exit case leads to O(N log N) exit).

In particular, as noted, it turns out trees, including `CTV`-trees, have O(N) total data, and an all-N-unilaterally-exit using `CTV` trees is actually O(N), because mipmaps are awesome.

Note that just because there is a trapdoor does ***not*** mean that the party-with-trapdoor must absolutely be some ***single*** trusted party.  With the magic of multiparty computation it may be possible to create a party-with-trapdoor that is actually the N-of-N of all participants, such that no one participant actually knows the trapdoor.  See `OP_EVICT` for an example.

(ROLL-YOUR-OWN-CRYPTO-WARNING) For the case of unilateral exit rather than eviction as with `OP_EVICT`, the trapdoor may be a simple sum of N pubkeys from all participants (as in `OP_EVICT`) --- you need to protect against key cancellation attacks as well, such as requiring proof of knowledge of private key of each pubkey share.  Suppose there are public keys `point[0]` to `point[n-1]`, the trapdoor being the corresponding sum of private keys (shards of which are known by the participants, but no one participant knows the entire trapdoor).  To commit to a particular `(point[i], value[i])`, then the signers `point[0]..point[i-1]` and `point[i+1]..point[n-1]` --- i.e. all the signers ***except*** `point[i]` --- construct a MuSig2 signature committing to `(point[i], value[i])`.  More specifically, the owner of `(point[i], value[i])` gets MuSig2 partial signatures and stores each partial signature with the corresponding `point[all except i]` from that participant.  Then the accumulator is the sum of `point[0]` to `point[n-1]`.

To delete from the accumulator, you show `(point[i], value[i])` you commit to, then subtract `point[i]` from the accumulator amount.  The owner of `(point[i], value[i])` then sums up all the partial signatures to generate a full MuSig2 signature, which it presents as witness that `(point[i], value[i])` was committed in the accumulator.  The next accumulator is then the current accumulator minus `point[i]`.  The other owners of `(point[j[, value[j])` where `i != j` then remove the partial signature from `point[i]` from their storage, so that when they exit later, their own witness sums up to the correct value.(we would need a variant of Schnorr signing which does not commit to the signing public key --- ROLL-YOUR-OWN-CRYPTO-WARNING).  Note that the last exiter cannot present a witness though --- maybe every such construct could use a publicized shared private key and amount to be used as the "last exiter" (so that the last *real* exiter is technically always the second-to-the-last exiter), which miners can then MEVil to get as fee.

But basically yes, the drawback of trapdoored O(1) accumulators can possibly be worked around with multiparty computation.  ***The problem is that the setup of a trapdoored-but-non-custodial/trust-minimized requires everyone to be online simultaneously to generate an N-of-N "trusted" party which is really the consensus of every participant***.  The advantage of `CTV`-trees and non-trapdoored accumulators is that they can be set up without requiring trust and thus without requiring building a consensus N-of-N.

-------------------------

