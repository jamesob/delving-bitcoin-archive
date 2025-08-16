# Confidential Script: Emulate soft forks using stateless TEEs

josh | 2025-08-15 23:25:58 UTC | #1

## TLDR

[confidential-script-lib](https://github.com/joshdoman/confidential-script-lib) is a Rust library for emulating Bitcoin script by converting script path spends to key path spends. Intended for use inside a Trusted Execution Environment (TEE), the library validates unlocking conditions and then authorizes the transaction using a deterministically derived private key.

This approach enables confidential execution of complex script, including opcodes not yet supported by the Bitcoin protocol. The actual on-chain footprint is a minimal key-path spend, and it is compatible with `rust-bitcoinkernel`, or a fork thereof.

## Motivation

This library is a follow-up to the BTC++ hackathon in Austin, where I presented this architecture without any code, which [won](https://devpost.com/software/confidential-script?_gl=1*1csbbzb*_gcl_au*MTY5NTI5ODMwMy4xNzUwOTY3OTI2*_ga*MTQ3NTc5NjQ5OS4xNzUwOTY3OTI2*_ga_0YHJK3Y10M*czE3NTUwMjM0MTMkbzMkZzAkdDE3NTUwMjM0MTMkajYwJGwwJGgw) the prize for "Most Ambitious."

The goal is to allow developers to experiment on mainnet with features not yet supported by the Bitcoin protocol (ex: `OP_CAT`, `OP_CTV`, `OP_CCV`, Simplicity, etc.), in a permissionless manner with minimal trust assumptions. This could provide a useful compromise in the soft fork debate and allow the community to see which upgrades have real world demand.

## What is a TEE?

A Trusted Execution Environment is an isolated system that segregates memory and the CPU from the outside world, keeping secrets stored in memory secure against side-channel and physical attacks and providing secure attestations about the code being run inside and the result of any computation.

A popular TEE is AWS's [Nitro Enclave](https://aws.amazon.com/ec2/nitro/nitro-enclaves/), which is used by [ACINQ](https://aws.amazon.com/ec2/nitro/nitro-enclaves/#ACINQ) to secure funds held in its LSP. AWS Nitro notably integrates with the AWS key management system (KMS), which can facilitate the provisioning of sensitive material. Users must trust AWS as the manufacturer and operator of the machine, but the security guarantees of the system have proven sufficient for many use cases where developers must secure significant funds.

## Overview

The `confidential-script-lib` library operates internally with a two-step process: emulation and signing.

1.  **Emulation**: A transaction is constructed using an input spending a *real* `previous_outpoint` with a witness that is a script-path spend from an *emulated* P2TR `script_pubkey`. The library validates this emulated witness using a `Verifier`, which matches the API of `rust-bitcoinkernel`. If compiled with the `bitcoinkernel` feature, users can use the actual kernel as the default verifier, or they can provide an alternative verifier that enforces a different set of rules (ex: a fork of `rust-bitcoinkernel` that supports Simplicity).

2.  **Signing**: If the transaction is valid, the library uses the provided parent private key and the merkle root of the *emulated* script path spend to derive a child private key, which controls the key-path of the *actual* UTXO being spent. The library then updates the transaction with a key-path spend signed with this child key.

To facilitate offline generation of the real `script_pubkey`, the child key is derived from the parent key using a non-hardened HMAC-SHA512 derivation scheme. This lets users generate addresses using the parent _public_ key, while the parent private key is secured elsewhere.

This library is intended to be run within a TEE, which is securely provisioned with the parent private key. This decouples script execution from on-chain settlement, keeping execution private and enabling new functionality with minimal trust assumptions.

## Failsafe Mechanism: Backup Script Path

To prevent funds from being irrecoverably locked if the TEE becomes unavailable, the library allows for the inclusion of an optional `backup_merkle_root` when creating the actual on-chain address. This backup merkle root defines the alternative spending paths that are available independently of the TEE.

A common use case for this feature is to include a timelocked recovery script (e.g., using `OP_CHECKSEQUENCEVERIFY`). If the primary TEE-based execution path becomes unavailable for any reason, the owner can wait for the timelock to expire and then recover the funds using a pre-defined backup script. This provides a crucial failsafe, ensuring that users retain ultimate control over their assets.

## Extensibility for Proposed Soft Forks

This library can be used to emulate proposed upgrades, such as new opcodes like `OP_CAT` or `OP_CTV` or new scripting languages like Simplicity. It accepts any verifier that adheres to the `rust-bitcoinkernel` API, allowing developers to experiment with new functionality by forking the kernel, without waiting for a soft fork to gain adoption on mainnet.

## Recommended Setup

This library is intended to be used within a Nitro Enclave, integrated with KMS such that any AWS account can provision an identical enclave with the same master private key. For maximum security, **the KMS key should be created with an irrevocable policy that makes it un-deletable and accessible only to enclaves running a specific, verified image**. Crucially, it should allow requests from any AWS account that meets this strict criteria, so that usage is permissionless.

To generate the master secret, an enclave should generate a random secret and use `GenerateDataKey` to encrypt it using KMS. To provision a different enclave with the secret, the user should provide the enclave the encrypted encryption key and the encrypted secret. The enclave can then decrypt the encryption key with KMS using `Decrypt` and subequently decrypt the secret.

Finally, the enclave should expose the master public key, so that users can independently derive the on-chain address they should send funds to.

## Usage

### Convert emulated transaction

```rust
/// Verifies an emulated Bitcoin script and signs the corresponding transaction.
///
/// This function performs script verification using bitcoinkernel, verifying an
/// emulated P2TR input. If successful, it derives an XOnlyPublicKey from the
/// parent key and the emulated merkle root, which is then tweaked with an 
/// optional backup merkle root to derive the actual spent UTXO, which is then 
/// key-path signed with `SIGHASH_DEFAULT`.
///
/// If the emulated script-path spend includes a data-carrying annex (begins with 
/// 0x50 followed by 0x00), the annex is included in the key-path spend. 
/// Otherwise, the annex is dropped.
///
/// # Arguments
/// * `verifier` - The verifier to use for script validation
/// * `input_index` - Index of the input to verify and sign (0-based)
/// * `emulated_tx_to` - Serialized transaction to verify and sign
/// * `actual_spent_outputs` - Actual outputs being spent
/// * `aux_rand` - Auxiliary random data for signing
/// * `parent_key` - Parent secret key used to derive child key for signing
/// * `backup_merkle_root` - Optional merkle root for backup script path spending
///
/// # Errors
/// Returns error if verification fails, key derivation fails, or signing fails
pub fn verify_and_sign<V: Verifier>(
    verifier: &V,
    input_index: u32,
    emulated_tx_to: &[u8],
    actual_spent_outputs: &[TxOut],
    aux_rand: &[u8; 32],
    parent_key: SecretKey,
    backup_merkle_root: Option<TapNodeHash>,
) -> Result<Transaction, Error>;
```

### Generate an address

```rust
/// Generates P2TR address from a parent public key and the emulated merkle root,
/// with an optional backup merkle root.
///
/// # Arguments
/// * `parent_key` - The parent public key
/// * `emulated_merkle_root` - The merkle root of the emulated input
/// * `backup_merkle_root` - Optional merkle root for backup script path spending
/// * `network` - The network to generate the address for
///
/// # Errors
/// Returns an error if key derivation fails
fn generate_address(
    parent_key: PublicKey,
    emulated_merkle_root: TapNodeHash,
    backup_merkle_root: Option<TapNodeHash>,
    network: Network,
) -> Result<Address, secp256k1::Error>;
```

## Next Steps

If there is interest in using this library, the next step would be to deploy it inside a TEE with a reproducible build. While users must trust the security guarantees of the TEE and the KMS provider, a proper setup can otherwise ensure that it is impossible to steal funds without a valid spend from an emulated or backup script. By setting up KMS such that any identical enclave can access the key, we can make the process of converting emulated script path spends to valid key path spends nearly permissionless.

*Special thanks to @Alex and Stephen DeLorme for encouraging me to put this idea into code.*

-------------------------

sanket1729 | 2025-08-15 05:09:15 UTC | #2

Really cool. Reminds of something Jeremy Rubin did for CTV. But this is better because of TEEs

Have you deployed this somewhere?

-------------------------

josh | 2025-08-16 02:51:53 UTC | #3

Thanks! No, I haven't yet deployed this in a TEE, but that's definitely the next step that I'm working towards. I'd also welcome attempts by others to use this library in a TEE. There are many possible deployments, and I expect that others may have more experience in that area than myself.

Separately, it would be really valuable to fork `rust-bitcoinkernel` to add support for the soft forks the community is interested in (`OP_CAT`,  `OP_CTV`, `OP_CCV`, Simplicity, etc.). Having all desired functionality in a single kernel would mean that you could deploy a single TEE to support them all, rather than have separate deployments for each.

-------------------------

