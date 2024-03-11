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

