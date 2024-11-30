# Implementing a Bridge Covenant on OP_CAT-Enabled Bitcoin: A Proof of Concept

sCrypt-ts | 2024-11-30 09:35:11 UTC | #1

![Implementing a Bridge Covenant on OP_CAT-Enabled Bitcoin: A Proof of Concept|1456x819](upload://e0QPBjPFVfGUjhL6N6KKVE5yCXm.png)


# Implementing a Bridge Covenant on OP_CAT-Enabled Bitcoin: A Proof of Concept

In this article, we explore how **sCrypt**, in collaboration with **StarkWare**, has developed a demo bridge covenant on Bitcoin. This proof-of-concept implementation is designed to serve as a foundational framework for a production-grade bridge connecting the Bitcoin blockchain to the Starknet Layer 2 (L2) network.

The bridge introduces a sophisticated mechanism for handling multiple deposit and withdrawal requests. It allows for the batching of these transactions into a single root transaction. This transaction is then integrated into the main bridge covenant, updating its state seamlessly. The state is maintained as a set of accounts, organized within a secure and efficient Merkle tree structure.

Given the inherent complexity of the bridge's covenant scripts, sCrypt has employed its advanced domain-specific language (DSL) to construct the implementation.

## Overview

The bridge consists of a recursive covenant Bitcoin script. Here, “covenant” means that the locking script is capable of enforcing conditions on the spending transaction, and “recursive” means that the rules above are sufficiently powerful to enable persistent logic and state onchain (a requirement for any onchain smart contract).

This script exists in a chain of transactions, each enforcing the structure of the subsequent transaction that unlocks an output from the current one. Each time a new transaction is added to this chain, it represents an update of the bridge’s state. Hence, the tip of this chain holds the current bridge state.

![|1024x201](upload://uDEby9EyIOU9BwGDec8IxuSVgEq.png)

The covenant’s state—specifically, its hash—is stored in an unspendable OP_RETURN output. While we’re not spending this UTXO, its data can be inspected while executing a covenant script. More concretely, the state holds the root hash of the Merkle tree containing account data, as follows:

![|733x550](upload://sYZFoEAGZLdRXEfCuJzIqdSlzeA.png)

The tree holds data about a fixed set of account slots. The leaves contain the hash of the respective account data, which consists of an address and a balance. To signify empty account slots, they are marked with zero bytes.

Each update of the bridge results in a change to the accounts tree. To facilitate such an update, we rely upon Merkle proofs, the verification of which is efficient within Bitcoin script. An update essentially consists of two steps. First, we verify a Merkle proof that proves the inclusion of a particular account’s current state. Then, after calculating the new state of this account, we use the same auxiliary nodes from the aforementioned Merkle proof to derive the new root hash.

![|812x505](upload://hqXLAnxYcxGU7XqYjvE4Ek3wL9v.png)

The update can be either a deposit or a withdrawal. The bridge can perform a batch of these updates within a single transaction.

## Deposits

We aim to enable users to submit deposits or withdrawal requests independently. To achieve this, users create separate transactions that pay to a deposit or withdrawal aggregation covenant, respectively. This covenant aggregates a batch of these requests into a Merkle tree. The root of this tree can be merged into the main bridge covenant, which then processes each deposit or withdrawal.

![|1024x531](upload://qiN3LF58JXshTDXV0lNEgDw05GW.png)

In deposit transactions, besides hashing the deposit data and constructing a Merkle tree, the covenant also ensures that the deposited satoshis locked into the covenant’s outputs accumulate correctly up to the tree’s root. The aggregation covenant makes it so the funds are only spendable by the correct onchain smart contract. (Of course, in a production system, we will also enable a user to cancel her deposit transaction.)

This kind of tree design results from a limitation of how covenant scripts are constructed, not allowing transactions with too many inputs and outputs. A tree structure allows us to scale to potentially arbitrary throughput.

## Withdrawal Requests

The aggregation of withdrawal requests is similar but differs in a few ways. First, we need an authentication method for a user to withdraw funds from their account. This contrasts with deposits, where anyone is allowed to deposit into anyone’s account, similar to Bitcoin addresses in general. Authentication is done at the leaf level of the aggregation tree. The withdrawal request aggregation covenant checks that the address being withdrawn from matches the P2WPKH address of the first input in the leaf transaction.

![|1024x462](upload://yN23dod2H8A3nLwR6k6Mr0T0hXS.png)

This ensures that the owner of the address approves the withdrawal since they signed the transaction requesting it. Another subtle difference compared to deposit aggregation is that we also hash intermediate sum amounts up the tree. This is done because we’ll need this data when expanding the withdrawals, but more on that later.

An astute reader might notice a potential problem with this model of authenticating withdrawal requests. What if the operator decides to cheat and creates a root transaction of an aggregation tree where the tree’s data is fabricated locally with fake withdrawal requests that weren’t authenticated? We need an efficient way to verify that this root transaction originates from valid leaf transactions.

To address this, we perform what is called a “genesis check.” Essentially, we make the aggregation covenant check its previous transaction and the transaction preceding that one — that is, its ancestor transactions. The covenant verifies that these transactions contain the same covenant script and perform the same checks. In this way, we achieve an inductive transaction history check. Because both of the previous transactions performed the same checks as this covenant does, we know that the ancestors of that transaction did the same, all the way back to the leaf (i.e., the genesis transaction).

![|1024x133](upload://lRtT947Qs2gDQ3M3qJwDgaymeCE.png)

Naturally, we perform this validation for both branches of the tree. So each aggregation node transaction checks up to six transactions in total.

## Expansion of Withdrawals

Now let’s move to the last part of our solution: the expansion of the withdrawals. Upon processing a batch of withdrawal requests, the main bridge covenant enforces an output that pays the total amount withdrawn to an expander covenant. We can think of this covenant as essentially performing the inverse of what the withdrawal request aggregation covenant did. It starts with the withdrawal tree’s root and expands it into two branches, each containing the appropriate amount of withdrawn funds that should go to that branch. It continues this process all the way down to the withdrawal tree’s leaves. The leaf transactions are enforced to have a simple payment output that pays the account owner’s address the amount they requested to withdraw.

![|1024x401](upload://1vXxBfT56OfTywXmRNegQxHTFoE.png)

## Implementation

To bring our bridge covenant to life, we’ve implemented four sCrypt smart contracts that handle different aspects of the system. In this section, we’ll provide a high-level overview of each contract.

### DepositAggregator Contract

The DepositAggregator contract aggregates individual deposits into a single Merkle tree, which can then be merged into the main bridge covenant. This aggregation allows for batch deposit processing, reducing the number of transactions that need to be individually processed by the bridge. Additionally, it allows users to independently submit deposits, which will be later picked up by an operator.

```
class DepositAggregator extends SmartContract {
  @prop()
  operator: PubKey

  @prop()
  bridgeSPK: ByteString
  /**
    * Covenant used for the aggregation of deposits.
    *
    * @param operator - Public key of bridge operator.
    * @param bridgeSPK - P2TR script of the bridge state covenant. Includes length prefix!
    */
  constructor(operator: PubKey, bridgeSPK: ByteString) {
      super(...arguments)
      this.operator = operator
      this.bridgeSPK = bridgeSPK
  }
  @method()
  public aggregate(
      shPreimage: SHPreimage,
      isPrevTxLeaf: boolean,
      sigOperator: Sig,
      prevTx0: AggregatorTransaction,
      prevTx1: AggregatorTransaction,
      // Additional parameters...
  ) {
      // Validation steps...
  }
  @method()
  public finalize(
      shPreimage: SHPreimage,
      sigOperator: Sig,
      prevTx: AggregatorTransaction,
      ancestorTx0: AggregatorTransaction,
      ancestorTx1: AggregatorTransaction,
      bridgeTxId: Sha256,
      fundingPrevout: ByteString
  ) {
      // Finalization steps...
  }
}
```

The contract constructor takes in two parameters:

* *operator* : The public key of the bridge operator who is authorized to aggregate deposits.
* *bridgeSPK* : The script public key (SPK) of the main bridge covenant, ensuring that the aggregated deposits are merged correctly.

The core functionality of the *DepositAggregator* is encapsulated in the *aggregate* method. This method performs the following steps:

**Validation of Sighash Preimage and Operator Signature:** Ensures the transaction is authorized by the bridge operator and that the sighash preimage is correctly formatted and belongs to the executing transaction. Read more about sighash preimage validation in [this article](https://medium.com/@scryptplatform/trustless-ordinal-sales-using-op-cat-enabled-covenants-on-bitcoin-0318052f02b2).

```
// Check sighash preimage.
const s = SigHashUtils.checkSHPreimage(shPreimage)
assert(this.checkSig(s, SigHashUtils.Gx))

// Check operator signature.
assert(this.checkSig(sigOperator, this.operator))
```

**Construction and Verification of Previous Transaction IDs** : Checks that the previous transactions aggregated are valid and correctly referenced.

```
// Construct previous transaction ID.
const prevTxId = AggregatorUtils.getTxId(prevTx, false)

// Verify that the transaction unlocks the specified outputs.
const hashPrevouts = AggregatorUtils.getHashPrevouts(
  bridgeTxId,
  prevTxId,
  fundingPrevout
)
assert(hashPrevouts == shPreimage.hashPrevouts)
```

**Merkle Tree Aggregation** : This verifies that the deposit data passed as witness hashes match the state stored in the previous transactions.

```
const hashData0 = DepositAggregator.hashDepositData(depositData0)
const hashData1 = DepositAggregator.hashDepositData(depositData1)

assert(hashData0 == prevTx0.hashData)
assert(hashData1 == prevTx1.hashData)
```

**Amount Verification** : Confirms that the amounts in the previous outputs match the specified deposit amounts, ensuring funds are correctly accounted for in the aggregation.

```
// Check that the prev outputs actually carry the specified amount
// of satoshis. The amount values can also carry aggregated amounts,
// in case we're not aggregating leaves anymore.
assert(
  GeneralUtils.padAmt(depositData0.amount) ==
  prevTx0.outputContractAmt
)
assert(
  GeneralUtils.padAmt(depositData1.amount) ==
  prevTx1.outputContractAmt
)
```

**State Update** : Computes a new hash by concatenating the hashes of the previous transactions and updates the state in the OP_RETURN output.

```
// Concatinate hashes from previous aggregation txns (or leaves)
// and compute new hash. Store this new hash in the state OP_RETURN
// output.
const newHash = hash256(prevTx0.hashData + prevTx1.hashData)
const stateOut = GeneralUtils.getStateOutput(newHash)
```

**Re-entrance Prevention** : Enforces strict output scripts and amounts to prevent unauthorized modifications or double-spending.

```
// Sum up aggregated amounts and construct contract output.
const contractOut = GeneralUtils.getContractOutput(
  depositData0.amount + depositData1.amount,
  prevTx0.outputContractSPK
)

// Recurse. Send to aggregator with updated hash.
const outputs = contractOut + stateOut
assert(
  sha256(outputs) == shPreimage.hashOutputs
)
```

Once the deposits are aggregated, they must be merged into the main bridge covenant. This is handled by the *finalize* method, whose steps include:

* **Validation of Previous Transactions** : Similar to the aggregate method, it verifies the previous transactions to ensure the integrity of the data being merged.
* **Integration with Bridge Covenant** : Checks that the aggregated deposits are correctly merged into the main bridge covenant by referencing the bridge’s transaction ID and script public key.

The full source code of the deposit aggregation contract can be found [on GitHub](https://github.com/Bitcoin-Wildlife-Sanctuary/scrypt-poc-bridge/blob/dad6aae8601d469d788fb1fa5d89ad233258d057/src/contracts/depositAggregator.ts).

### WithdrawalAggregator Contract

The *WithdrawalAggregator* contract is designed to aggregate individual withdrawal requests into a single Merkle tree, similar to how the *DepositAggregator* handles deposits. However, withdrawals require additional authentication to ensure that only the rightful owners can withdraw funds from their accounts.

```
class WithdrawalAggregator extends SmartContract {
  @prop()
  operator: PubKey

  @prop()
  bridgeSPK: ByteString

  /**
    * Covenant used for the aggregation of withdrawal requests.
    *
    * @param operator - Public key of bridge operator.
    * @param bridgeSPK - P2TR script of the bridge state covenant. Includes length prefix!
    */
  constructor(operator: PubKey, bridgeSPK: ByteString) {
      super(...arguments)
      this.operator = operator
      this.bridgeSPK = bridgeSPK
  }

  @method()
  public aggregate(
      shPreimage: SHPreimage,
      isPrevTxLeaf: boolean,
      sigOperator: Sig,
      prevTx0: AggregatorTransaction,
      prevTx1: AggregatorTransaction,
      // Additional parameters...
  ) {
      // Validation and aggregation logic...
  }

  @method()
  public finalize(
      shPreimage: SHPreimage,
      sigOperator: Sig,
      prevTx: AggregatorTransaction,
      ancestorTx0: AggregatorTransaction,
      ancestorTx1: AggregatorTransaction,
      bridgeTxId: Sha256,
      fundingPrevout: ByteString
  ) {
      // Validation logic...
  }
}
```

The core functionality of the *WithdrawalAggregator* is encapsulated in the *aggregate* method, which performs the following steps:

```
// Check sighash preimage.
const s = SigHashUtils.checkSHPreimage(shPreimage)
assert(this.checkSig(s, SigHashUtils.Gx))

// Check operator signature.
assert(this.checkSig(sigOperator, this.operator))
```

**Construction and Verification of Previous Transaction IDs** : This process verifies that the previous transactions being aggregated are valid and correctly referenced.

```
// Construct previous transaction IDs.
const prevTxId0 = AggregatorUtils.getTxId(prevTx0, isPrevTxLeaf)
const prevTxId1 = AggregatorUtils.getTxId(prevTx1, isPrevTxLeaf)

// Verify that the previous transactions are unlocked by the current transaction.
const hashPrevouts = AggregatorUtils.getHashPrevouts(
  prevTxId0,
  prevTxId1,
  fundingPrevout
)
assert(hashPrevouts == shPreimage.hashPrevouts)
```

**Ownership Proof Verification** : Verifying an ownership proof transaction ensures that only the rightful owner can withdraw funds from an account.

* **Ownership Proof Transaction** : A transaction that proves control of the withdrawal address. The contract checks that the address in the withdrawal request matches the address in the ownership proof transaction.

```
if (isPrevTxLeaf) {
  // Construct ownership proof transaction IDs.
  const ownershipProofTxId0 = WithdrawalAggregator.getOwnershipProofTxId(ownProofTx0)
  const ownershipProofTxId1 = WithdrawalAggregator.getOwnershipProofTxId(ownProofTx1)

  // Check that the leaf transactions unlock the ownership proof transactions.
  assert(ownershipProofTxId0 + toByteString('0000000000ffffffff') == prevTx0.inputContract0)
  assert(ownershipProofTxId1 + toByteString('0000000000ffffffff') == prevTx1.inputContract0)

  // Verify that the withdrawal addresses match the addresses in the ownership proof transactions.
  assert(withdrawalData0.address == ownProofTx0.outputAddrP2WPKH)
  assert(withdrawalData1.address == ownProofTx1.outputAddrP2WPKH)
}
```

**Genesis Check via Ancestor Transactions** : Similar to the *DepositAggregator* , the contract performs an inductive check by verifying ancestor transactions. This ensures the integrity of the transaction history and prevents the operator from injecting unauthorized withdrawal requests.

```
if (!isPrevTxLeaf) {
  // Construct ancestor transaction IDs.
  const ancestorTxId0 = AggregatorUtils.getTxId(ancestorTx0, isAncestorLeaf)
  const ancestorTxId1 = AggregatorUtils.getTxId(ancestorTx1, isAncestorLeaf)
  const ancestorTxId2 = AggregatorUtils.getTxId(ancestorTx2, isAncestorLeaf)
  const ancestorTxId3 = AggregatorUtils.getTxId(ancestorTx3, isAncestorLeaf)

  // Verify that previous transactions unlock the ancestor transactions.
  assert(prevTx0.inputContract0 == ancestorTxId0 + toByteString('0000000000ffffffff'))
  assert(prevTx0.inputContract1 == ancestorTxId1 + toByteString('0000000000ffffffff'))
  assert(prevTx1.inputContract0 == ancestorTxId2 + toByteString('0000000000ffffffff'))
  assert(prevTx1.inputContract1 == ancestorTxId3 + toByteString('0000000000ffffffff'))

  // Ensure that the ancestor transactions have the same contract SPK.
  assert(prevTx0.outputContractSPK == ancestorTx0.outputContractSPK)
  assert(prevTx0.outputContractSPK == ancestorTx1.outputContractSPK)
  assert(prevTx0.outputContractSPK == ancestorTx2.outputContractSPK)
  assert(prevTx0.outputContractSPK == ancestorTx3.outputContractSPK)
}
```

**Amount Verification and Sum Calculation** : This method calculates the total amount to be withdrawn by summing the amounts from the withdrawal requests or previous aggregations.

```
let sumAmt = 0n
if (isPrevTxLeaf) {
  sumAmt = withdrawalData0.amount + withdrawalData1.amount
} else {
  sumAmt = aggregationData0.sumAmt + aggregationData1.sumAmt
}
```

**State Update** : Computes a new hash that includes the hashes of the previous transactions and the sum of the withdrawal amounts. This hash is stored in the OP_RETURN output to update the state.

```
// Create new aggregation data.
const newAggregationData: AggregationData = {
  prevH0: prevTx0.hashData,
  prevH1: prevTx1.hashData,
  sumAmt
}
const newHash = WithdrawalAggregator.hashAggregationData(newAggregationData)
const stateOut = GeneralUtils.getStateOutput(newHash)
```

**Re-entrance Prevention and Output Enforcement** : Ensures that outputs are strictly defined to prevent unauthorized modifications or re-entrance attacks.

```
// Construct contract output with the minimum dust amount.
const contractOut = GeneralUtils.getContractOutput(
  546n,
  prevTx0.outputContractSPK
)

// Ensure outputs match the expected format.
const outputs = contractOut + stateOut
assert(
  sha256(outputs) == shPreimage.hashOutputs,
)
```

Full code of the withdrawal aggregation contract can be found [here](https://github.com/Bitcoin-Wildlife-Sanctuary/scrypt-poc-bridge/blob/dad6aae8601d469d788fb1fa5d89ad233258d057/src/contracts/withdrawalAggregator.ts).

### Bridge Contract

The *Bridge* contract is the core component of our system, acting as the main covenant that maintains the state of the bridge, including the accounts and their balances organized in a Merkle tree. It handles both deposits and withdrawals by integrating with the aggregator contracts we’ve previously discussed.

```
class Bridge extends SmartContract {
  @prop()
  operator: PubKey

  @prop()
  expanderSPK: ByteString

  constructor(
      operator: PubKey,
      expanderSPK: ByteString
  ) {
      super(...arguments)
      this.operator = operator
      this.expanderSPK = expanderSPK
  }

  @method()
  public deposit(
      shPreimage: SHPreimage,
      sigOperator: Sig,
      prevTx: BridgeTransaction,           // Previous bridge update transaction.
      aggregatorTx: AggregatorTransaction, // Root aggregator transaction.
      fundingPrevout: ByteString,
 
      deposits: FixedArray<DepositData, typeof MAX_NODES_AGGREGATED>,
      accounts: FixedArray<AccountData, typeof MAX_NODES_AGGREGATED>,
 
      depositProofs: FixedArray<MerkleProof, typeof MAX_NODES_AGGREGATED>,
      accountProofs: FixedArray<MerkleProof, typeof MAX_NODES_AGGREGATED>
  ) {
      // Method implementation...
  }

  @method()
  public withdrawal(
      shPreimage: SHPreimage,
      sigOperator: Sig,
      prevTx: BridgeTransaction,           // Previous bridge update transaction.
      aggregatorTx: AggregatorTransaction, // Root aggregator transaction.
      fundingPrevout: ByteString,
 
      withdrawals: FixedArray<WithdrawalData, typeof MAX_NODES_AGGREGATED>,
      accounts: FixedArray<AccountData, typeof MAX_NODES_AGGREGATED>,
 
      intermediateSumsArr: FixedArray<IntermediateValues, typeof MAX_NODES_AGGREGATED>,
 
      withdrawalProofs: FixedArray<MerkleProof, typeof MAX_NODES_AGGREGATED>,
      accountProofs: FixedArray<MerkleProof, typeof MAX_NODES_AGGREGATED>
  ) {
      // Method implementation...
  }

}
```

The contract constructor takes two parameters:

* *operator* : The public key of the bridge operator authorized to update the bridge state.
* *expanderSPK* : The script public key (SPK) of the *WithdrawalExpander* contract, used during the withdrawal process.

The *deposit* method handles the processing of aggregated deposit transactions, updating the accounts’ balances accordingly.

```
@method()
public deposit(
  shPreimage: SHPreimage,
  sigOperator: Sig,
  prevTx: BridgeTransaction,           // Previous bridge update transaction.
  aggregatorTx: AggregatorTransaction, // Root aggregator transaction.
  fundingPrevout: ByteString,

  deposits: FixedArray<DepositData, typeof MAX_NODES_AGGREGATED>,
  accounts: FixedArray<AccountData, typeof MAX_NODES_AGGREGATED>,

  depositProofs: FixedArray<MerkleProof, typeof MAX_NODES_AGGREGATED>,
  accountProofs: FixedArray<MerkleProof, typeof MAX_NODES_AGGREGATED>
) {
  // Common validation steps...
  // (Same as in previous contracts: sighash preimage check, operator signature verification, prevouts verification)

  // Ensure this method is called from the first input.
  assert(shPreimage.inputNumber == toByteString('00000000'))

  // Verify that the second input unlocks the correct aggregator script.
  assert(prevTx.depositAggregatorSPK == aggregatorTx.outputContractSPK)

  // Process deposits and update accounts.
  let accountsRootNew: Sha256 = prevTx.accountsRoot
  let totalAmtDeposited = 0n
  for (let i = 0; i < MAX_NODES_AGGREGATED; i++) {
      const deposit = deposits[i]
      if (deposit.address != toByteString('')) {
          accountsRootNew = this.applyDeposit(
              deposits[i],
              depositProofs[i],
              aggregatorTx.hashData,
              accounts[i],
              accountProofs[i],
              accountsRootNew
          )
      }
      totalAmtDeposited += deposit.amount
  }

  // Update the bridge state and outputs.
  // (Compute new state hash, construct contract output, enforce outputs)
}
```

The steps performed by the *deposit* method include:

**Processing Deposits and Updating Accounts** :

* Iterates over the deposits and applies each one to the corresponding account using the *applyDeposit* method.

**Updating the Bridge State and Outputs** :

* Computes the new accounts Merkle root after processing the deposits.
* Creates a new state hash to represent the updated bridge state.
* Constructs the contract output, adding the total deposited amount to the bridge’s balance.
* Enforces that the outputs match the expected format to maintain integrity.

The *withdrawal* method processes aggregated withdrawal transactions, updates the accounts’ balances, and prepares the funds for distribution through the *WithdrawalExpander* .

```
@method()
public withdrawal(
  shPreimage: SHPreimage,
  sigOperator: Sig,
  prevTx: BridgeTransaction,           // Previous bridge update transaction.
  aggregatorTx: AggregatorTransaction, // Root aggregator transaction.
  fundingPrevout: ByteString,

  withdrawals: FixedArray<WithdrawalData, typeof MAX_NODES_AGGREGATED>,
  accounts: FixedArray<AccountData, typeof MAX_NODES_AGGREGATED>,

  intermediateSumsArr: FixedArray<IntermediateValues, typeof MAX_NODES_AGGREGATED>,

  withdrawalProofs: FixedArray<MerkleProof, typeof MAX_NODES_AGGREGATED>,
  accountProofs: FixedArray<MerkleProof, typeof MAX_NODES_AGGREGATED>
) {
  // Common validation steps...
  // (Same as in previous contracts: sighash preimage check, operator signature verification, prevouts verification)

  // Ensure this method is called from the first input.
  assert(shPreimage.inputNumber == toByteString('00000000'))

  // Verify that the second input unlocks the correct aggregator script.
  assert(prevTx.withdrawalAggregatorSPK == aggregatorTx.outputContractSPK)

  // Process withdrawals and update accounts.
  let accountsRootNew: Sha256 = prevTx.accountsRoot
  let totalAmtWithdrawn = 0n
  for (let i = 0; i < MAX_NODES_AGGREGATED; i++) {
      const withdrawal = withdrawals[i]
      if (withdrawal.address != toByteString('')) {
          accountsRootNew = this.applyWithdrawal(
              withdrawal,
              withdrawalProofs[i],
              intermediateSumsArr[i],
              aggregatorTx.hashData,
              accounts[i],
              accountProofs[i],
              accountsRootNew
          )
      }
      totalAmtWithdrawn += withdrawal.amount
  }

  // Update the bridge state and outputs.
  // (Compute new state hash, construct contract output, create expander output, enforce outputs)
}
```

Steps performed by the *withdrawal* method include:

**Processing withdrawal requests and updating accounts:**

* Iterates over the withdrawals and applies each one to the corresponding account using the *applyWithdrawal* method.

**Updating the Bridge State and Outputs** :

* Computes the new accounts Merkle root after processing the withdrawals.
* Creates a new state hash to represent the updated bridge state.
* Constructs the contract output, subtracting the total withdrawn amount from the bridge’s balance.
* Creates an expander output for the *WithdrawalExpander* contract, carrying the total amount withdrawn.
* Enforces that the outputs match the expected format to maintain integrity.

The full source code is accessible [here](https://github.com/Bitcoin-Wildlife-Sanctuary/scrypt-poc-bridge/blob/dad6aae8601d469d788fb1fa5d89ad233258d057/src/contracts/bridge.ts).

### WithdrawalExpander Contract

The *WithdrawalExpander* contract is the final component in our bridge system, responsible for distributing the aggregated withdrawal amounts back to individual users based on their withdrawal requests. It reverses the aggregation process performed by the *WithdrawalAggregator* , expanding the aggregated withdrawal data back into individual user payments.

```
class WithdrawalExpander extends SmartContract {
  @prop()
  operator: PubKey

  constructor(
      operator: PubKey
  ) {
      super(...arguments)
      this.operator = operator
  }

  @method()
  public expand(
      shPreimage: SHPreimage,
      sigOperator: Sig,
      // Additional parameters...
  ) {
      // Expansion logic...
  }

}
```

The core functionality of the *WithdrawalExpander* is encapsulated in the *expand* method. This method takes the aggregated withdrawal data and recursively expands it into individual withdrawal transactions that pay out to users.

**Expanding to Leaves** : If the method expands to leaf nodes (individual withdrawals), it verifies the withdrawal data and constructs outputs that pay directly to the users’ addresses.

```
if (isExpandingLeaves) {
  // If expanding to leaves, verify the withdrawal data.
  if (isExpandingPrevTxFirstOutput) {
      const hashWithdrawalData = WithdrawalAggregator.hashWithdrawalData(withdrawalData0)
      assert(hashWithdrawalData == prevAggregationData.prevH0)
      hashOutputs = sha256(
          WithdrawalExpander.getP2WPKHOut(
              GeneralUtils.padAmt(withdrawalData0.amount),
              withdrawalData0.address
          )
      )
  } else {
      const hashWithdrawalData = WithdrawalAggregator.hashWithdrawalData(withdrawalData1)
      assert(hashWithdrawalData == prevAggregationData.prevH1)
      hashOutputs = sha256(
          WithdrawalExpander.getP2WPKHOut(
              GeneralUtils.padAmt(withdrawalData1.amount),
              withdrawalData1.address
          )
      )
  }
}
```

Further Expansion: If the method is not yet at the leaf level, it continues expanding by splitting the aggregation data into two branches and creating outputs that further expansion transactions will consume.

```
else {
  // Verify current aggregation data matches previous aggregation data.
  const hashCurrentAggregationData = WithdrawalAggregator.hashAggregationData(currentAggregationData)
  if (isPrevTxBridge) {
      assert(hashCurrentAggregationData == prevTxBridge.expanderRoot)
  } else if (isExpandingPrevTxFirstOutput) {
      assert(hashCurrentAggregationData == prevAggregationData.prevH0)
  } else {
      assert(hashCurrentAggregationData == prevAggregationData.prevH1)
  }

  // Prepare outputs for the next level of expansion.
  let outAmt0 = 0n
  let outAmt1 = 0n
  if (isLastAggregationLevel) {
      const hashWithdrawalData0 = WithdrawalAggregator.hashWithdrawalData(withdrawalData0)
      const hashWithdrawalData1 = WithdrawalAggregator.hashWithdrawalData(withdrawalData1)
      assert(hashWithdrawalData0 == currentAggregationData.prevH0)
      assert(hashWithdrawalData1 == currentAggregationData.prevH1)
      outAmt0 = withdrawalData0.amount
      outAmt1 = withdrawalData1.amount
  } else {
      const hashNextAggregationData0 = WithdrawalAggregator.hashAggregationData(nextAggregationData0)
      const hashNextAggregationData1 = WithdrawalAggregator.hashAggregationData(nextAggregationData1)
      assert(hashNextAggregationData0 == currentAggregationData.prevH0)
      assert(hashNextAggregationData1 == currentAggregationData.prevH1)
      outAmt0 = nextAggregationData0.sumAmt
      outAmt1 = nextAggregationData1.sumAmt
  }

  // Construct outputs for further expansion.
  let expanderSPK = prevTxExpander.contractSPK
  if (isPrevTxBridge) {
      expanderSPK = prevTxBridge.expanderSPK
  }

  hashOutputs = sha256(
      GeneralUtils.getContractOutput(outAmt0, expanderSPK) +
      GeneralUtils.getContractOutput(outAmt1, expanderSPK) +
      GeneralUtils.getStateOutput(hashCurrentAggregationData)
  )
}
```

## Conclusion

In this proof-of-concept implementation, we developed a bridge covenant on OP_CAT-enabled Bitcoin using the [sCrypt](https://docs.scrypt.io/) embedded DSL. The bridge leverages recursive covenants and Merkle trees to efficiently batch and process deposit and withdrawal requests while maintaining the integrity and security of user accounts. By designing and implementing four smart contracts — **DepositAggregator** , **WithdrawalAggregator** , **Bridge** , and **WithdrawalExpander** — we provided a method to manage stateful interactions on Bitcoin, facilitating interoperability with Layer 2 networks like Starknet. This work establishes a technical foundation for building production-grade bridges, potentially enhancing scalability and functionality within the Bitcoin ecosystem.

All of the code implementation, along with an end-to-end test, is [available on GitHub](https://github.com/Bitcoin-Wildlife-Sanctuary/scrypt-poc-bridge).

-------------------------

