# Understanding and Mitigating a OP_CTV Footgun: The Unsatisfiable UTXO

Chris_Stewart_5 | 2025-07-03 20:09:21 UTC | #1

Hi everyone,

A well-known "footgun" when working with `OP_CTV` (CheckTemplateVerify) is the scenario where an amount **less** than the expected value is sent to a `OP_CTV`-locked output. As `BIP119` (the proposal for `OP_CTV`) highlights in its discussion on ["forwarding address contracts"](https://github.com/bitcoin/bips/blob/fd1955694b95440bde70890475548dfb59e2e759/bip-0119.mediawiki#forwarding-addresses):

>Key-reuse with CHECKTEMPLATEVERIFY may be used as a form of "forwarding address contract". A forwarding address is an address which can automatically execute in a predefined way. For example, an exchange's hot wallet might use an address which can automatically be moved to a cold storage address after a relative timeout.

>The issue is that reusing addresses in this way can lead to loss of funds. Suppose one creates a template address which forwards 1 BTC to cold storage. Creating an output to this address with less than 1 BTC will be frozen permanently. Paying more than 1 BTC will lead to the funds in excess of 1BTC to be paid as a large miner fee. CHECKTEMPLATEVERIFY could commit to the exact amount of bitcoin provided by the inputs/amount of fee paid, but as this is a user error and not a malleability issue this is not done. Future soft-forks could introduce opcodes which allow conditionalizing which template or script branches may be used based on inspecting the amount of funds available in a transaction

**A Key Realization: The Importance of Multiple Inputs with `OP_CTV`**

A significant realization I had while developing this test case is that it seems fundamentally prudent to **commit to at least two inputs** when constructing an `OP_CTV` template. I can't identify any significant benefits to committing to a single input, while the downsides — specifically the risk of creating permanently unspendable `UTXOs` — are substantial.

If your `OP_CTV` template is designed to include a minimum of two inputs, you retain the ability to "craft" a secondary input to perfectly satisfy the total amount locked in the `OP_CTV` template. I've demonstrated this recovery mechanism in a [Python test here](https://github.com/Christewart/bitcoin/blob/6e13681b0b1612c7f796d7a81bb4ac63062be7fd/test/functional/feature_ctv_amount.py#L118).

This capability becomes especially relevant for applications like **vault constructions**, a use case many are excited about for `OP_CTV`. Vaults often involve locking large amounts of funds within `OP_CTV` outputs. Given that transaction hashes don't inherently convey the exact expected amount, having an "escape hatch" in case of an accidental underfunding (or even a slight amount mismatch due to floating-point arithmetic or human error) is incredibly valuable and prudent.

**The Trade-off:**

It's important to note that designing your `OP_CTV` template to always require two inputs means your spending transaction will inherently be larger in size and incur a slightly higher transaction fee, even if the primary `OP_CTV` input was perfectly sized. However, I believe the added peace of mind and the ability to recover otherwise frozen funds in the event of an amount mismatch far outweigh this marginal increase in cost.

### Future work

This research is part of piece of research on how OP_CTV can work with [OP_{IN,OUT}_AMOUNT](https://delvingbitcoin.org/t/op-inout-amount/549) to create safer forwarding addresses and realize the [AMOUNTVERIFY](https://github.com/bitcoin/bips/blob/fd1955694b95440bde70890475548dfb59e2e759/bip-0119.mediawiki#op_amountverify) idea written in BIP119.

-------------------------

instagibbs | 2025-07-04 13:24:37 UTC | #2

Aside from the additional costs involved, requiring an additional input to be created at the right amount (to free up the committed tx and also not overspend on fees!) and then spent, committing to two inputs also means that you don't get transaction id stability. 

This precludes use-cases such as connector/control outputs being made by these transactions, for instance.

-------------------------

1440000bytes | 2025-07-04 14:07:13 UTC | #3

It is also possible to have multiple spending paths with different number of inputs committed by CTV.

[quote="Chris_Stewart_5, post:1, topic:1809"]
This capability becomes especially relevant for applications like **vault constructions**, a use case many are excited about for `OP_CTV`. Vaults often involve locking large amounts of funds within `OP_CTV` outputs. Given that transaction hashes don’t inherently convey the exact expected amount, having an “escape hatch” in case of an accidental underfunding (or even a slight amount mismatch due to floating-point arithmetic or human error) is incredibly valuable and prudent.
[/quote]

Vaults will be used with a notification service or watchtower that can monitor such transactions, notify the user, replace transactions, etc.

-------------------------

Chris_Stewart_5 | 2025-07-04 18:07:40 UTC | #4

[quote="1440000bytes, post:3, topic:1809"]
Vaults will be used with a notification service or watchtower that can monitor such transactions, notify the user, replace transactions, etc.
[/quote]

Assuming we're dealing with an `OP_CTV` template that commits to **exactly one input**, I don't believe we can "replace" an already created, unsatisfiable `UTXO` in the way one might replace an unconfirmed transaction via RBF. Once a transaction creating such an underfunded `OP_CTV` `UTXO` is confirmed, that `UTXO` becomes a permanent part of the `UTXO` set. Its `OP_CTV` script's requirements (including the exact amount) are set in stone, effectively locking the funds if the received amount doesn't match the committed amount precisely.

A traditional watchtower monitors for *spending* attempts of the `OP_CTV` `UTXO` - this wouldn't be able to help with an *underfunded funding transaction* that has already confirmed. The watchtower would only see the `OP_CTV` `UTXO` available for spending, but wouldn't inherently know that it's unspendable due to an amount mismatch.

For a watchtower to truly help in this "underfunding" scenario, it would need to:

1. **Be aware of the `OP_CTV` hash preimage at the time of the `UTXO`'s creation:** This means the watchtower would need to know the *exact* transaction template that the `OP_CTV` hash commits to, including the expected input amount, before the funding transaction is even broadcast.
2. **Monitor the funding transaction:** It would then have to compare the actual amount received by the `OP_CTV` output in the funding transaction against the expected amount from the preimage.
3. **Alert the user *before confirmation*:** The watchtower would need to detect the mismatch and alert the user *while the funding transaction is still unconfirmed* (if it was an RBF-eligible transaction) so that it could potentially be replaced.

The challenge here is that the specific amount committed to within the `OP_CTV` hash is **not readily apparent from data available on-chain** when the `OP_CTV` `UTXO` is created. This commitment is only fully revealed when an attempt is made to *spend* the `OP_CTV` `UTXO` by providing the pre-image and the full transaction template as part of the witness. Therefore, a watchtower simply monitoring the blockchain wouldn't know at the funding stage that the `UTXO` is unsatisfiable.

This reinforces my point about committing to at least two inputs in the `OP_CTV` template. This design choice effectively provides a "rescue path" for correcting amount mismatches *after* the `UTXO` has been created.

-------------------------

1440000bytes | 2025-07-04 18:22:21 UTC | #5


[quote="Chris_Stewart_5, post:4, topic:1809"]
For a watchtower to truly help in this “underfunding” scenario, it would need to:
[/quote]

I have no clue why you can't setup a watchtower that does all of this if you already need one for vault.

-------------------------

salvatoshi | 2025-07-05 07:03:01 UTC | #6

To the best of my understanding, it is in general true that for (future) Script-based covenants based on programmatic restriction on the outputs (as opposed to restrictions obtained via presigned transactions), stability of txids doesn't seem to be useful. Therefore a simplified version of CTV that only commits to the outputs would allow greater flexibility, at no cost. Vaults constructed with CCV+CTV (essentially the same as VAULT+CTV) are an example.

In constructions with such expressive covenants, CSFS might also be used as a means of enabling certain spending paths without depending on stable txids, by signing a message instead.

However, *connector outputs* seem to be somewhat of an exception in that I am not aware of an equally easy *covenant-y* replacement for this primitive. Generalizations not based on presigned transactions do exist (see [ancestry proofs/singletons](https://delvingbitcoin.org/t/contract-level-relative-timelocks-or-lets-talk-about-ancestry-proofs-and-singletons/1353)) and it would be interesting to further explore what opcodes would enable the cleanest implementation.

-------------------------

jamesob | 2025-07-09 19:06:30 UTC | #7

[quote="salvatoshi, post:6, topic:1809"]
Therefore a simplified version of CTV that only commits to the outputs would allow greater flexibility, at no cost.
[/quote]

Doesn't this introduce a trivial pinning vector where an observer can tack on a very heavy input during broadcast? Is there some way of mitigating that?

-------------------------

salvatoshi | 2025-07-09 21:52:52 UTC | #8

I'm not yet very knowledgeable on mempool policies, but how is this any better with CTV, which only commits to the number of inputs?

That seems to be an issue with any script that doesn't commit to all inputs, rather than tied to specific opcodes.

-------------------------

ajtowns | 2025-07-10 00:10:33 UTC | #9

[quote="salvatoshi, post:8, topic:1809"]
I’m not yet very knowledgeable on mempool policies, but how is this any better with CTV, which only commits to the number of inputs?
[/quote]

If there's a commitment that there's a single input exactly, then you can't add extraneous inputs, so pinning in that direction isn't possible.

-------------------------

jamesob | 2025-07-10 01:00:41 UTC | #10

FWIW, there are a number of other reasons that only committing to the output side has problems (half-spend problem, malleation issues). See [rationale from BIP-119](https://github.com/bitcoin/bips/blob/c17a3dbcebafb6d48a8510438ab9dab58ece1add/bip-0119.mediawiki#committing-to-the-number-of-inputs).

-------------------------

