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

andrewtoth | 2024-05-27 22:13:47 UTC | #6

[quote="josibake, post:1, topic:877"]
### Access to the private keys

It’s also worth mentioning that the `OutputGenerator` *might* not need access to the private keys, but can accept something like an “ECDH share”:

a_{sum} = a_1 + a_2 + ... + a_n \\ a_{sum} \cdot B_{scan} = a_1 \cdot B_{scan} + a_2 \cdot B_{scan} + ... + a_n \cdot B_{scan}

asum=a1+a2+...+anasum⋅Bscan=a1⋅Bscan+a2⋅Bscan+...+an⋅Bscana_{sum} = a_1 + a_2 + ... + a_n \\ a_{sum} \cdot B_{scan} = a_1 \cdot B_{scan} + a_2 \cdot B_{scan} + ... + a_n \cdot B_{scan}

So a signing device could return the ECDH share for a private key, instead of the full private key. Mentioning this as theoretically possible because this is not something that is provably secure and there are definite reasons to *not* have a signing device respond to Diffie-Hellman requests.
[/quote]

This could work and could eliminate the need for the OutputGenerator role altogether- each output has a new field for the sum of ECDH shares. Each signer adds their share to the sum. The last signer to add their share can then also generate all the output scripts and add their signature, and then the PSBT can be given to all other signers again to add their signatures.
But, how does each signer know that the other signers have added their correct share to the sum for each output? Can we have a running DLEQ proof sum as well? Otherwise we would need to store a proof for each input for each output and can't just have a running sum for each output.

-------------------------

josibake | 2024-05-28 12:16:55 UTC | #7

[quote="andrewtoth, post:6, topic:877"]
Can we have a running DLEQ proof sum as well? Otherwise we would need to store a proof for each input for each output and can’t just have a running sum for each output.
[/quote]

Not sure, but a running proof sum would be ideal. Also seems fine to include a proof with each input. Planning to take a look at the Musig2 PSBT BIP draft this week to see if we can learn anything from their approach, because this feels conceptually very similar.

-------------------------

andrewtoth | 2024-05-29 13:51:12 UTC | #8

I took a look at Musig2 PSBT BIP draft, and unless I'm mistaken it only has fields for inputs, i.e. spending support. So I don't think it would be useful for sending support.

If we can't have a running proof sum, the PSBT will have worst case inputs * outputs number of proofs. This could be too much data for coinjoins, but should be fine for regular transactions.

A workflow could look like this:

Signer gets a PSBT with no output scripts generated yet.

Case 1: Signer has private keys to all inputs and wallet trusts the signer. The signer generates the outputs itself and signs.

Case 2: Signer has private keys to all silent payment eligible inputs, but other inputs that are ignored by silent payment outputs are present or the wallet does not trust the signer. The signer generates the outputs, but also the ECDH sum and proof for all eligible inputs for each output and attaches one ECDH sum/proof to each output. The other signers that can sign for the non-silent payment eligible inputs use the proofs to verify the silent payment outputs before signing their inputs, or the wallet verifies the proof before broadcasting.

Case 3: Signer has some private keys to silent payment eligible inputs. It creates a ECDH share of the sum of all inputs it knows for each output, and attaches that along with the proof to each output, with the set of indexes that the proof corresponds to as the key data and the proofs as the value data. If all other shares and proofs for unknown inputs are already attached to the outputs, it can verify the proofs and generate the outputs using the other shares and sign. If some shares and proofs from other inputs are missing, it cannot sign yet and must receive the PSBT back after they are added by other signers, making this a 2-round step for this signer.

Edit - reading this again, all cases can be encompassed by case #3, and if the wallet trusts the signer then the proofs can just be omitted.

-------------------------

josibake | 2024-06-01 18:04:52 UTC | #9

[quote="andrewtoth, post:8, topic:877"]
If we can’t have a running proof sum, the PSBT will have worst case inputs * outputs number of proofs. This could be too much data for coinjoins, but should be fine for regular transactions.
[/quote]

