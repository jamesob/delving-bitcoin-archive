# Bitcoin PIPEs: Introducting Covenants without Soft Forks

MishaKomarov | 2024-10-07 20:23:33 UTC | #1

# **Bitcoin PIPEs: Covenants on Bitcoin Without Soft Fork**

## Introduction

This post introduces Bitcoin PIPEs (covenants via Polynomial Inner Product Encryption (FH-MIPE)) as a practical approach to implementing covenants on for Bitcoin without requiring a soft fork.

Covenants provide a way to enhance Bitcoin’s capabilities by allowing users to set specific rules on how their coins can be spent in the future.

With covenants, Bitcoin can unlock a range of advanced features:

- **Advanced spending rules**: Users could restrict coins to be spent only at specific addresses or after certain conditions, such as a delay in time.
- **New use cases**: Covenants, for example, enable the native verification of Zero-Knowledge Proofs (ZKPs), native tokens with complex behaviors, and even restaking mechanisms to pool Bitcoin and help secure other networks.

Covenants can make Bitcoin much more versatile but require upgrades to the network, making them increasingly unlikely.

**Polynomial Inner Product Encryption (PIPE (FH-MIPE))** is a cryptographic technique that enables developers to implement advanced covenants without altering Bitcoin’s core protocol. It is possible to emulate missing opcodes (e.g. `OP_CAT` or `OP_CTV`) via establishing a PIPE for a particular opcode desired, avoiding the need for a soft fork or the use of trusted solutions.

With PIPEs, Bitcoin can support:

- **Complex transaction conditions**: Enabling multiple steps or conditions before transaction is processed/funds are released.
- **Efficient batch transactions**: Predefining transaction templates to save space and reduce fees.
- **Scalable cryptographic proofs**: Facilitating Zero-Knowledge Proofs, which improve Bitcoin's scalability and performance.

Bitcoin PIPEs offer a way to introduce powerful new features to Bitcoin without the risks of a soft fork or changing its core protocol. We hope to open the door to a more advanced and functional era for Bitcoin.

## Preliminaries

Bitcoin’s scripting language is known for its simplicity and security. However, this simplicity comes with limitations, particularly when it comes to handling complex transaction types or advanced cryptographic operations. 

To address these limitations, covenants have been proposed as a way to extend Bitcoin's scripting capabilities. Covenants allow users to impose specific conditions on how bitcoins can be spent in the future, offering greater flexibility and control.

Covenants enhance Bitcoin's functionality by enabling more complex transaction rules, allowing for use cases that are currently impossible to implement with the existing scripting language. 

Covenants can be applied to various scenarios, both simple and complex, to enhance security, privacy, and functionality in the Bitcoin ecosystem.

- **Advanced Transaction Structuring with `OP_CAT`:**  `OP_CAT`  allows for the concatenation of two or more elements on the stack, enabling more complex transaction structures. For example, `OP_CAT` could be used to combine a user’s address with a unique identifier, ensuring that the transaction is only valid if the combined data matches a specific pattern.
- **Batch Payments and Congestion Control with `OP_CTV`:** The CheckTemplateVerify (`OP_CTV`) opcode allows users to predefine a specific set of transaction outputs, ensuring that a transaction can only be executed if it matches this predefined template. This reduces the need for multiple individual transactions, lowering fees and reducing network congestion.
- **Restaking Mechanisms:** Covenants could be used to enable restaking, where users can lock up their bitcoins to participate in staking-like mechanisms without needing to move their funds off-chain. For example, a covenant could be designed to define a slashing condition for a protocol - a withdrawal can be enabled only in case the covenant is not satisfied.
- **ZKP Verification (zkRollup Support):** Zero-Knowledge Rollups (zkRollups) are a scaling solution that aggregates many computations into a single proof, which is then submitted to the blockchain. Covenants could be designed to enforce that only transactions within a zkRollup are valid, ensuring that large numbers of transactions can be processed off-chain while maintaining security on-chain.

## **Bitcoin PIPE: Covenants via Polynomial Inner Product Encryption**

**We present Polynomial Inner Product Encryption (PIPE) as an efficient solution to unlocking the benefits of covenants without requiring any changes to the network’s consensus rules.**

