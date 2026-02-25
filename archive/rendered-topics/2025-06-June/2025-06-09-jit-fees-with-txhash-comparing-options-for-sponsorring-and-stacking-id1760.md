# JIT fees with TXHASH: comparing options for sponsorring and stacking

stevenroose | 2025-06-09 10:29:38 UTC | #1

Specifying fees upfront in second layer solutions is annoying. Both predicting future feerates and deciding how to allocate the money for the fee are non-trivial problems that can be avoided if fees can be paid when they are needed (just-in-time): when (usually pre-signed) off-chain contracts go on-chain.

I have two aims with this post:
1. compare different options for simple fee sponsorring both for a single and for multiple transactions
2. show that TXSIGHASH can be used for effective stacking of transactions by third parties

I make a comparisong for paying fees JIT with CPFP, Tx Sponsors and TXSIGHASH, both for a single transaction and for stacks of transactions. You can skip to the [summary table](#p-5256-summary-table-16) or the bit on [TXHASH-based stacking](#p-5256-txsighash-15) if you wish.


## CPFP

The currently available solution for paying fees JIT is [CPFP, or child-pays-for-parent](https://bitcoinops.org/en/topics/cpfp/):
a new transaction is created that spends the original transaction and can add fees to cover the entire [_package_](https://bitcoinops.org/en/topics/package-relay/).

The advantage of CPFP is that it uses existing transaction dependency rules (doesn't need any special-case consensus rules) and that all transactions involved have a stable txid. The disadvantage is that it is quite costly, as a fee anchor output needs to be made and an entire new child transaction with at least 2 inputs.

## Tx Sponsors

There is currently a proposal to add a construction to bitcoin's consensus that is aimed specifically at solving this problem more elegantly and more efficiently: [Tx Sponsors](https://bitcoinops.org/en/topics/fee-sponsorship/).

The idea of sponsors at a high level is that a transaction can specify any other transaction by its txid and consensus would require that transactioon to be included in the same block. Like this, the new transaction effectively "sponsors" the other transaction to encourage its inclusion in a block.

I'll be basing on the [more recent ideas](https://delvingbitcoin.org/t/improving-transaction-sponsor-blockspace-efficiency/696) of David Harding on how to implement sponsorship:

- a new kinda input/output marker is to be used to indicate a transaction is a sponsor, this could be free (below denoted as `p2tr-spon`)
- the sponsored txids are committed into the sighash of the sponsor transaction's input
- the sponsor transaction has an additional witness element with 2 bytes per sponsored which hold the offsets of the txids in the block


## TXHASH

[TXHASH](https://covenants.info/proposals/txhash/) is a proposal for a consensus change that adds constructions to introspect the current transaction from the bitcoin Script system. The BIP currently specifies two opcodes, `OP_TXHASH` and `OP_CHECKTXHASHVERIFY` that can be used inside the Script system to inspect or assert certain transaction properties.

It specifies the concept of a `TxFieldSelector` that is used to exactly specify what parts of the transaction the user wants to have included in their _tx hash_. The `TxFieldSelector` is quite powerful and supports:

- selecting top-level transaction fields like version and lockTime
- selecting all individual fields of the inputs and outputs
- selecting inputs and outputs one wants to include, by either
  - selecting all in/outputs
  - selecting the leading N in/outputs
  - selecting individual in/outputs by their absolute index
  - selecting individual in/outputs by their relative index to the current inputs
- selecting various fields related to the current execution like input index, control block, annex, OP_CODESEPARATOR etc

### TXSIGHASH

In this write-up, I will use a different construction based on TXHASH: a TXHASH-based sighash. I do this because using the `OP_TXHASH` opcode comes with a significant size penalty because of how tapscript works and if TXHASH would become popular, adopting a sighash system based on TXHASH will be a reasonable decision.

The construction defines a new key type for p2tr outputs. For example 33-byte p2tr outputs with the first byte being 0x01[^4]. For such outputs, the signature field is interpreted differently: all bytes appended to the 64-byte signature are interepreted as a `TxFieldSelector` and the sighash for the signature is calculated as if it was the _tx hash_ calculated as per TXHASH fasion for the given `TxFieldSelector`.

To pay fees using TXSIGHASH, the original transaction will use a sighash that only commits to its own inputs and outputs and the sponsor will add an additional input and output that will commit to all inputs and outputs, including the additional ones. This locks in the original transaction's inputs and outputs.

One important note here is that this will obviously change the txid of the transaction. Currently, many constructions like Lightning and Ark are built in a way that they require txids to be stable and reliable. However, both [Lightning](https://bitcoinops.org/en/topics/eltoo/) and [Ark](https://delvingbitcoin.org/t/re-evolving-the-ark-protocol-using-ctv-and-csfs/1667) can be build differently using rebindable signatures to no longer rely on this property.


# Scenarios

I will outline a few different scenarios, explain how spending works using TXSIGHASH and compare the costs with CPFP and tx sponsors.

For our base cases, I will work with transactions that have simple p2tr key-spend inputs and outputs[^1]. 

## Single-transaction Sponsorring

### 1-in-1-out

Let's start with the simplest case: a simple transaction with a single input and a single output.

```text
| inputs     |   outputs | 
+============+===========+
| p2tr-ks    |      p2tr |
+------------+-----------+
```

This simple transaction has a virtual size of 111 vB.


#### CPFP

For CPFP, the transaction would have to be extended with a fee anchor ([p2a](https://bitcoinops.org/en/topics/ephemeral-anchors/)) as follows:

```text
| inputs     |   outputs | 
+============+===========+
| p2tr-ks    |      p2tr |
+------------+-----------+
|            |    anchor |
+------------+-----------+
```

The transaction than becomes 124 vB.

The "child" transaction that will pay the fee, will have to spend the fee anchor and add additional funds (using p2tr). That transaction looks as follows:

```text
| inputs     |   outputs | 
+============+===========+
| anchor     |      p2tr |
+------------+-----------+
| p2tr-ks    |           |
+------------+-----------+
```

This transaction has 153 vB. The complete construction here, parent + child, totals 277 vB.

### Tx Sponsors

In order to sponsor a transaction, no changes have to made to the original sponsoree. It can remain a 1-in-1-out transaction.

The sponsor would then create an additional new transaction, the _sponsor transaction_, that looks as follows:

```text
| inputs     |   outputs | 
+============+===========+
| p2tr-spon* |      p2tr |
+------------+-----------+
```

In the simple case, the `p2tr-spon` input would have an additinal input witness element of 2 bytes that indicates the relative index of the sponsoree transaction in the block.

This transaction has 112 vB.


#### TXSIGHASH

In order for this transaction to be used with TXSIGHASH, all `scriptPubkeys` will need to flag TXSIGHASH support with an additional byte. In theory, only the input strictly needs to support it, but for all fairness, we will also add the extra byte on the outputs, because if the construction needs to be used further along, the byte will have to be present. I will denote this as follows:

```text
| inputs     |   outputs | 
+============+===========+
| p2txsh-ks  |    p2txsh |
+------------+-----------+
```
This transactions has 113 vB.

Note that the sighash of this transaction will only cover the first input and first output. This can be cheaply expressed using a 4-byte `TxFieldSelector`. The same 4-byte cost remains for other transactions as long as the number of in/outputs doesn't exceed 7936; you can have a different number of inputs and outputs.


In order to pay fees for this transaction using external funds that support TXSIGHASH, we can construct a transaction by simply adding our funding input and creating an additional output, as follows:

```text
| inputs     |   outputs | 
+============+===========+
| p2txsh-ks  |    p2txsh |  <- original
+------------+-----------+
| p2txsh-ks  |    p2txsh |  <- sponsor
+------------+-----------+
```

As we described above, the sighash of the first input will cover only the first input and output (4-byte `TxFieldSelector`) and the sighash of the last input will cover the entire transaction (1-byte `TxFieldSelector` [^2]).

This entire transaction has 215 vB.


### Other transaction configurations

Other common transaction configurations are the common 2-in-2-out for regular payments that require change or Lightning channels. We will also cover the 1-in-4-out case which is very common in constructions using transaction trees like Ark.

All numbers are in virtual bytes (vB) and the difference compared with the CPFP case.

|            | base | CPFP | Tx Sponsors    | TXSIGHASH      |
|-|-|-|-|-|
| 1-in-1-out | 111  | 277  | 223 (-54,-19%) | 215 (-62,-22%) |
| 2-in-2-out | 212  | 378  | 324 (-54,-14%) | 318 (-60,-16%) |
| 1-in-4-out | 240  | 406  | 352 (-54,-13%) | 347 (-59,-15%) |



## Multi-transaction Sponsorring or Stacking

Next, we want to look at scenarios where a single sponsor sponsors multiple transactions at once.

We will explore the case of stacking 10 and 100 transactions, but will use 3 in our example diagrams below.

### CPFP

Sponsorring multiple transactions using CPFP can be done by having a single child spend all the anchors from all the transactions that one wants to sponsor. Under the current mempool rules, it is not yet possible to relay such transactions, but work is almost done to [support this](https://bitcoinops.org/en/topics/package-relay/).

Constructions are very simple, the regular transactions can remain exactly the same and the child will simply have to spend all the anchors.

For the case of sponsorring 3 transactions, the child (sponsor) transaction looks as follows:

```text
| inputs     |   outputs | 
+============+===========+
| anchor     |      p2tr |
+------------+-----------+
| anchor     |           |
+------------+-----------+
| anchor     |           |
+------------+-----------+
| p2tr-ks    |           |
+------------+-----------+
```


### Tx Sponsors

Sponsorring multiple transactions using Tx Sponsors is very easy. The original transaction don't have to be changed and can just be relayed and included as they are. Possibly it would be required for the author of the transaction to mark the transaction as eligible for sponsorship, but this can be done without costing any additional bytes.

The sponsor transaction still looks the same, having a single input and a single output, but having an additional 2 witness bytes per transaction it is sponsorring.


### TXSIGHASH

For the case of TXSIGHASH, we will look into _stacking_. Stacking is when multiple semantic transactions are squashed into a single transaction that encompases all original semantic transactions (meaning their inputs and outputs). It is somewhat similar to doing a coinjoin round, but stacking is generally considered non-interactive.

Transactions of arbitrary fan-in-fan-out can be stacked using TXSIGHASH. However, special care have to be taken by the _stackees_. This is because the signatures used in the stacked inputs will be different depending on the resulting stacking configuration. If one wants to allow an external _stacker_ to stack their transactions, multiple different signatures have to be generated in order to facilitate various different stacking configurations and these different signatures will have to be communicated to the _stacker_. This can either be done by an out-of-band protocol provided by the _stacker_ for his _stackees_ (similar to current coinjoin protocols etc), or a special-purpose relay network can be created to relay ready-to-go stack packages. (The latter is relevant because not all packages might need external fees, but being included in a stack can still allow you to save fees for the fixed transaction boilerplate bytes.)

In the case of off-chain protocols where these transactions are pre-signed, the same variation of signatures will have to be made beforehand and stored by all parties involved in the protocol.

Before we lay out an example stack, I'll explain the strategy. We will use the term _stackee tx_ for a semantic transaction, consisting of some inputs and some outputs. Intuitively, they are like bitcoin transactions, but in the stacking process, they might be pulled apart, while still ensuring that they always must all be included in a transaction together (atomicity).

We will use the `TxFieldSelector` relative indices so that sighashes of the _stackee tx_ inputs can cover the desired outputs. We make the following reasonings:

- The user doesn't care in what order their inputs and outputs are included, so they can re-order them to suite the stacking process.
- We make the assumption here that "inputs don't require to be included", meaning that if some external party decides to fund the outputs instead, a user will be happy to get free money. If for some external reason, a user requires his inputs to be included, a variation of this stacking protocol can be designed but it will be slightly more costly.
- We use the first input of the _stackee tx_ to fix all outputs.
- All other inputs will commit to the existence of the first input to indirectly also commit to the outputs.
- Inputs will always be aligned, meaning that all inputs of a _stackee tx_ will follow each other in the resulting _stack tx_.
- The case where the number of outputs is equal or less than the number of inputs is easy:
  - The first input commits to the outputs at relative indices $0, 1, .., N-1$ for N outputs.
  - Each other input commits to input at relative index $-1, -2, .. -(N-1)$[^3]
  - For this case, only exactly $N$ signatures have to be produced.
- If there are more outputs than inputs, the user will generate a number of different signature variants, where the relative index for the outputs with indices after the last input's index will be moved each time one further. This allows the _stacker_ to pick the right signatures matching the eventual configuration of the _stack tx_.
- Since all indices are relative, there are no strict limits on how large a _stack_ can get, especially if on average the number of inputs and outputs kind matches and the puzzle algorithm finds ways to construct stacks where outputs don't have to put places too far away from the inputs.

This is a somewhat naive version of a stacking algorithm. Many optimizations can be made to increase the likelihood that a set of transactions can be stacked. This is however out of the scope of this write-up as it doesn't affect the on-chain costs of the stacks.

Let's look at an example then. We will try to stack the following three _stackee txs_:

- transaction A with 2 inputs and 1 outputs
  ```text
  | inputs     |   outputs | 
  +============+===========+
  | A1         |        A1 |
  +------------+-----------+
  | A2         |           |
  +------------+-----------+
  ```
- transaction B with 1 inputs and 3 outputs
  ```text
  | inputs     |   outputs | 
  +============+===========+
  | B1         |        B1 |
  +------------+-----------+
  |            |        B2 |
  +------------+-----------+
  |            |        B3 |
  +------------+-----------+
  ```
- transaction C with 2 inputs and 3 outputs
  ```text
  | inputs     |   outputs | 
  +============+===========+
  | C1         |        C1 |
  +------------+-----------+
  | C2         |        C2 |
  +------------+-----------+
  |            |        C3 |
  +------------+-----------+
  ```

The _stacker_ will ask the _stackees_ to provide signatures up to $N$ relative distance for the additional outputs. In reality, $N$ will be in the order of magnitude of 100, but for this example, we will show the case where $N$ equals $5$. The _stacker_ will then receive the following signatures:

- input A1 committing to output A1 at same index
- input A2 committing to input A1 at index -1
- 5 signatures for input B1 committing to outputs B1, B2 and B3 at indices
  - +0, +1, +2
  - +0, +2, +3
  - +0, +3, +4
  - +0, +4, +5
  - +0, +5, +6
- 5 signatures for inputs C1 committing to outputs C1, C2 and C3 at indices
  - +0, +1, +2
  - +0, +1, +3
  - +0, +1, +4
  - +0, +1, +5
  - +0, +1, +6
- input C2 committing to input C1 at index -1

The _stacker_ can add an input and output S (using SIGHASH_ALL) to pay for the fees, and construct the following _stack tx_:

```text
| inputs     |   outputs | 
+============+===========+
| C1         |        C1 |
+------------+-----------+
| C2         |        C2 |
+------------+-----------+
| A1         |        A1 |
+------------+-----------+
| A2         |        C3 |
+------------+-----------+
| B1         |        B1 |
+------------+-----------+
| S          |        B2 |
+------------+-----------+
|            |        B3 |
+------------+-----------+
|            |         S |
+------------+-----------+
```
Alternatively, he could have constructed this transaction as well:
```text
| inputs     |   outputs | 
+============+===========+
| B1         |        B1 |
+------------+-----------+
| A1         |        A1 |
+------------+-----------+
| A2         |        B2 |
+------------+-----------+
| S          |        B3 |
+------------+-----------+
| C1         |        C1 |
+------------+-----------+
| C2         |        C2 |
+------------+-----------+
|            |        C3 |
+------------+-----------+
|            |         S |
+------------+-----------+
```


### Summary table

Below is a table with the costs for sponsorring/stacking 10 and 100 _identical_ transactions. I chose identical transactions because it is otherwise really hard to make random configurations.


|            | N   | CPFP  | Tx Sponsors   | TXSIGHASH     |
|-|-|-|-|-|
| 2-in-2-out | 10  | 2769  | 2232 (-537)   | 2165 (-604)   |
| 2-in-2-out | 100 | 26686 | 21312 (-5374) | 20638 (-6048) |
| 1-in-4-out | 10  | 3054  | 2517 (-537)   | 2465 (-589)   |
| 1-in-4-out | 100 | 29536 | 24162 (-5374) | 23640 (-5896) |

Or in vB per _stackee tx_:

|            | N   | base | CPFP | Tx Sponsors    | TXSIGHASH      |
|-|-|-|-|-|-|
| 2-in-2-out | 1   | 212  | 378  | 324 (-54,-14%) | 318 (-60,-16%) |
| 2-in-2-out | 10  | 212  | 277  | 223 (-54,-19%) | 217 (-60,-22%) |
| 2-in-2-out | 100 | 212  | 267  | 213 (-54,-20%) | 207 (-60,-23%) |
| 1-in-4-out | 1   | 240  | 406  | 352 (-54,-13%) | 347 (-59,-15%) |
| 1-in-4-out | 10  | 240  | 306  | 252 (-54,-18%) | 247 (-59,-19%) |
| 1-in-4-out | 100 | 240  | 296  | 242 (-54,-18%) | 237 (-59,-20%) |


# Conclusion

TXSIGHASH based constructions provide an extremely cost-effective way for both sponsorring single transactions and for stacking multiple transactions together, **possibly by a third party. With stacking, the cost in virtual bytes of each stacked transaction can even be lower than their original cost without a sponsor included**.

It is also worth noting that stacking with TXSIGHASH results in having a single big transaction instead of a series of multiple transactions, like both CPFP and Tx Sponsors. Additionally, all inputs are "simple"[^5] key-spends, meaning that they could be aggregated if [CISA](https://bitcoinops.org/en/topics/cross-input-signature-aggregation/) were to be deployed.

The drawback of sponsorring and stacking using TXSIGHASH is the overhead of creating, storing and relaying multiple variants of the signatures used in the transaction. This has to be done by the creators of any transaction that wants to be eligible for stacking, at the time of its creation. This added complexity can be considered quite significant, especially when compared with Tx Sponsors which doesn't require any special preparation by the original transactions.





[^1]: In theory other policies could be used, especially if they include a signature check that can use TXSIGHASH semantics. Other covenant-based policies should be reviewed on an individual basis.
[^2]: In theory this could maybe be even a 0-byte `TxFieldSelector` as it would make sense to make the "default" sighash type to be ALL.
[^3]: The careful reader will have noted that this construction is vulnerable to an attack where if the user for some reason produces two different signatures for the first input, the other inputs could be linked to the other spends as well. This can be dangerous. Full atomicity can be guaranteed by putting commitments into the annex, but this requires quite some changes. A possible simple and free improvement is that the other inputs also commit to the outputs below their own index. By carefully ordering the outputs by value and putting the largest one at the index equivalent with the last input, the signatures can be made significantly more safe.
[^4]: This one byte could be saved by defining a new segwit version (v2).
[^5]:  "simple" because they do use different SIGHASHES, but still only have a single pubkey and signature per input

-------------------------

ademan | 2025-09-30 20:25:10 UTC | #2

This is possibly a dumb question, but why does your stacking protocol not keep outputs contiguous? If I’m reading the TXHASH BIP correctly, you could also commit to output indices (-2, -1, 0), (-1, 0, +1), (0, +1, +2), (+1, +2, +3), (+2, +3, +4) for instance.

It’s not immediately clear to me if either approach should be successful more often, though I suspect they should be similar. Keeping outputs contiguous was the “obvious” approach to me so I’m wondering if I am missing an obvious problem with it.

-------------------------

stevenroose | 2026-02-25 16:14:01 UTC | #3

Tbh those are optimization problems where various strategies can exist and work.

Providing “current” is cheaper than an index. And eventually the optimization problem will be to minimize the total number of different signatures required from the users. There is probably space for more naive algorithms that required slightly more signatures and smarter algorithms that require less.

-------------------------