Thinking about this more, we only need a proof for each input, i.e. a proof that the secret key used to create the ECDH share $a\cdot B_{scan}$ is the same key for $a\cdot G=A$.

We don't need a proof for each output because the software wallet simply needs to check the proof for each input and then sum up the ECDH shares and verify the outputs, i.e. 

$$
SS = a_1\cdot B_{scan} + .. + a_n\cdot B_scan \\
P = B_{spend} + hash_{BIP0352/SharedSecret}(SS || k)\cdot G \\
P \in \texttt{txoutputs}
$$

If a single signer has access to all of the silent payment private keys and is able to generate the outputs and sign the transaction, the signer can omit the per input proofs and instead provide a transaction level proof that $a_{sum}\cdot B_{scan}$ is the same key as $a_{sum}\cdot G = A_{sum}$, along with the shared secret $SS$, allowing the software wallet to verify the proof and calculate

$$
P = B_{spend} + hash_{BIP0352/SharedSecret}(SS || k)\cdot G \\
P \in \texttt{txoutputs}
$$

same as above.

-------------------------

andrewtoth | 2024-06-02 01:44:13 UTC | #10

[quote="josibake, post:9, topic:877"]
Thinking about this more, we only need a proof for each input, i.e. a proof that the secret key used to create the ECDH share a\cdot B_{scan}a⋅Bscana\cdot B_{scan} is the same key for a\cdot G=Aa⋅G=Aa\cdot G=A.

We don’t need a proof for each output
[/quote]


But doesn't the ECDH share a⋅Bscan differ for each output with a silent payment code, i.e. a different  Bscan? What I meant was this proof needs to be computed and attached for each silent payment code, i.e. each output, so there would be input * output many proofs in worst case where each input has a unique signer and each output has a unique Bscan.

-------------------------

josibake | 2024-06-02 08:44:37 UTC | #11

Ah, correct. I was thinking in the context of a single $B_{scan}$, but indeed if each output was to a separate silent payment output and each signer signs for only one input, then it becomes $inputs \cdot outputs$. Perhaps a better way to think of it would be $n_{proofs} = recipients \cdot signers$, since each signer can provide a proof which covers multiple inputs and each recipient can have more than one output?

-------------------------

andrewtoth | 2024-06-11 03:41:52 UTC | #12

I updated the draft BIP at https://gist.github.com/andrewtoth/dc26f683010cd53aca8e477504c49260. It removes the OutputGenerator role and instead uses the ECDH share technique described by @josibake here.

-------------------------

achow101 | 2024-06-13 23:11:18 UTC | #13

I had a call with @josibake last week discussing this, and I have a different proposal that I think is more efficient and also covers all of the edge cases.

There would be the following new input fields:

| Field name | `<keydata>` | key description | `<valuedata>` | value description |
|---|---|---|---|---|
| `PSBT_IN_SP_ECDH_SHARE` | `<33 byte scan key>` | The scan key this ECDH share is for | `<32 byte share>`| ECDH share for a scan key, computed with `a * B_scan` where `a` is this input's private key and `B_scan` is the scan key of a recipient |
| `PSBT_IN_SP_DLEQ` | `<33 byte scan key>` | The scan key this proof covers | `<64 byte dleq proof>` | The DLEQ proof computed with this input's private key and the scan key of the recipient |

and the following new output field:

| Field name | `<keydata>` | key description | `<valuedata>` | value description |
|---|---|---|---|---|
| `PSBT_OUT_SP_V0_INFO` | None | No key data | `<33 byte scan key> <33 byte spend key>` | The silent payments scan and spend pubkeys from the silent payments address |

