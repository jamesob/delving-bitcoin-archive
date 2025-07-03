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

