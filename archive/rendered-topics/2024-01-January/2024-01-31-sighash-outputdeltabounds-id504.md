# `sighash_outputdeltabounds`

ZmnSCPxj | 2024-01-31 06:52:48 UTC | #1

Here is a proposal for a new `SIGHASH` scheme.  The intent of this new scheme is to allow Poon-Dryja to defer feerate decisions later while paying from the in-channel funds (instead of requiring an extra UTXO and an artificial change output ("anchor")), as well as to allow Decker-Russell-Osuntokun update transactions to be paid from in-channel funds (normally the requirement that the input amount == output amount prevents update transactions from paying their own fee and requiring an additional input to pay fee) in addition to letting feerate decisions be deferred later, as well.

In a new Tapleaf version, the signature operand given to `OP_CHECKSIG`/`OP_CHECKSIG_VERIFY`/`OP_CHECKSIGADD` can be 84 bytes:

* 1 byte: `SIGHASH` flags
* 3 bytes: unsigned integer indicating a `vout_index`
* 8 bytes: signed integer indicating a difference, the `delta_min`
* 8 bytes: signed integer indicating a difference, the `delta_max`
* 64 bytes: Schnorr signature as defined by Taproot

If so, the operation `OP_CHECKSIG`(`ADD`,`VERIFY), in addition to performing signature validation, also performs the following:

* Get the amount of the input this is spending as `input_amount`, a signed integer.
* Calculate: `min = input_amount - delta_min` (`delta_min` is signed so this can become an addition, if this would cause overflow, fail; the result can be negative and negative `min` is still valid)
* Calculate: `max = input_amount - delta_max` (ditto)
* Check `min <= max`, if not, fail
* Get the amount of the output that is at `vout_index` as `output_amount`, a signed integer.
* Check `min <= output_amount <= max`, if not, fail.

If the above validation succeeds, then the sighash function is also modified as follows:

* Instead of a 1-byte `SIGHASH`, use the first 20 bytes (ie. `SIGHASH|vout_index|delta_min|delta_max`)
* When getting the amount for this input, treat it as being 0 amount.
* When getting the amount for the output at `vout_index`, treat it as being 0 amount.
* Otherwise same as Taproot sighash.

For example, suppose you have a tx with an input `txid1:vout1` with amount `100` and two outputs `(A, 42)` and `(B, 15)`, and you do a `SIGHASH_ALL` in the sighash byte while having a `SIGHASH_OUTPUTDELTABOUNDS` (i.e. using the above 84-byte blob instead of a 64-byte or 65-byte blob) with `vout=1`, `delta_min=90`,`delta_max=42`. Then you would sign as if the transaction had an input `txid1:vout1` with amount `0`, and outputs `(A, 42)` and `(B, 0)`.  The `OP_CHECKSIG` would also validate that the output `(B,15)` has an amount between `100 - 90 <= output <= 100 - 42`, which in this case would pass.

The result of this is that a signature that signs for `SIGHASH_OUTPUTDELTABOUNDS` will impose bounds on a particular output, so that it is within some number of satoshis of the input being signed for.

Poon-Dryja
=========

Let me propose an alternate Poon-Dryja-based protocol.  When exchanging signatures for commitment transactions, we send a signatures with `SIGHASH_ALL|SIGHASH_OUTPUTDELTABOUNDS` for the commitment transaction of the peer. As a concrete example, suppose the commitment transaction has the following state:

* channel funding input = 100
* output to A = 42
* output to B = 58
* No HTLCs

As is typical, the channel funding output is a 2-of-2 between A and B.  In our case, it would have a tapleaf with version supporting `SIGHASH_OUTPUTDELTABOUNDS`, whose script is a trivial separate-signature 2-of-2 `<A> OP_CHECKSIGVERIFY <B> OP_CHECKSIG`.

Suppose A is giving the signature for the commitment transaction held by B.  In that case, it creates a `SIGHASH_ALL|SIGHASH_OUTPUTDELTABOUNDS` with `vout_index=1` (the index of output to B), `delta_max=42` (the channel funding input minus the amount to B), and `delta_min=100` (the channel funding input).  This is a signature that commits spending the funding outpoint to the output set `(A, 42)` and `(B, 0)`, as it is `SIGHASH_ALL`.

Now suppose after A gives that signature, A never comes online again.  In the meantime, cute dog webp images become popular on the blockchain, so that in order for B to get the commitment transaction confirmed, it needs to pay 43 units.  In that case, B constructs a commitment transaction that has outputs `(A,42)`, `(B, 15)` --- total output = 57 and the channel funding input is 100, so the fee is 43.  B signs this version of the commitment with `SIGHASH_ALL`, and provides the signature it got from A, which is a `SIGHASH_ALL|SIGHASH_OUTPUTDELTABOUNDS(vout=1,delta_max=42,delta_min=100)`.  This validates, as the `delta_min` and `delta_max` bounds mean that the B output can be anywhere from `(B, 0)` to `(B, 58)`, and A still signs off on its output being `(A, 42)`.

The net effect is that B is the one paying for the commitment transaction onchain fee using its in-channel funds.

Similarly, second-stage HTLC transactions can use the same technique.  Suppose there was an HTLC offerred by B to A.  When A is signing for the second-stage HTLC transaction (an "HTLC-timeout" tx in BOLT 3 parlance), A would sign `SIGHASH_ALL|SIGHASH_OUTPUTDELTABOUNDS` with `vout_index=0`,`delta_max=0`,`delta_min=<htlc_amount>`.  This allows B to confirm this second-stage HTLC by spending up to the entire amount of the HTLC (`delta_min=<htlc_amount>`) into fees, or having very little fees (`delta_max=0`) if the mempool is empty.

Decker-Russell-Osuntokun
==============

Now let us turn to Decker-Russell-Osuntokun.  A weakness of this scheme is that, because the update transaction MUST be spendable by another, later update transaction, the amount of the update transaction output MUST equal the amount of the channel funding outpoint.  This means that an update transaction CANNOT spend any in-channel funds to pay its own fees.  Instead, you are supposed to add on an extra input to pay for fees, and since it is very unlikely you have a coin that is exactly the fee you need, you also want to add on a change output for your input coin minus the fee you pay.  This means that most update transactions will be 2 inputs (1 = channel funding output OR previous update transaction output, 1 = your fee-paying input) and 2 outputs (1 = new update transaction output, 1 = change for your fee payment).

Now suppose we instead used `SIGHASH_NOINPUT|SIGHASH_OUTPUTDELTABOUNDS`.  In such a case, all participants in the Decker-Russell-Osuntokun would need to agree on some "maximum fee" that an update transaction would pay.  However, we can then make update transactions 1-input 1-output, deducting fees from the shared funds of the mechanism.

The reason why we have a "maximum fee" is that the output of the update transaction is still semantically a shared UTXO between all participants.  If one of the participants in the Decker-Russell-Osuntokun mechanism was secretly a miner, and there was no maximum fee, then the miner could set the output of the update transaction to 0 (or dust, whatever) and give all the amount to fee for itself.

This maximum fee is then imposed by the `delta_min`, such that if the agreed-upon maximum fee is `maximum_update_tx_fee`, `delta_min=maximum_update_tx_fee` and `delta_max=0`.

When participants are exchanging signatures for the update transactions, they sign with `SIGHASH_NOINPUT|SIGHASH_OUTPUTDELTABOUNDS(vout=0,delta_max=0,delta_min=maximum_update_tx_fee)`.

When one of the participants needs to publish an update transaction, they add their signature with `SIGHASH_ALL` committing to a specific spent input and output amount, and thus indirectly commits to a specific fee deducted.   They also then use the signatures they received from the other participants in the mechanism, which bound how much of the total channel funds can be spent in a single update transaction.

Since the amount of the update transaction is also variable now, the state transactions of the Decker-Russell-Osuntokun also need to be signed using `SIGHASH_OUTPUTDELTABOUNDS` in addition to `SIGHASH_NOINPUT`.  Further, each participants gets its own version of the state transaction.  In its own version, it can freely deduct its own output --- the `SIGHASH_OUTPUT_DELTABOUNDS` points to "their" output as its `vout` --- but cannot change the other outputs, which have fixed amounts committed to by `SIGHASH_NOINPUT`.  This is similar to the previous example of Poon-Dryja (where each participant has its own commitment tx), and generalizes to the state transaction of this variant of Decker-Russell-Osuntokun just as well (where instead of a common state transaction as in the original paper, we have each participant its own state transaction, with the rule that the signatures they hold allow them to only deduct from their own transaction output).  The net effect is that if a participant is the one that publishes its version of the state transaction, it is the one that pays for the fees of all the update transactions and the state transaction.

### Multiple Update Transactions Oh No!

Now of course an edge case with Decker-Russell-Osuntokun is that the participants can do update transaction tennis, where if the latest state is N, one participant first publishes N - 1000, then another participant publishes N - 999, then another publishes N - 998, then another publishes N - 997 etc. until one participant finally publishes update transaction N.  Of course, each such update transaction will need to pay fees.

If we are using `SIGHASH_OUTPUTDELTABOUNDS` then it means that each update transaction lowers the shared channel funding to pay for the fee, until none of the state transactions can be published because the final update transaction output amount is too small to pay for the state transaction.

This is however a case of "Doctor, it hurts when I ***DO THIS***" and the doctor says "***EWW DO NOT DO THAT BAD THING I NEVER EVEN DID THAT TO THE CADAVERS I PLAYED WITH IN MED SCHOOL***".  That is: do not play update transaction tennis, always just honestly publish the latest update transaction to stop the bleeding of the shared mechanism funds into fees for update transactions.

Of course, the problem is that *in practice*, what you *think* is the latest *might not* be the latest.

The classic example is of course the naive Lightning Network operator who restores from a backup that was made 2 seconds ago, when the latest update to the channel happened 1 second ago.  But if your peer restores from an old backup --- then just do not engage in update transaction tennis and publish the latest update transaction.

Another and perhaps more problematic example is a sophisticated Lightning Network operator who distributes their updates among multiple watchtowers.  Suppose that the node has signed off on a new state and handed over signatures, but has not gotten around to sending its received signatures for the latest state to the watchtowers.  Worse, our sophisticated Lightning Network operator heard that one of its watchtowers was actually operated by furries, who everyone on the Internet hates because furries, so they stop sending their updates to that watchtower. And then their counterparty, who is a naive Lightning Network operator, restores from a backup that was made 5 seconds ago.  And now the sophisticated Lightning Network operator, while rage-tweeting about furry-run watchtowers on the platform formerly known as Twitter and now known as the X Windows System, accidentally kicks the power supply of the Raspberry Pi (like why is the Raspberry Pi USB-C receptacle so weak anyway???).  This causes the watchtower run by furries to publish the last update transaction it received from 4 seconds ago (because that is how recently the sophisticated Lightning Network operator learned that they were run by furries).  However, the Real Watchtower run by Real Men And Women And Definitely Not Furries was given an update transaction from 3 seconds ago, but because everyone was using it now because it is not run by furries, it had a bit longer latency to respond to the new block and can only publish the 3-seconds-ago update transaction after the furry-run watchtower.  (Un)Fortunately, the sophisticated Lightning Network operator has managed to bring the Raspberry Pi back online, which now publishes the REAL latest update transaction from 2 seconds ago, because it was 2 seconds ago that its power supply was kicked and it was just encrypting the packet it was going to send to the Real Watchtower run by Real Men And Women And Definitely Not Furries.

On the other hand, we can argue that an accident like the above is unlikely.  As a mitigation, watchtowers of Decker-Russell-Osuntokun mechanisms using this `SIGHASH_OUTPUTDELTABOUNDS` can probably afford to wait a few blocks in case the "real" nodes come online to give the REAL latest update transaction, to help mitigate this comedy of errors that leads to an accidental update transaction tennis.

Generalizing
==========

Of course, why settle for special signatures?

Instead of "just" signatures, we could add new fields in the Taproot annex.  If so, we should probably TLV-encode the Taproot annex, so as to maximize the flexibility of the fields that the Taproot annex can contain in the future.  When can then define a special annex TLV type that contains the `vout_index|delta_min|delta_max` tuple.  If this annex TLV type exists, then the indicated output must be within `delta_max` to `delta_min` satoshis of the input the annex is on.

We can then make `SIGHASH_OUTPUTDELTABOUNDS` a "real" `SIGHASH` that you can bitwise-OR, e.g. `0x40`.  If this flag is set, then the Taproot annex MUST contain the `vout_index|delta_min|delta_max` TLV type, AND the `SIGHASH` computes as if the input has 0 amount and the output at `vout_index` has 0 amount (to allow the input and output to vary).  The annex itself is committed to under most `SIGHASH` variants. This would be equivalent in power to the scheme initially described above.  This scheme may be "more correct" than the initially-described scheme.

As an annex field, we can then add new opcodes that can introspect the `vout_index`, `delta_min` and `delta_max` for whatever covenanty goodness it might have.  This can be done in a later softfork.

### Intuition

A signature with `SIGHASH_OUTPUTDELTABOUNDS` effectively says "I allow this input to contribute fees within these bounds --- adjust the output at `vout_index` to pay for the fees".

If we go the annex way, then the annex says "this input contributes fees within these bounds --- adjust the output at `vout_index` to pay for the fees".

-------------------------

ZmnSCPxj | 2024-02-04 12:21:02 UTC | #2

Intuitively, we can consider that the entire point of this scheme is to specify that one output pays the fees, thus we do not commit to that particular output having a particular amount. Optionally, we can impose a maximum amount that the output can pay, in case the output is (still) a shared output.  Protocols using this require a separate signature, coming from an owner of the output, which allows that owner to fix the fee paid. This also requires only one participant to be online at that time (i.e. other participants need not be online), allowing that participant to arbitrarily sign unilaterally and decide how much fees to deduct from their owned output, while still imposing that the other outputs are untouched.

Just as `SIGHASH_NOINPUT` is implementable via `OP_CHECKTEMPLATEVERIFY` and `OP_CHECKSIGFROMSTACK`, we can also consider how to implement this scheme using `OP_CHECKTEMPLATEVERIFY`.

For example, suppose we extend `OP_CHECKTEMPLATEVERIFY` in the following manner:

* If the stack top is 32 bytes, follow BIP-119.
* If the stack top is 33 bytes or more:
  * Take the first byte as the `CTVHASH` type.
  * If the first byte is:
    * `0x00`, this is an "output pays fee" `CTVHASH` type.
    * anything else, `OP_CHECKTEMPLATEVERIFY` acts as an `OP_NOP` (this allows for future extensions).

When verifying an "output pays fee" `CTVHASH` type, we extract the following pieces of data:

* An output index.
* An optional amount to be deducted.
* A 32-byte hash commitment.

The length of the stack top determines how the data is interpreted.  The last 32 bytes is always the 32-byte hash commitment.  Then if the stack top length is:

* 34 bytes: byte 1 is the output index, there is no amount to deduct.
* 35 bytes: bytes 1-2 is the output index in little-endian, there is no amount to deduct.
* 36 bytes: bytes 1-3 is the output index in little-endian, there is no amount to deduct.
* 37 bytes: bytes 1-4 is the output index in little-endian, there is no amount to deduct.
* 38 bytes: byte 1 is the output index, bytes 2 to 5 is the amount to deduct in little-endian.
* 39 bytes: bytes 1-2 is the output index, bytes 3 to 6 is the amount to deduct in little-endian.
* 40 bytes: bytes 1-3 is the output index, bytes 4 to 7 is the amount to deduct in little-endian.
* 41 bytes: bytes 1-4 is the output index, bytes 5 to 8 is the amount to deduct in little-endian.
* 42 bytes: byte 1 is the output index, bytes 2 to 9 is the amount to deduct in little-endian.
* 43 bytes: bytes 1-2 is the output index in little-endian, bytes 3 to 10 is the amount to deduct in little-endian.
* 44 bytes: bytes 1-3 is the output index in little-endian, bytes 4 to 11 is the amount to deduct in little-endian.
* 45 bytes: bytes 1-4 is the output index in little-endian, bytes 5 to 12 is the amount to deduct in little-endian.
* otherwise: fail validation

> **Rationale** The above encoding does not require spending extra bytes as for `CVarSize`; instead, the push opcode that gets the above onto the stack effectively encodes how many bytes to use for the output index and (optional) amount to deduct.
>
> The expectation is that most transactions will be small and the output index will correspondingly be small as well. In contrast, amounts are expected to be large. Nevertheless, it is also expected that amounts involved will rarely exceed 42.94967295 BTC, thus the amount to deduct may also be encoded in 4 bytes instead of 8. The expectation is that every byte matters.

In addition, the "output pays fee" `CTVHASH` type requires the following validation:

* The script interpreter maintains a "heavy CTV hashes" variable that is initialized to 0 on starting script execution. On encountering an "output pays fee" `CTVHASH`, increment this variable. If the variable is now 2 or higher, fail validation.
* The transaction must have exactly 1 input. Otherwise, fail validation.
* `nSequence` of the sole input must be `0x00000000`.
* If the "amount to deduct" is specified, then perform the following additional validation:
  * Get the amount of this input, then subtract the amount to deduct.
  * The amount of the output at the given output index must be **greater than or equal to** the above difference.

> **Rationale** The above prevents quadratic hashing via this new "output pays fee" `CTVHASH` extension. As this changes the hashing of the outputs, the output hashes cannot be cached as in the original BIP-119. Instead, the above rules prevent quadratic hashing by only allowing a single heavyweight hash operation per transaction.
>
> We specify that the script interpreter maintains a "heavy CTV hashes" variable so that future `CTVHASH` types may also use the same variable, preventing quadratic hashing by using different heavyweight `CTVHASH` types.
>
> The transaction can feasibly be limited to exactly 1 input since this mechanism is intended to replace other mechanisms where fees are paid by other inputs. Instead, the fees are paid from the given single input, which serves as a shared fund, with one of the outputs being reduced in order to fund the mining fee.
>
> An `nSequence` of `0` forces the transaction to opt-in to RBF (which is a desired feature of this mechanism, which allows one participant to RBF freely by deducting the output it owns, which is specified in the stack argument here), and enables `nLockTime` if that is needed.
>
> The amount to deduct, if specified, prevents the output from being entirely used up to pay for fees. This may be used if the output is still shared among multiple participants, as in Decker-Russell-Osuntokun or intermediate outputs in CTV-trees. The amount is subtracted from the input, and the input is **not** committed to in the hash, to allow intermediate `OP_CHECKTEMPLATEVERIFY` transactions to have variable fees that can adapt to changing onchain conditions, with having the total fees of all intermediate transactions debited from a final output.

The template hash is then computed by hashing the below:

* All but the last 32 bytes of the stack top argument (i.e. `CTVHASH` type, the formatted output index that pays fees, plus the optional amont to deduct, in the formatted order).
* `nVersion`
* `nLockTime`
* output count
* output hash, modified as follows:
  * For the output at the index specified in the stack top argument, set the amount as 0.

> **Rationale** As this `CTVHASH` type is intended to support an "internal fees" mechanism, as noted above, it already restricts the number of inputs to just 1. There is thus no need to commit to the number of inputs or the `nSequence` (which is fixed above, as well), or the input index this is executing on.

---------

The uses of this new `CTVHASH` mode are congruent to the original posting up top, and the equivalence is left to the reader.

We can create a CTV-tree where fees are dynamically decided at publish time, with all fees paid internally instead of externally (i.e. there is no need to own another UTXO that pays for fees) by using this mechanism. For example, suppose we want to commit to outputs A, B, C, D with particular amounts. We construct a root address that spends to 4 possible tapleaves to create a 1-input, 2-output top node:

* "output pays fee" `CTVHASH` that points to `A,B` output paying up to some max fee plus fixes the `C,D`output, plus a `OP_CHECKSIG` from `A`.
* "output pays fee" `CTVHASH` that points to `A,B` output paying up to some max  fee plus fixes the `C,D` output, plus a `OP_CHECKSIG` from `B`.
* "output pays fee" `CTVHASH` that points to `C,D` output paying up to some max  fee plus fixes the `A,B` output, plus a `OP_CHECKSIG` from `C`.
* "output pays fee" `CTVHASH` that points to `C,D` output paying up to some max  fee plus fixes the `A,B` output, plus a `OP_CHECKSIG` from `D`.

Then, the `A,B` (corresponding `C,D`) output of the top node would have 2 possible tapleaves to create a 1-input, 2-output final node:

* "output pays fee" `CTVHASH` that points to `A` output paying the fee plus fixes the `B` output, plus a `OP_CHECKSIG` from `A`.
* "output pays fee" `CTVHASH` that points to `B` output paying the fee plus fixes the `A` output, plus a `OP_CHECKSIG` from `B`.

Then, if a participants wants to publish their output, they can instantiate the path to their node, paying all fees for intermediate nodes and the final node in the CTV-tree.

-------------------------