One of the challenges in enhancing Bitcoin’s scripting capabilities is the absence of certain opcodes that would enable more complex and powerful transaction logic. Bitcoin PIPEs (Polynomial Inner Product Encryption) emulates these missing opcodes without requiring changes to Bitcoin’s core protocol, allowing developers to create sophisticated covenants without the need for soft forks or increased trust assumptions.

### What is Bitcoin PIPE?

Polynomial Inner Product Encryption (PIPE) is a FH-MIPE (full-hiding multi-input inner product encryption) scheme that enables the creation of predicates - logical statements that can sign Bitcoin transactions using the encrypted signing key in case certain logic is satisfied without revealing the key itself.

By embedding IPA-based proof system verifier inside of FH-MIPE inner product computation process, it is, for example, possible to enforce the execution of complex logic and emulate the behavior of missing opcodes, such as `OP_CAT` (which concatenates elements on the stack) or `OP_CTV` (which predefines specific transaction templates).

Bitcoin PIPE allows transactions to be signed with Schnorr signatures (Taproot signatures), without the need for protocol upgrades or new opcodes in case the verification was completed successfully.

Besides emulating opcodes, Bitcoin PIPEs can be defined to be much more complicated than trivial opcode’s logic:

- **Application-Specific Covenants**: Through defining a PIPE, developers can craft covenants tailored to specific applications, like enforcing cryptographic proofs (e.g., zero-knowledge proofs) within transactions.
- **Enhancing Bitcoin's Programmability**: A key advantage of Bitcoin PIPEs is that it enhances the Bitcoin programmability without the need for protocol changes or additional trust assumptions. This means that developers can introduce more sophisticated logic and conditions into their transactions without waiting for consensus on new opcodes or the use of less secure solutions (sidechains).

### Bitcoin PIPEs Architecture

Bitcoin PIPE acts as an off-chain FH-MIPE-based processor of an encrypted transaction signing key, which yields a signature in case the predicate logic is satisfied (e.g. a zk-proof is verified or a concatenation is done correctly).

![Image 1: Usage Workflow](https://prod-files-secure.s3.us-west-2.amazonaws.com/38e9ffa2-a958-42ea-b848-2fb5407a62b8/1e7bc1ed-d468-4f42-b723-1c2b212b3f89/BItcoin_PIPEs.jpg)

Image 1: Usage Workflow

### Bitcoin PIPE Setup

1. Define a functionality desired to be executed before the transaction happens.
2. Create a zero-knowledge proof (ZKP) to prove execution of computations desired.
3. Perform a DKG/Encryption phase for the address which a PIPE (Covenant) is supposed to restrict.
4. Submit PIPE ciphertext as a Taproot transaction into Bitcoin (inscription-alike mechanics).

### Bitcoin PIPE Usage

1. Retrieve a ciphertext and a decryption key from a PIPE inscription.
2. Prepare the transaction data for signing.
3. Put the transaction through the PIPE decryption functionality execution.
4. If a transaction doesn’t violate the application logic desired, then a PIPE will produce a signed Taproot transaction.
5. Submit the transaction to Bitcoin.

A PIPE being set up this way (as an example) represents non-custodial storage for Bitcoin which allows withdrawals in case certain application logic conditions are satisfied.

### Security Assumptions

Since existing conceptual frameworks for enhancing Bitcoin’s functionality and enabling complex, programmable transactions rely on trust-minimized (not fully trustless) systems or social consensus for protocol changes, it is important to further minimize trust assumptions for Bitcoin PIPEs. 

Given the need to eliminate trust assumptions as much as possible, Bitcoin PIPEs become effectively trustless after completion of a one-time trusted setup (DKG), which brings trust assumptions to 1-out-of-n for each new covenant creation. In other words, so long as one participant behaves honestly the system is secure. To clarify, Bitcoin PIPEs do not rely on moving assets off-chain (i.e. no required use of bridging or sidechains), custodians (or custodian chains), or multi-signatures.

## **Conclusion**

Bitcoin covenants, made possible by PIPEs, represent a significant leap forward in Bitcoin's evolution. This provides a method to introduce complex, secure, and flexible transaction rules without the need for consensus on new opcodes or soft forks. This approach not only preserves Bitcoin's foundational principles, but also significantly broadens its functional horizon, making it ready for more advanced financial and contractual applications while maintaining its core security and simplicity.

-------------------------

