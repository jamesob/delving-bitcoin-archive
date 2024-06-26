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

dgpv | 2024-02-17 05:27:25 UTC | #2

Since I like to try B'SST on any not-entirely-trivial script I stumble upon, I've tried it with `vault_trigger_withdrawal` script from your demo. (B'sst is one of the names of Bastet, the ancient egyptian cat-goddess, so cannot ignore the CAT demo :-))

I think it might be interesting to look at the report, as it shows what this script does quite concisely, in my opinion.

The annotated script can be found here: https://gist.github.com/dgpv/f875e021905eb113070a23eb7fa981f6, you need to call `bsst-cli` with `--explicitly-enabled-opcodes=cat`

The report:

```
==============================
Enforced constraints per path:
==============================

All valid paths:
----------------

        EQUAL(&script_computed_sig, precomputed_sig_sans_last_byte<wit0>.x('00')) @ 77:L104
        CHECKSIG(precomputed_sig_sans_last_byte<wit0>.x('01'), $G_X) @ END

=================================
Witness usage and stack contents:
=================================

All valid paths:
----------------
Witnesses used: 17

Stack values:
        <result> = CHECKSIG(precomputed_sig_sans_last_byte<wit0>.x('01'), $G_X) : one_of(0, 1)

================
Data references:
================

        outputs_single_hash = SHA256(amount_buffer<wit4>.script_pubkey_buffer<wit3>.$DUST_AMOUNT.target_script_pubkey_buffer<wit5>)
        spent_scripts_single_hash = SHA256(script_pubkey_buffer<wit3>.fee_script_pubkey_buffer<wit1>)
        spent_amounts_single_hash = SHA256(amount_buffer<wit4>.fee_amount_buffer<wit2>)
        sig_hash = epoch<wit16>.control<wit15>.tx_version<wit14>.lock_time<wit13>.prevouts_single_hash<wit12>.&spent_amounts_single_hash.&spent_scripts_single_hash.prev_sequences_single_hash<wit11>.&outputs_single_hash.spend_type<wit10>.input_idx<wit9>.leaf_hash<wit8>.key_version_0<wit7>.code_separator_pos<wit6>
        tagged_sig_hash = SHA256(SHA256($TAPSIGHASH_TAG).SHA256($TAPSIGHASH_TAG).&sig_hash)
        s_value = SHA256(SHA256($BIP0340_CHALLENGE_TAG).SHA256($BIP0340_CHALLENGE_TAG).$G_X.$G_X.&tagged_sig_hash)
        script_computed_sig = $G_X.&s_value
```
(edit: I wonder if it is possible to make the codeblock to have the text to wrap, it would look better I think

edit2: it seems that currently it only wraps on whitespace, but not as terminal would wrap on any char)

There's one obvious witness size optimization that comes to mind when looking at the report: 

`epoch<wit16>.control<wit15>.tx_version<wit14>.lock_time<wit13>.prevouts_single_hash<wit12>`

and 

`spend_type<wit10>.input_idx<wit9>.leaf_hash<wit8>.key_version_0<wit7>.code_separator_pos<wit6>`

can be given as just two witness values, not as 10 witnesses - this will save a few bytes used to encode witness sizes.

-------------------------

rijndael | 2024-02-22 13:42:54 UTC | #3

This is super cool. 

The way I have been experimenting with using CAT to make covenants is I have a function to build up the elements of the BIP341 SigMsg and then spit them out as a vector of elements, and then I pre-commit to the ones that I want to be fixed in the script, and push the rest of them in the transaction witness. In the script, it CATs all of these items together (assembling the SigMsg), and then use that to construct a tagged hash, then CAT on some more tag values and copies of the secp generator point and hash all of that to get the `s` value of a schnorr signature that is valid for the transaction. I knew that if I pre-concatenated the "free" values of the SigMsg outside the script instead of on the stack, I could save some bytes, but I hadn't done it yet to keep my code more flexible for experimenting. It's **very** cool to have BSST tell me exactly how much overhead that's costing me.

I need to spend some time playing with BSST!

-------------------------

dgpv | 2024-02-22 14:16:26 UTC | #4

I think that in addition to the number of witnesses, there's also overhead in extra opcodes that are used to CAT the parts that can be pre-concatenated

This demo excellently showcases the use of CAT for tx introspection, even with the scripts that are not max-optimized.

But with less witness inputs these scripts can be easier to understand for those who want to look at this demo in detail

-------------------------

dgpv | 2024-04-10 20:20:22 UTC | #5

[quote="rijndael, post:1, topic:576"]
We use the CAT-checksig technique to validate that the amount and scriptpubkey of the first input and first output are the same. We enforce that the second output is amount is exactly 546 sats, but we do not place any restrictions on the scriptpubkey. We also enforce that there are two inputs and two outputs.
[/quote]

I think that the covenant script does not actually enforce all that.

It enforces that the amount and scriptpubkey of first input is the same as the amount and scriptpubkey of the first output.

But It does not validate the sizes of the buffers - that means that `target_scriptpubkey_buffer` can contain extra data, for extra outputs.

The `fee_scriptpubkey_buffer` and `fee_amount_buffer` can contain extra data, too, for extra inputs.

If the `script_pubkey_buffer` contains extra data, it will interfere with calculation of `spent_scripts_single_hash` as that extra data will be taken as scriptpubkey data, while it will need to contain the amounts. But IIRC, non-standard taproot outputs are treated as anyone-can-spend by miners, so maybe some manipulation is possible here, too. 

I think the script should have size checks added for the sizes of all the buffers, just in case.

-------------------------

rijndael | 2024-04-10 17:23:10 UTC | #6

Good call-out. I'll add length checks on the next iteration :)

-------------------------

dgpv | 2024-04-12 08:54:41 UTC | #7

The miner can drain this vault at 'trigger withdrawal' transaction by putting their input as first input, and the covenant-locked input as the second input. The output_0 will just be their input_0 amount going back to them (so `input_0=output_0` condition will be satisfied) , while the covenant input_1 amount will be split between 546-sat dust amount at output_1 and the fee. Since the fee goes to the miner, they will be taking everything from the vault. Non-miner can just sabotage the vault by draining it to a random miner.

I currently don't know the way how this can be fixed (edit: there are ways, see below).

By the way, 546 sat is the dust for non-segwit, AFAIR. For segwit, the amount calculated by `IsDust()` will be lower

-------------------------

dgpv | 2024-04-11 21:59:38 UTC | #8

I guess you can avoid draining by miner by enforcing zero fee: require the non-covenant input to be dust amount (the same amount as output_1), and then the transaction will need to be paid for by CPFP using the dust-paying output.

There could also be other complications on the 'complete withdrawal' phase, but I did not analyze it yet

-------------------------

rijndael | 2024-04-11 22:23:13 UTC | #9

Excellent find! You're right: if we are using input/output ordering to implicitly check for where the covenant input/output is, but can't enforce it, we're gonna have a bad time. 

for this vault case specifically, one easy mitigation would be to require a signature check in addition to the rest of the script. OP_VAULT has the idea of a "trigger key". I think that's a feature that makes a lot of sense. There's not a great reason it's not included in this implementation, I just didn't do it. I'll add that to my todo list as well :slight_smile: 

Thinking more generally, we do cover the [index of the input](https://github.com/taproot-wizards/purrfect_vault/blob/ee7ee57ba9b5ed8fee7382334625bacbe27f8c36/src/vault/signature_building.rs#L266) in our signature message ([full spec in bip341](https://github.com/bitcoin/bips/blob/b3701faef2bdb98a0d7ace4eedbeefa2da4c89ed/bip-0341.mediawiki#common-signature-message)) so I think we could enforce that the covenant input is at index 0 (or whatever). 


On the dust, you are also right. I was seeing a lot of 546-value'd outputs and just picked that as reasonable-dust.

-------------------------

dgpv | 2024-04-11 22:26:20 UTC | #10

[quote="rijndael, post:9, topic:576"]
Thinking more generally, we do cover the [index of the input](https://github.com/taproot-wizards/purrfect_vault/blob/ee7ee57ba9b5ed8fee7382334625bacbe27f8c36/src/vault/signature_building.rs#L266) in our signature message ([full spec in bip341](https://github.com/bitcoin/bips/blob/b3701faef2bdb98a0d7ace4eedbeefa2da4c89ed/bip-0341.mediawiki#common-signature-message)) so I think we could enforce that the covenant input is at index 0 (or whatever).
[/quote]

Yes, I think enforcing that the current input index is 0 should fix this.

-------------------------

dgpv | 2024-04-11 23:01:25 UTC | #11

What is the purpose of fixing `output_1` to dust amount ?

It seems to me that if you fix the covenant input index to 0 you can allow second output to be any amount.

`input_0 = output_0` and covenant_input_index is 0, and there is only two inputs and two outputs.

Therefore, the `output_1` amount can only be equal to `input_1` amount minus the fee.

-------------------------

dgpv | 2024-04-12 17:47:47 UTC | #12

[quote="dgpv, post:8, topic:576"]
There could also be other complications on the ‘complete withdrawal’ phase, but I did not analyze it yet
[/quote]

the 'complete withdrawal' case must enforce `input index = 0`, too -- otherwise you can take two input utxo that are in 'trigger withdrawal' state and use one as 'contract input' and second one as 'fee-paying input', and they can be from different vaults. The second utxo can then be drained to the miner as the fee.

-------------------------

rijndael | 2024-04-12 18:01:01 UTC | #13

yep, anywhere we check input.spk == output.spk, we should also be doing an input index assertion.

-------------------------

dgpv | 2024-04-14 19:43:53 UTC | #14

The structure of the transaction that sends funds to the vault for the first time is also important

Conventional wallets might shuffle outputs.

If conventional wallet is used to send to such vault, it can put vault output at index 1 and change output at index 0

In this situation, the 'complete withdrawal' covenant case can be abused: vault address will be used as destination for withdrawal, but the amount of "change" from "envault" transaction will be used as withdrawal amount, and the rest will go to the change or the fee

-------------------------

dgpv | 2024-04-14 21:54:52 UTC | #15

Given that 'complete withdrawal' is not expected to be available without 'trigger withdrawal', I guess the "envault" transaction must have only one output

-------------------------

dgpv | 2024-04-17 12:40:19 UTC | #16

I've done some modelling and analysis using [Alloy](https://alloytools.org/), and it looks like you don't actually need to enforce the number of inputs and outputs of the current transaction - only the number of outputs of the previous transaction in 'complete withdrawal' case

If there's no enforcement on number of outputs, it turns out that you also don't need 'cancel' as a separate covenant case - it can be handled by the same covenant as the 'trigger' case - simply any 'trigger' transaction that does not have 2 outputs is effectively 'cancel' transaction

You need to enforce the input indexes, though.

I'm still tinkering with the model, so the above is not final conclusions.

I will publish the details a bit later in a separate post.

-------------------------

dgpv | 2024-04-19 21:49:44 UTC | #17

Here's the details of my analysis using Alloy model checker: https://delvingbitcoin.org/t/analysing-simple-vault-covenant-with-alloy/819

-------------------------

rijndael | 2024-04-21 21:02:51 UTC | #18

Oh cool! I've done a fair amount of modeling in TLA+ in a past life, checking out Alloy has always been on my todo list. I'll check out your analysis on the other thread. Thanks for sharing this!

-------------------------

wade-liwei | 2024-05-04 10:57:30 UTC | #19

Could you please provide the demo with golang?
Rust is too diffcult forme.

-------------------------