With these fields, I believe it is okay to use PSBTv2 instead of defining a PSBTv3, with one modification to PSBTv2's invariats: at least one of `PSBT_OUT_SCRIPT` or `PSBT_OUT_SP_V0_INFO` must be present; both can also be included. For silent payment's aware parsers, the `PSBT_OUT_SP_V0_INFO` lets them compute the output script once all of the inputs are set and the ECDH shares are computed. For unaware parsers, the lack of `PSBT_OUT_SCRIPT` means that the PSBT will be seen as invalid and therefore abort and not do anything that could be problematic. Once all information is available, both could be set, in which case both silent payment's aware and unaware parsers will be able to handle the PSBT correctly.

The silent payments signers must check that the `PSBT_OUT_SCRIPT` is as expected if both `PSBT_OUT_SCRIPT` and `PSBT_OUT_SP_V0_INFO` are present. I suppose there is some risk of being tricked if you have an unaware signer, but that seems unlikely as they should be validating that the output script is as expected, and what would an unaware parser be expecting for this kind of output?

I also don't think there is a need for any changes to `PSBT_GLOBAL_TX_MODIFIABLE`.

In particular, constructors adding silent payments outputs need there to be either no inputs in the transaction, or that no inputs use a segwit version greater than 1. Signers will also need to validate that if there are any silent payments v0 outputs, there are no segwit v2+ inputs.

***

I'm probably forgetting a few things, but @josibake has notes so hopefully those can cover whatever it is I've forgotten.

-------------------------

andrewtoth | 2024-06-14 18:10:10 UTC | #14

Thanks for this! It is not far off from what I have.

In your approach though, each input will have a share and proof. In mine, all inputs that a signer has the private key for are combined into one share and proof. Can we combine these into having one proof for each scan key and set of inputs? Perhaps a global field where key data is 33 bytes scan followed by set of indexes?

Constructors also need to check if there are any ANYONECANPAY sighashes on any inputs before adding a silent payment, and updaters need to check as well, but I suppose that can be done without modifying `PSBT_GLOBAL_TX_MODIFIABLE`. The reason I added a Has Silent Payments flag to it is for the same reason there is a Has Sighash Single flag.

-------------------------

achow101 | 2024-06-14 18:38:31 UTC | #15

[quote="andrewtoth, post:14, topic:877"]
Can we combine these into having one proof for each scan key and set of inputs? Perhaps a global field where key data is 33 bytes scan followed by set of indexes?
[/quote]
Inputs can be added and removed if Inputs Modifiable is set, so I don't think it's a good idea to have anything that relies on the ordering of inputs to be consistent. Having any field that includes a list of indexes could become messed up by an unaware constructor that adds an input in the wrong place.

[quote="andrewtoth, post:14, topic:877"]
Constructors also need to check if there are any ANYONECANPAY sighashes on any inputs before adding a silent payment, and updaters need to check as well, but I suppose that can be done without modifying `PSBT_GLOBAL_TX_MODIFIABLE`. The reason I added a Has Silent Payments flag to it is for the same reason there is a Has Sighash Single flag.
[/quote]
I remember discussing this with @josibake on the call, but don't fully remember the conclusion. I think it was something like it wasn't necessary to make any special consideration for ANYONECANPAY if we require that silent payments outputs can only be added if there are no inputs yet, or inputs with no signatures, or Inputs Modifiable is not set.

As long as an input uses SIGHASH_ALL, I don't think an input with ANYONECANPAY is actually an issue.

-------------------------

andrewtoth | 2024-06-14 19:24:15 UTC | #16

[quote="achow101, post:15, topic:877"]
As long as an input uses SIGHASH_ALL, I don’t think an input with ANYONECANPAY is actually an issue.
[/quote]

Hmmm.... but if signing with ANYONECANPAY, even with SIGHASH_ALL, then Inputs Modifiable will not be set to False. So another SP unaware constructor can add a new input that modifies the shared secret and invalidates the already signed outputs.

-------------------------

andrewtoth | 2024-06-14 23:48:23 UTC | #17

