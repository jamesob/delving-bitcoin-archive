# OP_CAT Use cases series 3 : vaults

sCrypt-ts | 2024-07-01 00:23:02 UTC | #1

# Bitcoin OP_CAT Use Cases Series #3: Vaults
## Delayed spending

Following our [series #1](https://delvingbitcoin.org/t/bitcoin-op-cat-use-cases-series-1-covenants/990) and [#2](https://delvingbitcoin.org/t/op-cat-use-cases-series-2/988), we demonstrate how to construct non-custodial vaults, to provide enhanced security for stored bitcoins. They are typically used to protect against theft by requiring a time delay to access the funds. We can think of vault smart contracts as special accounts whose keys can be neutralized if they fall into the hands of attackers. Vaults are Bitcoin’s decentralized version of calling your bank to report a stolen credit card, rendering the attacker’s transactions null and void. This disincentives key theft in the first place, as attackers know they cannot get away with theft.

![|700x700](upload://oXlLTsW6lxrFGbA2dlan1TDbGrc.jpeg)

# How Vaults Work

Funds locked in a vault contract can be accessed using either of two keys: a *vault* key, which is intended to be kept online and stored in a hot wallet, and a *recovery* key, which is kept securely offline in a cold wallet and used only for recovery. Typically, the *vault* key is used to create transactions that spend coins from the vault. Regardless of which key is used, any funds spent from the vault must pass through a time lock that holds the funds for a fixed period, such as 24 hours. This mechanism ensures that if a malicious actor obtains the *vault* key, they must broadcast a time-locked transaction on the blockchain before gaining access to the funds. This gives the vault owner a 24-hour window to detect the unauthorized movement of their funds and take action. During the time lock period, the contract allows the funds to be redirected to another address using the *recovery* key.

To spend bitcoins from a vault, two sequential steps are required:

1. Issue a withdrawal request to move coins out of the vault through a transaction known as an unvault.
2. Wait for a predefined period (called the unvaulting period), such as 24 hours, after the first transaction is mined before moving the coins out in a subsequent transaction.

The first transaction indicates an attempt to transfer the coins and gives the owner a chance to block the second transaction that would complete the transfer. Step 1 is similar to transferring money from a savings account to a checking account before spending it, while step 2 provides a 24-hour window to revert an unauthorized payment made from the checking account.

# Implementation

The implementation of the vault mechanism using sCrypt involves three smart contracts: Trigger, Complete, and Cancel. Each is a leaf in the vault taproot as shown below.

![|700x199](upload://hJoIAOEliHMYwCMNPqz8EKjc4Ef.png)

```

export class VaultTriggerWithdrawal extends SmartContract {
    @method()
    public trigger(
        shPreimage: SHPreimage,
        sig: Sig,
        vaultSPK: ByteString,
        feeSPK: ByteString,
        vaultAmt: ByteString,
        feeAmt: ByteString,
        targetSPK: ByteString
    ) {
        // Check sig.
        assert(this.checkSig(sig, this.withdrawalPubKey))

        // Check sighash preimage.
        const s = SigHashUtils.checkSHPreimage(shPreimage)
        assert(this.checkSig(s, SigHashUtils.Gx))

        // Enforce spent scripts.
        const hashSpentScripts = sha256(vaultSPK + feeSPK)
        assert(hashSpentScripts == shPreimage.hashSpentScripts, 'hashSpentScripts mismatch')

        // Enforce spent amounts.
        const hashSpentAmounts = sha256(vaultAmt + feeAmt)
        assert(hashSpentAmounts == shPreimage.hashSpentAmounts, 'hashSpentAmounts mismatch')

        // Enforce outputs.
        const dust = toByteString('2202000000000000')
        const hashOutputs = sha256(
            vaultAmt + vaultSPK +
            dust + targetSPK
        )
        assert(hashOutputs == shPreimage.hashOutputs, 'hashOutputs mismatch')
    }

    // Default taproot key spend must be disabled!
}
```

 we validate the trigger transactions

* contain two inputs and two outputs.
* the amount in the first input matches the amount in the first output.
* the address of the first input is identical to the address of the first output.

```
export class VaultCompleteWithdrawal extends SmartContract {
    @method()
    public complete(
        shPreimage: SHPreimage,
        prevTxVer: ByteString,
        prevTxLocktime: ByteString,
        prevTxInputContract: ByteString, // First input chunk should also include length prefix...
        prevTxInputFee: ByteString,
        vaultSPK: ByteString,
        vaultAmt: ByteString,
        targetSPK: ByteString,
        feePrevout: ByteString
    ) {
        this.csv(this.sequenceVal)

        // Check sighash preimage.
        const s = SigHashUtils.checkSHPreimage(shPreimage)
        assert(this.checkSig(s, SigHashUtils.Gx))

        // Construct prev tx.
        const dust = toByteString('2202000000000000')
        const prevTxId = hash256(
            prevTxVer +
            prevTxInputContract +
            prevTxInputFee +
            toByteString('02') + vaultAmt + vaultSPK + dust + targetSPK +
            prevTxLocktime
        )

        // Enforce prevouts.
        const hashPrevouts = sha256(
            prevTxId + toByteString('00000000') +
            feePrevout
        )
        assert(hashPrevouts == shPreimage.hashPrevouts, 'hashPrevouts mismatch')

        // Enforce outputs
        const hashOutputs = sha256(
            vaultAmt + targetSPK
        )
        assert(hashOutputs == shPreimage.hashOutputs, 'hashOutputs mismatch')
    }
```

The Complete transactions must satisfy the following properties:

* having two inputs and one output
* the previous transaction is a Trigger transaction and contains the destination address in the second output. We parse the content of the previous transaction in line 20–28
* the amount in the first output of the previous transaction matches the single output amount in the current transaction.

```
export class VaultCancelWithdrawal extends SmartContract {
    @method()
    public trigger(
        shPreimage: SHPreimage,
        sig: Sig,
        vaultSPK: ByteString,
        feeSPK: ByteString,
        vaultAmt: ByteString,
        feeAmt: ByteString
    ) {
        // Check sig.
        assert(this.checkSig(sig, this.cancelPubKey))

        // Check sighash preimage.
        const s = SigHashUtils.checkSHPreimage(shPreimage)
        assert(this.checkSig(s, SigHashUtils.Gx))

        // Enforce spent scripts.
        const hashSpentScripts = sha256(vaultSPK + feeSPK)
        assert(hashSpentScripts == shPreimage.hashSpentScripts, 'hashSpentScripts mismatch')

        // Enforce spent amounts.
        const hashSpentAmounts = sha256(vaultAmt + feeAmt)
        assert(hashSpentAmounts == shPreimage.hashSpentAmounts, 'hashSpentAmounts mismatch')

        // Enforce outputs.
        const hashOutputs = sha256(
            vaultAmt + vaultSPK
        )
        assert(hashOutputs == shPreimage.hashOutputs, 'hashOutputs mismatch')
    }

    // Default taproot key spend must be disabled!
}
```

Cancel transactions must ensure

* there are exactly two inputs and one output
* the amount of the first input matches the amount of the output.

A single run results in the following transactions:

* **Trigger Transaction ID**:

[
## Bitcoin Signet Transaction: 8c87a62ae1f95efc653af384d034846693a11bf380184aaddb91540c5e74950b
### Explore the full Bitcoin ecosystem with The Mempool Open Source Project®. See the real-time status of your…
mempool.space
](https://mempool.space/signet/tx/8c87a62ae1f95efc653af384d034846693a11bf380184aaddb91540c5e74950b)

* **Complete Transaction ID**:

(https://mempool.space/signet/tx/66b314e72c8ab07fb9b35db9e40aeff6e1b2817b89e653d6b50f7a9675d8fd9a)

The full code of our vault implementation can be found [on GitHub](https://github.com/sCrypt-Inc/scrypt-btc-vault). It has been previously [implemented on Bitcoin SV](https://scryptplatform.medium.com/non-custodial-bitcoin-vaults-880c781effa7), which has a similar but different set of opcodes.

## Script versions

There are alternative implementations in bare scripts, like [purrfect_vault](https://github.com/taproot-wizards/purrfect_vault/blob/main/src/vault/script.rs) from Taproot Wizards. One major benefit of using sCrypt for implementing these vaults is its readability and maintainability. Traditional scripts are often difficult to read and modify.

```
OP_TOALTSTACK
        OP_TOALTSTACK
        OP_TOALTSTACK
        OP_2DUP
        OP_TOALTSTACK
        OP_TOALTSTACK
        OP_TOALTSTACK
        OP_TOALTSTACK
        OP_TOALTSTACK
        // start with encoded leaf hash
        OP_CAT
        OP_CAT
        OP_CAT
        OP_CAT
        push_slice (*DUST_AMOUNT)
        OP_FROMALTSTACK
        OP_CAT
        OP_FROMALTSTACK
        OP_FROMALTSTACK
        OP_CAT
        OP_SWAP
        OP_CAT
        OP_SHA256
        OP_SWAP
        OP_CAT
        OP_CAT
        OP_FROMALTSTACK
        OP_FROMALTSTACK
        OP_FROMALTSTACK
        OP_FROMALTSTACK
        OP_SWAP
        OP_TOALTSTACK
        OP_CAT
        OP_SWAP
        OP_TOALTSTACK
        OP_SHA256
        OP_SWAP
        OP_CAT
        OP_FROMALTSTACK
        OP_FROMALTSTACK
        OP_CAT
        OP_SHA256
        OP_SWAP
        OP_CAT
        OP_CAT
        OP_CAT
        OP_CAT
        OP_CAT
        OP_CAT
```

-------------------------

