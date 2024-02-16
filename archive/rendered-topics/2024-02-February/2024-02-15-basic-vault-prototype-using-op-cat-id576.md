# Basic vault prototype using OP_CAT

rijndael | 2024-02-15 22:18:50 UTC | #1

I've been curious to see what it would look like in practice to use OP_CAT to assert transaction fields and properties. I hacked together a very basic vault using OP_CAT and no other non-mainnet features. 

The repo with a working demo you can run on regtest is available here: https://github.com/taproot-wizards/purrfect_vault

[BIP341 signature validation has us create a message called a `SigMsg`](https://github.com/bitcoin/bips/blob/deae64bfd31f6938253c05392aa355bf6d7e7605/bip-0341.mediawiki#common-signature-message) that contains commitments to the fields of a transaction. That SigMsg is then used as the message in constructing a Schnorr signature. [Andrew Polestra observed](https://medium.com/blockstream/cat-and-schnorr-tricks-i-faf1b59bd298) that if you set the Public Key (P) and Public Nonce Commitment (R) to the generator point (G), then the s value of the resulting Schnorr signature will be equal to the SigMsg + 1. We are using that technique in order to allow for transaction introspection by passing in the SigMsg components as witness data, and then using OP_CAT to construct the SigMsg on the stack. We then construct the tagged hashes specified in BIP341, and eventually CAT on an extra G to serve as the R component of the signature. Then we call CHECKSIG to validate the signature. If it is valid, then it means we've constructed the SigMsg correctly, and the transaction is valid.

We use that in a few different ways in this demo.

All the scripts are commented and in the `src/vault/script.rs` file.

### Trigger Withdrawal

* Inputs
  1. contract input
  2. fee-paying input
* Outputs
  1. Contract output with amount to be withdrawn
  2. Target address with dust amount

We use the CAT-checksig technique to validate that the amount and scriptpubkey of the first input and first output are the same. We enforce that the second output is amount is exactly 546 sats, but we do not place any restrictions on the scriptpubkey. We also enforce that there are two inputs and two outputs.

### Complete Withdrawal

* Inputs
  1. Withdrawal input
  2. Fee-paying input
* Outputs
  1. Destination output with contract amount

This is probably the most interesting transaction. We want to enforce that the first output has the scriptpubkey that matches the second output of the **trigger** transaction. To validate this, we pass in the serialized transaction data (version, inputs, outputs, locktime) as witness data, do some manipulation of the outputs, and then hash this previous-transaction data twice to get the TXID. We then validate that the first input of the current transaction is the same as the TXID of the previous transaction with vout=0. This ensures that the first input of the current transaction is the same as the first output of the previous transaction, and lets us *inspect the state of the previous transaction*.

The first output of this transaction is enforced to be the scriptpubkey of the second output of the trigger, and the amount is enforced to be the same as the amount of the first output of the trigger. The second input is unencumbered and used for change.

There is also a plain-old CSV relative timelock of 20 blocks on the first input.

### Cancel Withdrawal

* Inputs
  1. Any contract input
  2. Fee-paying input
* Outputs
  1. Contract output

This is the simplest transaction. We just enforce that there are two inputs and one output, and that the first input is the same as the first output.


There are some missing features and rough edges in the demo, but I think its constructive to see what it looks like in practice to use CAT to enforce different components of inputs and outputs, and to assert state from a previous transaction. 

Check out the README and code for gory details.

-------------------------

