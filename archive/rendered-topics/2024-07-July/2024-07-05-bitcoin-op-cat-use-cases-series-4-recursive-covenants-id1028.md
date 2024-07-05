# Bitcoin OP_CAT Use Cases Series #4: Recursive Covenants

sCrypt-ts | 2024-07-05 11:12:36 UTC | #1

# Bitcoin OP_CAT Use Cases Series #4: Recursive Covenants
## Stateful Bitcoin Smart Contracts

# Recursive Covenants

As we explained in [series #1](https://scryptplatform.medium.com/trustless-ordinal-sales-using-op-cat-enabled-covenants-on-bitcoin-0318052f02b2), a Bitcoin covenant is a mechanism that allows the sender of a Bitcoin transaction to impose certain conditions on how the receiver can spend the coins.

A recursive covenant is a type of covenant that applies not only to the immediate transaction but also to any subsequent transactions that spend the bitcoins. This means that the conditions imposed by the covenant could be enforced recursively in perpetuity.

The main difference between a regular non-recursive covenant and a recursive covenant is the scope of the conditions. A regular covenant only applies to the immediate next transaction, while a recursive one extends to all future transactions that spend the bitcoins.

Here’s a simple example to illustrate the difference:

* Regular covenant: Alice sends Bob 1 BTC with the condition that Bob can only spend the BTC if he provides a valid signature from a specific public key. This condition only applies to the immediate transaction.
* Recursive covenant: Alice sends Bob 1 BTC with the condition that Bob can only spend the BTC if he provides a valid signature from a specific public key, and that any future transactions spending the BTC must also provide a valid signature from the same public key. This condition applies to all future transactions that spend the BTC.

Recursive covenants can be more powerful and flexible than regular covenants. They represent a significant step forward in the programmability and flexibility of Bitcoin transactions, potentially opening up a wide array of new applications and use cases. For instance, they allow implementing more complex transaction logic required for interoperability with sidechains or other Layer 2 solutions.

![|700x700](upload://pEeD4n5pJ5LPLiz9jtgIqvHO0H4.jpeg)

# Bitcoin Smart Contracts with State

In Bitcoin’s UTXO model, smart contracts are inherently one-off and stateless, as the UTXO containing the contract is consumed and destroyed when spent. Recursive covenants introduce a form of state that can be maintained and updated across multiple transactions. When a transaction spends a UTXO (Unspent Transaction Output) containing a stateful contract, the state of the contract is updated, and the new state is stored in the output of the spending transaction, all enforced by recursive covenants. Unlike traditional Bitcoin transactions, which are stateless and immutable once confirmed, stateful smart contracts enable the tracking and modification of state over time, akin to smart contracts on other blockchain platforms like Ethereum.

Let us illustrate how it works with a simple counter contract. This basic contract maintains a single state: how many times it has been called since deployment.

![|700x176](upload://votKflVm6UMEwUk0Fzww4DNjJEJ.png)

Counter in a chain of transactions with state 0, 1, and 2

The state is stored in an adjacent output next to the output containing the contract itself. More specifically, it is in an OP_RETURN output.
```
OP_RETURN OP_PUSHBYTES [counter value]
```

The Counter contract below resides in a Taproot output. There are two tricks worth highlighting:

1. We choose to store state in a separate output, instead of the same taproot output. This allows us to avoid tweaking the taproot address in the contract, since the address remains unchanged. Tweaking involves heavy elliptic curve arithmetic, which necessitates either excessively long script or new opcode like [OP_TAPLEAF_UPDATE_VERIFY](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2021-September/019419.html)/OP_TLUV.
2. We use covenant to get the txid of the previous transaction, which in turns allows us to parse the raw transaction and access its outputs.

```
export class Counter extends SmartContract {

    @prop()
    static readonly ZEROSAT: ByteString = toByteString('0000000000000000')

    constructor() {
        super(...arguments)
    }

    @method()
    public increment(
        shPreimage: SHPreimage,
        prevTxVer: ByteString,
        prevTxLocktime: ByteString,
        prevTxInputContract: ByteString, // First input includes input count prefix...
        prevTxInputFee: ByteString,
        feePrevout: ByteString,
        contractOutputSPK: ByteString, // contract output scriptPubKey
        contractOutputAmount: ByteString, // contract output amount
        contractOutputAmountNew: ByteString, // updated contract output amount
        count: bigint
    ) {
        // Check sighash preimage.
        const s = SigHashUtils.checkSHPreimage(shPreimage)
        assert(this.checkSig(s, SigHashUtils.Gx))

        // Construct prev tx.
        const opreturnScript = OpCode.OP_RETURN + Counter.writeCount(int2ByteString(count))
        const opreturnOutput =
            Counter.ZEROSAT +
            int2ByteString(len(opreturnScript)) +
            opreturnScript
        const prevTxId = hash256(
            prevTxVer +
            prevTxInputContract +
            prevTxInputFee +
            toByteString('02') +
            contractOutputAmount +
            contractOutputSPK +
            opreturnOutput +
            prevTxLocktime
        )

        // Validate prev tx.
        const hashPrevouts = sha256(
            prevTxId + toByteString('00000000') + feePrevout
        )
        assert(hashPrevouts == shPreimage.hashPrevouts, 'hashPrevouts mismatch')
        assert(
            shPreimage.inputNumber == toByteString('00000000'), 'contract must be called via first input'
        )

        // Increment counter.
        const newCount = count + 1n
        const opreturnScriptNew = OpCode.OP_RETURN + Counter.writeCount(int2ByteString(newCount))
        const opreturnOutputNew =
            Counter.ZEROSAT +
            int2ByteString(len(opreturnScriptNew)) +
            opreturnScriptNew

        // Enforce outputs.
        const hashOutputs = sha256(
            // recurse: same scriptPubKey
            contractOutputAmountNew + contractOutputSPK + opreturnOutputNew
        )
        assert(hashOutputs == shPreimage.hashOutputs, 'hashOutputs mismatch')
    }
}
```

The contract ensure the spending transactions satisfy the following properties:

* recursive covenant: the address of the first input is identical to the address of the first output
* state transition: the second output (i.e., the state output) of the current transaction must have a counter value exactly one larger than that in the previous transaction’s second output
* having one input and one two outputs

A single run results in the following transactions:

* **Deploy Transaction ID with initial state 0**:

[
## Bitcoin Signet Transaction: 142782e6dd8ffcf06554b8222637c237a65f47aab27c373da3ddd7b46cd8428c
### Explore the full Bitcoin ecosystem with The Mempool Open Source Project®. See the real-time status of your…
mempool.space
](https://mempool.space/signet/tx/142782e6dd8ffcf06554b8222637c237a65f47aab27c373da3ddd7b46cd8428c?source=post_page-----6a3127a24af4--------------------------------)

* **Transaction ID with state 1 after incremented once**:

[
##### Bitcoin Signet Transaction: 1d1112a7ba7d3dde969006ab3984564b67b5060d1d323d2d2bf963069b600f20
##### Explore the full Bitcoin ecosystem with The Mempool Open Source Project®. See the real-time status of your…
mempool.space
](https://mempool.space/signet/tx/1d1112a7ba7d3dde969006ab3984564b67b5060d1d323d2d2bf963069b600f20?source=post_page-----6a3127a24af4--------------------------------)

* **Transaction ID with state 2 after incremented twice**:

[
##### Bitcoin Signet Transaction: 01a5ed59ec9497ec82d80dc2ba41025342d66a25eec6b4046ec5b8c4964295d1
##### Explore the full Bitcoin ecosystem with The Mempool Open Source Project®. See the real-time status of your…
mempool.space
](https://mempool.space/signet/tx/01a5ed59ec9497ec82d80dc2ba41025342d66a25eec6b4046ec5b8c4964295d1?source=post_page-----6a3127a24af4--------------------------------)

Full code can be found at https://github.com/sCrypt-Inc/scrypt-btc-counter.

More sophisticated state machines can be implemented similarly, where state transition is enforced entirely on chain. There can be alternative places and ways to store and encode states, different from the counter contract.

-------------------------

