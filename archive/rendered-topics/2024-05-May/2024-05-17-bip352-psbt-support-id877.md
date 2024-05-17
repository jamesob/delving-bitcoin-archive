# BIP352: PSBT support

josibake | 2024-05-17 11:51:04 UTC | #1

Started having some conversations about how to support sending and spending silent payment outputs in PSBTs, felt like a good time for a delving post to get some more eyes on it! Also, I am in no way an expert on PSBTs, so likely to make a lot of errors here.

## Spending

Spending seems the easiest to support and might be possible today by using the `PSBT_IN_PROPRIETARY` type. This would include the `shared_secret_tweak` (32 bytes) and the spend public key (33 bytes). The spend public key is used to get the spend private key and the spend private key is tweaked with the `shared_secret_tweak` when signing. The label tweak (if used) would already be added to the `shared_secret_tweak`.

Alternatively, we could create new field(s), something like `PSBT_IN_BIP352_TWEAK`, for example. Not sure if its possible to update the existing BIP174/370 or if we would need a new BIP for this.

## Sending

This is a bit more complicated. Constraints introduced when sending to a silent payments address are:

* The generated outputs depend on the inputs, so must be regenerated if the inputs change
* Generating the outputs requires access to the private keys*
* There constraints around what types of inputs are allowed in a silent payments transaction

delvingless-andrewtoth started a draft BIP for sending support here: https://gist.github.com/andrewtoth/dc26f683010cd53aca8e477504c49260.

The biggest change is the draft proposes a new `OutputGenerator` role: 

> The OutputGenerator role is only present for PSBTv3. The OutputGenerator must have access to all private keys for all inputs used for shared secret derivation. If the Has Silent Payments Flag is set to False, this role may be omitted. If there is not at least 1 eligible input for silent payment output generation, then the OutputGenerator must fail. The OutputGenerator generates the output scripts for all silent payment outputs using PSBT_OUT_SILENT_PAYMENT_CODE. It updates all PSBT_OUT_SCRIPT fields for all silent payment outputs with the appropriate outputs. It sets the Silent Payments Modifiable Flag to False. It sets the Inputs Modifiable to False. TODO: It may also generate a DLEQ proof using all the public keys of the inputs used for shared secret derivation and add it to the PSBT field PSBT_GLOBAL_SILENT_PAYMENTS_DLEQ_PROOF.
>
> A single entity is likely to be both an OutputGenerator and Signer.

This is nice in the sense that all of the silent payments logic can be encapsulated in this new role, requiring minimal changes to the other roles. The only change required for the signer would be to verify this new field, `DLEQ_PROOF`, which is a of verifying the outputs were created correctly without needing access to the private keys uses to create them. There are additional constraints listed in the draft for each of the other roles.

### Access to the private keys

It's also worth mentioning that the `OutputGenerator` *might* not need access to the private keys, but can accept something like an "ECDH share": 

$$
a_{sum} = a_1 + a_2 + ... + a_n \\
a_{sum} \cdot B_{scan} = a_1 \cdot B_{scan} + a_2 \cdot B_{scan} + ... + a_n \cdot B_{scan}
$$

So a signing device could return the ECDH share for a private key, instead of the full private key. Mentioning this as theoretically possible because this is not something that is provably secure and there are definite reasons to *not* have a signing device respond to Diffie-Hellman requests.

-------------------------