[quote="achow101, post:15, topic:877"]
Inputs can be added and removed if Inputs Modifiable is set, so I don’t think it’s a good idea to have anything that relies on the ordering of inputs to be consistent. Having any field that includes a list of indexes could become messed up by an unaware constructor that adds an input in the wrong place.
[/quote]

If we can only add silent payment outputs if Inputs Modifiable is not set, then this point is moot, no?

I don't think we should dismiss this optimization. Consider the common case of a wallet with 10 small utxos, and it wants to make a single payment with all of them to a silent payment address. Without consolidating all shares and proofs, the hardware wallet signer will need to compute 10 times more shares and proofs.

-------------------------

andrewtoth | 2024-06-14 23:45:29 UTC | #18

[quote="achow101, post:15, topic:877"]
As long as an input uses SIGHASH_ALL, I don’t think an input with ANYONECANPAY is actually an issue.
[/quote]

Thinking about this more, if any fully signed transaction spending silent payments that contains an ANYONECANPAY input that spends at least the amount of the inputs it signs (could be SIGHASH_ALL), couldn't any outside observer simply strip the other inputs and rebroadcast, which would still be a valid tx but all silent payment outputs would be invalid?

-------------------------

josibake | 2024-06-18 13:10:57 UTC | #21

[quote="achow101, post:13, topic:877"]
I’m probably forgetting a few things, but @josibake has notes so hopefully those can cover whatever it is I’ve forgotten.
[/quote]

Checking over my (not so great :sweat_smile:) notes, I think the only thing missing is the ordering of the silent payment addresses in cases where two or more `PSBT_OUT_SP_V0_INFO` fields contain the same scan key. In order to ensure any one of the SP aware signers can arrive at the same set of generated output scripts, each signer will need to sort the silent payment address by scan public key and spend public key, in lexicographic order. This guarantees that everyone gets the same values for `k` when creating multiple outputs for the same scan public key. As a reminder, this has nothing to with the final ordering in the transaction.

---

@andrewtoth - regarding `ANYONECANPAY`:

[quote="andrewtoth, post:16, topic:877"]
Hmmm… but if signing with ANYONECANPAY, even with SIGHASH_ALL, then Inputs Modifiable will not be set to False. So another SP unaware constructor can add a new input that modifies the shared secret and invalidates the already signed outputs.
[/quote]

These are the possible scenarios I see:

1. PSBT does not contain any `PSBT_OUT_SP_V0_INFO` fields and someone signs with `ALL | ACP`. At this point, the outputs have been committed to, making it impossible to add a silent payment recipient

2. PSBT contains only `PSBT_OUT_SP_V0_INFO` fields. SP unaware signers can't sign since there are no `PSBT_OUT_SCRIPT` fields, and SP aware signers will either add their shares/proofs or generate the SP outputs and sign after checking that all shares / proofs are present

3. PSBT contains a mix of `PSBT_OUT_SP_V0_INFO` and `PSBT_OUT_SCRIPT` fields. If the SP aware signers have not finished adding their SP input data outputs with `PSBT_OUT_SP_V0_INFO` set will not have a `PSBT_OUT_SCRIPT` field, which means any non-SP aware signer will fail if trying to sign with `ALL | ACP` due to seeing the PSBT as malformed. This means the last SP signer to add their share / proof MUST sign with `ALL`. Said differently, this is only a problem if an SP aware signer were to validate the shares / proofs, generate the SP output scripts and then sign with `ALL | ACP`

I think this ends up being fine with the understanding that a non-SP aware signer cannot sign the transaction until each output has a `PSBT_OUT_SCRIPT` field. The `PSBT_OUT_SCRIPT` fields can only be set once all of the SP fields have been set, which always gives the last SP signer a chance to lock the inputs with `ALL`.

