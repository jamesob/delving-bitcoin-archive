# OP_CAT Use Cases series 2

sCrypt-ts | 2024-06-25 13:36:48 UTC | #1

# Bitcoin OP_CAT Use Cases Series #2: Merkle Trees

Following our [series #1](https://scryptplatform.medium.com/trustless-ordinal-sales-using-op-cat-enabled-covenants-on-bitcoin-0318052f02b2), we demonstrate how to construct and verify Merkle trees using OP_CAT.

![|700x400](upload://rxcgOJLu6Q7xijDjZsjxrBhnAJ.jpeg)

In Bitcoin, Merkle trees are utilized as the data structure for verifying data, synchronization, and effectively linking the blockchain’s transactions and blocks together. The OP_CAT opcode, which allows for the concatenation of two stack variables, can be used with SHA256 hashes of public keys to streamline the Merkle tree verification process within Bitcoin Script. OP_CAT uniquely allows for the creation and opening of entries in Merkle trees, as the fundamental operation for building and verifying Merkle trees involves concatenating two values and then hashing them.

There are numerous applications for Merkle trees. We list a few prominent examples below.

## Merkle proof

A Merkle proof is a cryptographic method used to verify that a specific transaction is included in a Merkle tree without needing to download the entire blockchain. This is particularly useful for lightweight clients and enhancing the efficiency of data verification.

## Tree signature

A [tree signature](https://scryptplatform.medium.com/tree-signatures-8d03a8dd3077) is a cryptographic method that enhances the security and efficiency of digital signatures using tree structures, particularly Merkle trees. Compared to regular Multisig, this approach is used to generate a more compact and private proof that a message or a set of messages has been signed by a specific key.

## Zero-Knowledge Proof

STARK (Succinct Transparent Arguments of Knowledge) is a type of zero-knowledge proof system. STARKs are designed to allow a prover to demonstrate the validity of a computation to a verifier without revealing any sensitive information about the computation itself. If OP_CAT were to be added to Bitcoin, it could potentially enable the implementation of a [STARK verifier](https://starkware.co/scaling-bitcoin-for-mass-use) in Bitcoin Script, with [work already ongoing](https://github.com/Bitcoin-Wildlife-Sanctuary/bitcoin-circle-stark/). This would allow for secure and private transactions on the Bitcoin network. Compared to pairing-based proof systems such as SNARK, STARK is considered to be [more Bitcoin-friendly](https://hackmd.io/@l2iterative/bitcoin-polyhedra).

# Implementation

The implementation of the merkle tree using sCrypt is trivial. The following code calculates the root hash of a merkle tree, given a leaf and its merkle path, commonly used in verifying a merkle proof.

```
/**
 * According to the given leaf node and merkle path, calculate the hash of the root node of the merkle tree.
*/
@method()
static calcMerkleRoot(
    leaf: Sha256,
    merkleProof: MerkleProof
): Sha256 {
    let root = leaf

    for (let i = 0; i < MERKLE_PROOF_MAX_DEPTH; i++) {
        const node = merkleProof[i]
        if (node.pos != NodePos.Invalid) {
            // s is valid
            root =
                node.pos == NodePos.Left
                    ? Sha256(hash256(node.hash + root))
                    : Sha256(hash256(root + node.hash))
        }
    }

    return root
}
```

Full code is at https://github.com/sCrypt-Inc/scrypt-btc-merkle.

A single run results in the following transactions:

* **Deploying Transaction ID**:

[
## Bitcoin Signet Transaction: c9c421b556458e0be9ec4043e1804d951011047b4cc75c991842b91b11bae006
### Explore the full Bitcoin ecosystem with The Mempool Open Source Project®. See the real-time status of your…
mempool.space
](https://mempool.space/signet/tx/c9c421b556458e0be9ec4043e1804d951011047b4cc75c991842b91b11bae006?source=post_page-----8e7c3f7afe8d--------------------------------)

* **Spending Transaction ID**

[
## Bitcoin Signet Transaction: e9ac5444d7d20a20011f6dcac04419e2c5581e79bf0692ccd2dc4bbb9bd74e28
### Explore the full Bitcoin ecosystem with The Mempool Open Source Project®. See the real-time status of your…
mempool.space
](https://mempool.space/signet/tx/e9ac5444d7d20a20011f6dcac04419e2c5581e79bf0692ccd2dc4bbb9bd74e28?source=post_page-----8e7c3f7afe8d--------------------------------)

[Many OP_CATs](https://mempool.space/signet/tx/e9ac5444d7d20a20011f6dcac04419e2c5581e79bf0692ccd2dc4bbb9bd74e28#vin=0)

## Script versions

There are alternative implementations in bare scripts, like the one below. One significant advantage of using sCrypt for implementing merkle trees is its readability and maintainability. Scripts are often extremely hard to read and work on.

```
OP_TOALTSTACK
OP_CAT
OP_CAT
OP_TOALTSTACK
OP_CAT
OP_TOALTSTACK

<0x8743daaedb34ef07d3296d279003603c45af71018431fd26e4957e772df122cb>
OP_CAT
OP_CAT
OP_HASH256

OP_DEPTH
OP_1SUB
OP_NOT
OP_NOTIF
OP_SWAP
OP_CAT
OP_CAT
OP_HASH256

OP_DEPTH
OP_1SUB
OP_NOT
OP_NOTIF
OP_SWAP
OP_CAT
OP_CAT
OP_HASH256

OP_DEPTH
OP_1SUB
OP_NOT
OP_NOTIF
OP_SWAP
OP_CAT
OP_CAT
OP_HASH256

OP_DEPTH
OP_1SUB
OP_NOT
OP_NOTIF
OP_SWAP
OP_CAT
OP_CAT
OP_HASH256

OP_DEPTH
OP_1SUB
OP_NOT
OP_NOTIF
OP_SWAP
OP_CAT
OP_CAT
OP_HASH256

...
```

Stay tuned for more OP_CAT use cases.

-------------------------

