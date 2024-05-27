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

Sosthene | 2024-05-19 17:32:48 UTC | #2

FWIW we already wrote with cygnet some basic psbt workflow for donation wallet (I'm also using it in my wasm experiments). That's hacky and has limitations, but it already works so it can serve as a basis for further improvements.

For spending from sp outputs, I'm not quite sure what you mean with 
[quote="josibake, post:1, topic:877"]
The spend public key is used to get the spend private key and the spend private key is tweaked with the `shared_secret_tweak` when signing.
[/quote]

but basically we use the proprietary field of inputs that are spending a sp prevout to keep the tweak to the spend private key, and I think that's what you mean. 

When signing we pass along the spend private key and simply add the tweak to get the signing key, see https://github.com/cygnet3/sp-client/blob/0a81059e2959b87798432fc10c9447ba276dc1a1/src/spclient.rs#L684

As for spending to sp address, it's indeed a bit more complicated but as long as you only bother about simple use cases I didn't think it was that bad. We use the output proprietary field to keep the recipient sp address and put a placeholder scriptpubkey (basically the pubkey computed from a NUMS) in the actual unsigned transaction, to allow for accurate fee calculation. The psbt can be modified, inputs added or whatever, up to the point you want to cement things down and compute the actual output keys for whatever is the current state of the transaction https://github.com/cygnet3/sp-client/blob/0a81059e2959b87798432fc10c9447ba276dc1a1/src/spclient.rs#L267

This part could certainly be improved and made more efficient, but it worked quite well so far, we had a lot of issues but not with that.

It's simpler in our use case though because we're making pure silent payments wallets, so we just assume that prevouts are silent payment taproot scripts and don't bother about ineligible inputs. We also assume that we own all prevouts for now. 

I think that since psbt already saves basically all relevant information about the prevout, discriminating between eligible or non eligible prevout will be easy. The 2nd problem is harder, as we could do something like the ECDH share you mentioned but I have no idea if that would be safe either. Probably something we should talk about with the secp256k1 guys, because I would be very disappointed not to be able to build a coinjoin wallet around silent payment.

-------------------------

josibake | 2024-05-20 12:01:29 UTC | #3

[quote="Sosthene, post:2, topic:877"]
For spending from sp outputs, I’m not quite sure what you mean with
[/quote]

In the PSBT you need the `shared_secret_tweak` and also a way to specify *which* spend private key to tweak. For example, I am spending outputs sent to two entirely separate silent payment addresses in the same PSBT: I need a way to say "the `shared_secret_tweak` for this input is used to tweak spend key A and the `shared_secret_tweak` for that input is used to tweak spend key B."

[quote="Sosthene, post:2, topic:877"]
I think that since psbt already saves basically all relevant information about the prevout, discriminating between eligible or non eligible prevout will be easy
[/quote]

This is one I need to double-check: does the PSBT provide enough information about the prevout for us to ensure the output being spent is not a segwit witness version > 1, for example?

-------------------------

andrewtoth | 2024-05-20 17:56:49 UTC | #4

[quote="josibake, post:1, topic:877"]
delvingless-andrewtoth
[/quote]

Delvingless no longer :smiley:. 

[quote="Sosthene, post:2, topic:877"]
and put a placeholder scriptpubkey (basically the pubkey computed from a NUMS) in the actual unsigned transaction, to allow for accurate fee calculation.
[/quote]

I think we shouldn't do this because if PSBT_OUT_SCRIPT is in the PSBT then older version implementations would still sign it and ignore the actual silent payment field. That is why PSBT_OUT_SCRIPT must now be optional for outputs when the constructor adds them. Once the outputs are generated, the Inputs Modifiable Flag is set to False so no input changes can invalidate the computed silent payment addresses. The new Silent Payments Modifiable Flag is set to False as well so no new silent payment outputs can be added. This makes it so the outputs only need to be generated once.
As for fee calculation, anything computing the weight can just use the taproot output size for any silent payment output.

[quote="josibake, post:3, topic:877"]
[quote="Sosthene, post:2, topic:877"]
I think that since psbt already saves basically all relevant information about the prevout, discriminating between eligible or non eligible prevout will be easy
[/quote]

This is one I need to double-check: does the PSBT provide enough information about the prevout for us to ensure the output being spent is not a segwit witness version > 1, for example?
[/quote]

We can get the information we need to determine if an output being spent is not a segwit witness version > 1, but it will require us to merge the Constructor and Updater roles.
In my draft, the Constructor and Updater are responsible for maintaining mutual exclusion between silent payment outputs, and inputs that are either ANYONECANSPEND or Segwit version > 1. A PSBT can't have both, and this is maintained with the new flags Silent Payments Modifiable and Has Silent Payments.
The Constructor can set Has Silent Payments if it adds a silent payment output, and the Updater can check this flag and refuse to add ANYONECANSPEND sighash flags to any inputs if set. Vice-versa with Updater setting Silent Payments Modifiable to False if it adds ANYONECANSPEND, and then the Constructor can't add any Silent Payment Outputs.
This won't work for Segwit version > 1 inputs though, since the Constructor can only add the outpoint as an input, and the Updater is responsible for adding the input's scriptPubkey. If the scriptPubkey is a Segwit version > 1, it's too late because the input is already added. This would result in a malformed PSBT. Not sure how to proceed with this.

-------------------------

andrewtoth | 2024-05-27 00:34:44 UTC | #5

Updated the draft BIP, added the DLEQ proof based on https://gist.github.com/RubenSomsen/be7a4760dd4596d06963d67baf140406.

-------------------------

