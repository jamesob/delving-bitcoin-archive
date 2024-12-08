# Bitcoin OP_CAT Use Cases Series #5: Drivechain

sCrypt-ts | 2024-12-06 08:09:05 UTC | #1

# Bitcoin OP_CAT Use Cases Series #5: Drivechain

## Hashrate Escrows Without BIP300

We have created a smart contract that operates similarly to the hashrate escrow mechanism in Bitcoin’s Drivechain proposal. However, instead of requiring a substantial protocol upgrade like [BIP300](https://en.bitcoin.it/wiki/BIP_0300), it leverages general smart contract functionality to establish a sidechain covenant using OP_CAT.

![|700x400](upload://zcDYAaSuq2l1Y53EDTb6JyMTZoi.jpeg)

# Drivechain

Drivechain in Bitcoin are a mechanism for implementing **sidechains**, which are independent blockchains pegged to Bitcoin. They allow users to transfer BTC between the Bitcoin mainchain and sidechains, enabling new features or experimental technologies without altering the mainchain. Drivechain use a two-way peg system and rely on Bitcoin miners to enforce transfers.

**Peg-In: Transferring to a Sidechain**

* Users initiate a transaction that locks their bitcoins in a special output on the main Bitcoin chain.
* This action generates an equivalent amount of tokens on the chosen sidechain, permitting users to transact under the sidechain’s distinct rules and features.

**Peg-Out: Returning to the Mainchain**

* To move assets back, users execute a transaction on the sidechain that effectively burns the sidechain tokens.
* A corresponding withdrawal request is then submitted to the mainchain, to be voted on by miners as detailed below.

# BIP300: Hashrate Escrow

The Drivechain proposal for Bitcoin is encapsulated in two Bitcoin Improvement Proposals: [BIP300](https://en.bitcoin.it/wiki/BIP_0300) for Hashrate Escrows and [BIP301](https://en.bitcoin.it/wiki/BIP_0301) for Blind Merged Mining. We focus on the former since involves opcode change.

In the Drivechain proposal BIP300, **miner voting** is a critical mechanism that governs the approval of withdrawals (peg-outs) from the sidechain back to the main Bitcoin blockchain. Transactions are not signed using cryptographic keys. Instead, they are “signed/voted” by the collective hashpower over time: hashrate-secured escrow. This process functions like a large multisignature arrangement, requiring something akin to 1500 out of 3000 “signatures,” with each mined block representing one signature.

Here’s how miner voting operates in our hashrate escrow contract:

## **1. Withdrawal Request Creation**

A user on the sidechain burns their sidechain tokens to initiate a withdrawal request to the main Bitcoin chain.

## **2. Miner Voting Period**

* The withdrawal request is submitted to Bitcoin miners, who vote on its validity.
* This voting process typically spans **3 to 6 months**, allowing ample time for miners to evaluate the request.

## **3. Voting Process**

* Miners include a signal in the blocks they mine, either **approving** or **rejecting** the withdrawal request.
* Signals are embedded in block headers or auxiliary data fields, making them visible on the Bitcoin blockchain.

## **4. Consensus Threshold**

* For a withdrawal to be approved, a majority of miners (usually defined as a percentage of the total hashrate, like 51%) must vote in favor of the request during the voting period.
* If the required threshold is not met, the withdrawal is denied.

# Overview

Our hashrate escrow of the drivechain implementation features:

* **Operator Control**: Using m-of-n signatures from trusted operators.
* **Miner Voting**: Decentralized consensus for withdrawals.
* **Dynamic State Updates**: The smart contract contains state data that contains values such as vote count and timestamps for voting periods.

The contract includes four primary methods: `lock`, `initWithdrawal`, `vote`, and `finishWithdrawal`.

**Operator-Controlled State Management**

* A group of operators (m-of-n configuration) manage the state transitions.
* The operators are responsible for initializing withdrawal proposals.
* Their actions are validated through a multisig, ensuring no single party can unilaterally control the funds.

**Miner Voting**

* Miners vote on withdrawal proposals via coinbase transactions.
* Voting is based on a two-thirds majority rule, ensuring wide miner participation and decentralized decision-making.

**Dynamic Contract State**

* The smart contract contains state data that contains values such as vote count and timestamps for voting periods.
* Each state transition is hashed and stored in an unspendable OP_RETURN output. Transaction introspection allows us to read this value in the transaction that follows.

The contract operates through the following phases:

1. **Lock**: Funds are bridged into the drive chain.
2. **InitWithdrawal**: Operators propose withdrawals.
3. **Vote**: Miners vote to approve or reject proposals.
4. **FinishWithdrawal**: Approved withdrawals are executed or state resets for unapproved proposals.

![|700x230](upload://uKM4zYzYYlpFYIml5rsCJOScyzj.png)

Transaction diagram depicting a call to the lock() method

![|700x353](upload://m5fFX8Hnpnx7ts0pRzYq1Y2KiVZ.png)

Transaction diagram depicting a call to the vote() method

# Implementation

## State Representation

The contract state includes:

* `startPeriod`: Block height of the last update.
* `voteCnt`: Miner votes for the current withdrawal.
* `payoutAmt`: BTC amount to withdraw.
* `payoutSPK`: ScriptPubKey of the withdrawal output.

These values are hashed and enforced to be stored within an unspendable OP_RETURN output by the contract, using [recursive covenants](https://scryptplatform.medium.com/bitcoin-op-cat-use-cases-series-4-recursive-covenants-6a3127a24af4).

## Locking Extra Funds

The `lock` method validates and bridges BTC to the covenant. It ensures the bridged amount exceeds 10,000 satoshis and verifies state consistency.

```ts
@method()
public lock(
    shPreimage: SHPreimage,
    prevTx: DriveChainTx,
    amtBridged: bigint,
    spentAmountsSuffix: ByteString,
    feePrevout: ByteString
) {
    this.checkContractContext(shPreimage, prevTx, feePrevout)

    // Check passed spent amounts info is valid.
    assert(
        shPreimage.hashSpentAmounts ==
        sha256(
            DriveChain.padAmt(prevTx.contractOutputAmount) +
            DriveChain.padAmt(amtBridged) +
            spentAmountsSuffix
        )
    )

    // Check bridged amount is sufficient.
    assert(amtBridged > 10000n)

    // Ensure state transition is reflected in outputs.
    const hashOutputs = sha256(
        DriveChain.padAmt(prevTx.contractOutputAmount + amtBridged) +
        prevTx.contractOutputSPK +
        DriveChain.getStateOut(prevTx.stateHash)
    )
    assert(hashOutputs == shPreimage.hashOutputs)
}
```

## Initializing a Withdrawal

Operators propose withdrawals by providing signatures and specifying payout details. The contract validates these signatures, ensures sufficient time has passed since the last proposal, and updates the state.

```ts
@method()
public initWithdrawal(
    shPreimage: SHPreimage,
    prevTx: DriveChainTx,
    operatorSigs: FixedArray<Sig, 3>,
    operatorPubKeys: FixedArray<PubKey, 5>,
    currentState: DriveChainState,
    nLockTimeInt: bigint,
    payoutAmt: bigint,
    payoutSPK: ByteString,
    feePrevout: ByteString
) {
    this.checkContractContext(shPreimage, prevTx, feePrevout);

    // Verify the provided state matches the stored hash
    assert(DriveChain.getStateHash(currentState) == prevTx.stateHash);

    // Validate operator signatures and keys
    assert(this.checkMultiSig(operatorSigs, operatorPubKeys));

    // Ensure withdrawal period requirements are met
    assert(nLockTimeInt >= currentState.startPeriod + 2088n);

    // Update state with new payout details
    const newState: DriveChainState = {
        startPeriod: nLockTimeInt,
        voteCnt: 0n,
        payoutAmt: payoutAmt,
        payoutSPK: payoutSPK,
    };

    // Ensure state transition is reflected in outputs
    const hashOutputs = sha256(
        DriveChain.padAmt(prevTx.contractOutputAmount) +
        prevTx.contractOutputSPK +
        DriveChain.getStateOut(DriveChain.getStateHash(newState))
    );
    assert(hashOutputs == shPreimage.hashOutputs);
}
```

## Voting on Withdrawal

Miners vote by using a coinbase transaction. Their votes affect the `voteCnt` positively or negatively, and the contract enforces single-use votes.

```ts
@method()
public vote(
    shPreimage: SHPreimage,
    prevTx: DriveChainTx,
    coinbaseTx: CoinbaseTx,
    currentState: DriveChainState,
    agree: boolean,
    feePrevout: ByteString,
    changeOut: ByteString
) {
    this.checkContractContextVote(
        shPreimage,
        prevTx,
        coinbaseTx,
        feePrevout
    )

    // Check passed state.
    assert(DriveChain.getStateHash(currentState) == prevTx.stateHash)

    // Check block height in coinbase tx is greater than start period.
    assert(coinbaseTx.blockHeight > currentState.startPeriod)

    // Adjust vote count.
    // Implements 66% agree-voting threshold.
    if (agree) {
        currentState.voteCnt += 1n
    } else {
        currentState.voteCnt -= 2n
    }

    // Ensure state transition is reflected in outputs.
    const hashOutputs = sha256(
        DriveChain.padAmt(prevTx.contractOutputAmount) +
        prevTx.contractOutputSPK +
        DriveChain.getStateOut(DriveChain.getStateHash(currentState)) +
        changeOut
    )
    assert(hashOutputs == shPreimage.hashOutputs)
}
```

## Finalizing a Withdrawal

Once the voting period ends, withdrawals are finalized if the `voteCnt` reaches the threshold. The payout amount is sent, and the state is reset for future proposals.

```ts
@method()
public finishWithdrawal(
    shPreimage: SHPreimage,
    prevTx: DriveChainTx,
    nLockTimeInt: bigint,
    currentState: DriveChainState,
    feePrevout: ByteString
) {
    this.checkContractContext(shPreimage, prevTx, feePrevout)
    
    // Check passed state.
    assert(DriveChain.getStateHash(currentState) == prevTx.stateHash)
    // Check passed nLockTime int value.
    assert(shPreimage.nLockTime == DriveChain.padTime(nLockTimeInt))
    // This method can only be called at least 2016 blocks after startPeriod.
    assert(nLockTimeInt >= currentState.startPeriod + 2016n)
    // Check votes were made.
    assert(currentState.voteCnt > 0n)
    // Construct payout output.
    const payoutOut =
        DriveChain.padAmt(currentState.payoutAmt) + currentState.payoutSPK
    // Update state; reset vote count, payout amt and set new start preiod.
    const newState: DriveChainState = {
        startPeriod: nLockTimeInt,
        voteCnt: 0n,
        payoutAmt: 0n,
        payoutSPK: toByteString(''),
    }
    // Ensure state transition is reflected in outputs.
    const hashOutputs = sha256(
        DriveChain.padAmt(prevTx.contractOutputAmount) +
        prevTx.contractOutputSPK +
        DriveChain.getStateOut(DriveChain.getStateHash(newState)) +
        payoutOut
    )
    assert(hashOutputs == shPreimage.hashOutputs)
}
```
The full code of the smart contract can be found [on GitHub](https://github.com/sCrypt-Inc/cat-contracts/blob/master/src/contracts/driveChain.ts).

# Acknowledgement

Our implementation draws inspiration from [the SHA-gate contract](https://github.com/mr-zwets/upgraded-SHA-gate) designed for Bitcoin Cash. Our work adapts the mechanics of the SHA-gate contract to BTC with OP_CAT re-enabled.

-------------------------

1440000bytes | 2024-12-08 16:55:22 UTC | #2

Thank you for writing this post. I think this is the best use case for OP_CAT.

-------------------------