[quote="andrewtoth, post:18, topic:877"]
Thinking about this more, if any fully signed transaction spending silent payments that contains an ANYONECANPAY input that spends at least the amount of the inputs it signs (could be SIGHASH_ALL), couldn’t any outside observer simply strip the other inputs and rebroadcast, which would still be a valid tx but all silent payment outputs would be invalid?
[/quote]

I don't see how this situation would be possible, so long as we require silent payments aware signers to never use `ALL | ACP`, right? A non-SP signer cannot sign until the `PSBT_OUT_SCRIPTS` exist and these fields cannot exist until all of the SP aware signers have added their data, which means the last SP aware signer to add will always have a chance to sign the transaction.

---
@achow101 , @andrewtoth regarding requiring a proof per input:

[quote="andrewtoth, post:17, topic:877"]
I don’t think we should dismiss this optimization. Consider the common case of a wallet with 10 small utxos, and it wants to make a single payment with all of them to a silent payment address. Without consolidating all shares and proofs, the hardware wallet signer will need to compute 10 times more shares and proofs.
[/quote]

This is a good point considering this would require the signer to do 3x the work (~30 ECC mults): signature, ECDH share, DLEQ (signature) for each input. If we allow the signer to consolidate the shares / proofs, this would be one ECDH, one proof, and 10 signatures. One alternative is to allow the proof to be duplicated for inputs belong to the same signer. This removes any order dependence, but requires the verifier of the proofs / shares to be able to group them and add up the public keys for grouped inputs, i.e.,

```
input_0:proofA:shareA
input_1:proofB:shareB
input_2:proofA:shareA

// Verifier

sum(input_0_pubkey, input_3_pubkey); verify with proofA:shareA
input_1; verify with proofB:shareB
```

Seems like we can save the signers extra computation at the expense of more data in the PSBT (not really a concern imo) and more work for the verifier.

---
EDIT: delving was yelling at me for posting too many small replies, so I deleted the inline replies and consolidated them into this post

-------------------------

andrewtoth | 2024-06-18 15:50:41 UTC | #22

[quote="josibake, post:21, topic:877"]
I don’t see how this situation would be possible, so long as we require silent payments aware signers to never use `ALL | ACP`, right?
[/quote]

Right, we need to require silent payments aware signers to never use `ACP` at all if there are any silent payment outputs present. If there are no silent payment outputs they can sign with whatever sighash they wish. I suppose we don't need to track that state for Constructor and Updater, since in BIP174 the Signer has the check:
> * If a sighash type is provided, the signer must check that the sighash is acceptable. If unacceptable, they must fail.

which for silent payment aware signers they would check for any `ACP` on any inputs and fail if there are any silent payment outputs. I'm not sure why there is a Has Sighash Single flag though and we shouldn't extend the `PSBT_GLOBAL_TX_MODIFIABLE` to have a Has Silent Payments flag to track this state in the same way. That way Constructors won't add silent payment outputs if there have been any ACP sigs and Updaters won't add any ACP sighashes to inputs if there are any silent payments. It should be backwards compatible too to add a flag to that field.

However, the next rule for Signer needs to be modified for silent payment aware signers:
> * If a sighash type is not provided, the signer should sign using SIGHASH_ALL, but may use any sighash type they wish.

should be:

> * If a sighash type is not provided and there are silent payment outputs present, the signer must sign using SIGHASH_ALL. If a sighash type is not provided and there are no silent payment outputs present, the signer should sign using SIGHASH_ALL, but may use any sighash type they wish.

[quote="josibake, post:21, topic:877"]
One alternative is to allow the proof to be duplicated for inputs belong to the same signer.
[/quote]

How about adding the shares and proofs as globals, and the key data would be the scan key followed by the set of input outpoints instead of input indexes? That would let the psbt change input order and would not duplicate the shares and proofs for each input.

[quote="josibake, post:21, topic:877"]
and more work for the verifier.
[/quote]

This would be less work for the verifier too, right? Less proofs to verify.

-------------------------

