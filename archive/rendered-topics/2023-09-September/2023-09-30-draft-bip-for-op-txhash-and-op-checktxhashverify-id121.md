# Draft BIP for OP_TXHASH and OP_CHECKTXHASHVERIFY

stevenroose | 2023-09-30 22:21:50 UTC | #1

CC from the mailing list, so that we can have discussion here as well.

--- 
Hey all


The idea of TXHASH has been around for a while, but AFAIK it was never formalized. After conversations with Russell, I worked on a specification and have been gathering some feedback in the last several weeks.

I think the draft is in a state where it's ready for wider feedback and at the same time I'm curious about the sentiment in the community about this idea.

The full BIP text can be found in the attachment as well as at the following link:
https://github.com/bitcoin/bips/pull/1500

I will summarize here in this writing.

**What does the BIP specify?**

* The concept of a TxFieldSelector, a serialized data structure for selecting data inside a transaction.
  * The following global fields are available:
    - version
    - locktime
    - number of inputs
    - number of outputs
    - current input index
    - current input control block
  * For each input, the following fields are available:
    - previous outpoint
    - sequence
    - scriptSig
    - scriptPubkey of spending UTXO
    - value of spending UTXO
    - taproot annex
  * For each output, the following fields are available:
    - scriptPubkey
    - value
  * There is support for selecting inputs and outputs as follows:
    - all in/outputs
    - a single in/output at the same index as the input being executed
    - any number of leading in/outputs up to 2^14 - 1 (16,383)
    - up to 64 individually selected in/outputs (up to 2^16 or 65,536)
    - The empty byte string is supported and functions as a default value which commits to everything except the previous outpoints and the scriptPubkeys of the spending UTXOs.

* An opcode OP_TXHASH, enabled only in tapscript, that takes a serialized TxFieldSelector from the stack and pushes on the stack a hash committing to all the data selected.

* An opcode OP_CHECKTXHASHVERIFY, enabled in all script contexts, that expects a single item on the stack, interpreted as a 32-byte hash value concatenated with (at the end) a serialized TxFieldSelector. Execution fails is the hash value in the data push doesn't equal the calculated hash value based on the TxFieldSelector.

* A consideration for resource usage trying to address concerns around quadratic hashing. A potential caching strategy is outlined so that users can't trigger excessive hashing.
  * Individual selection is limited to 64 items.
  * Selecting "all" in/outputs can mostly use the same caches as sighash calculations.
  * For prefix hashing, intermediate SHA256 contexts can be stored every N items so that at most N-1 items have to be hashed when called repeatedly.
  * In non-tapscript contexts, at least 32 witness bytes are required and because (given the lack of OP_CAT) subsequent calls can only re-enforce the same TxFieldSelector, no additional limitations are put in place.
  * In tapscript, because OP_TXHASH doesn't require 32 witness bytes and because of a potential addition of operations like OP_CAT, the validation budget is decreased by 10 for every OP_TXHASH or OP_CHECKTXHASHVERIFY operation.


**What does this achieve?**

* Since the default TxFieldSelector is functionally equivalent to OP_CHECKTEMPLATEVERIFY, with no extra bytes required, this proposal is a strict upgrade of BIP-119.

  * The flexibility of selecting transaction fields and in/output (ranges), makes this construction way more useful
    - when designing protocols where users want to be able to add fees to their transactions without breaking a transaction chain;
    - when designing protocols where users construct transactions together, each providing some of their own in- and outputs and wanting to enforce conditions only on these in/outputs.

* OP_TXHASH, together with OP_CHECKSIGFROMSTACK (and maybe OP_CAT*) could be used as a replacement for almost arbitrarily complex sighash constructions, like SIGHASH_ANYPREVOUT.

* Apart from being able to enforce specific fields in the transaction to have a pre-specified value, equality can also be enforced, which can f.e. replace the desire for opcodes like OP_IN_OUT_VALUE.*
* The same TxFieldSelector construction would function equally well with a hypothetical OP_TX opcode that directly pushes the selected fields on the stack to enable direct introspection.


**What are still open questions?**

* Does the proposal sufficiently address concerns around resource usage and quadratic hashing?

* *: Miraculously, once we came up with all possible fields that we might consider interesting, we filled exactly 16 spots. There is however one thing that I would have liked to be optionally available and I am unsure of which side to take in the proposal. This is including the TxFieldSelector as part of the hash. Doing so no longer makes the hash only represent the value being hashed, but also the field selector that was used; this would no longer make it possible to proof equality of fields. If a txhash as specified here would ever be used as a signature hash, it would definitely have to be included, but this could be done after the fact if OP_CAT was available. For signature hashes, the hash should ideally be somehow tagged, so we might need OP_CAT, or OP_CATSHA256 or something anyway.

  * A solution could be to take an additional bit from each of the two "in/output selector" bytes, and assign to this bit "commit to total number of in/outputs" (instead of having 2 bits for this in the first byte).
    * This would free up 2 bits in the first byte, one of which could be used for including the TxFieldSelector in the hash and the other one could be left free (OP_SUCCESS) to potentially revisit later-on.
    * This would limit the number of selectable leading in/outputs to 8,191 and the number of individually selectable in/outputs to 32, both of which seem reasonable or maybe even more desirable from a resource usage perspective.

* General feedback of how people feel towards a proposal like this, which could either be implemented in a softfork as is, like BIP-119 or be combined in a single softfork with OP_CHECKSIGFROMSTACK and perhaps OP_CAT, OP_TWEAKADD and/or a hypothetical OP_TX.


This work is just an attempt to make some of the ideas that have been floating around into a concrete proposal. If there is community interest, I would be willing to spend time to adequately formalize this BIP and to work on an implementation for Bitcoin Core.


Looking forward to your thoughts

Steven

-------------------------

