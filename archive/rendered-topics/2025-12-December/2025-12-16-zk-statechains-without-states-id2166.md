# ZK-Statechains Without States

evd0kim | 2025-12-16 19:47:15 UTC | #1

*Statechains are a concept that were introduced by Ruben Somsen more than 6 years ago. Since then Mercury Layer was launched by Commerce Block as arguably the most advanced Statechain - with blinded signing. In this article, we are going to introduce a spin-off protocol that we believe possesses advanced privacy properties. In this protocol, state transitions inside a dedicated ledger are replaced with zkSNARKs. The selected programmable ZK stack also allows for more flexibility when it comes to enabling smart contract support. We outline ideas for market-based solutions to address the non-fungibility of statecoins and bring-in private, albeit limited, zero-knowledge smart contracts for this execution layer on top of Bitcoin.*

# Intro

When working on a bitcoin-specific application for private payments, our team began thinking about ways to improve the trust assumptions related to pegging in and out of the protocol. Currently, the network hosts a “bridge” contract on a bitcoin rollup, making the protocol a Bitcoin “L3” per marketing language used on twitter.

But the application itself is not a “layer” on top of any protocol. It is a sovereign network with its own consensus mechanism. The fact that it can host light clients on other protocols, for purposes of message passing, can connect it to bitcoin (or any other blockchain).

When thinking through this lens, we’ve asked ourselves, “how can we improve the trust assumptions for bitcoiners who want improved privacy?” We thought through many ideas; BitVM, Ark, and other types of Layer 2 protocols.

We’ve found that our protocol works well with statechain-like assumptions. Meaning, a user can deposit funds into a multisig collaboratively with an operator. They can transfer ownership of this UTXO, backing private notes on the private network, by transferring their spending key to a recipient. When doing this, the operator is trusted to not collude with the spender to double-spend the recipient.

In this post, we cover how a statechain-like deposit mechanism can offer a different set of tradeoffs for users who want to have improved privacy related to their bitcoin holdings.

# Literature Review

![image|395x296](upload://134kwfxLwAIlmTVf3pZc2vZuJZN.png)

The basic idea behind statechains is that a user Bob (public key B) locks his funds in a 2-of-2 multisig. One multisig key is held by the statechain entity Alice (public key A) and the other by Bob (public key X). The statechain entity (Alice) and Bob together pre-sign a unilateral exit transaction for Bob in case of unresponsiveness from the statechain entity (Alice). When Bob wants to transfer the money (the entire UTXO) to Claire (public key C), he simply hands over the private key for X to Claire.

Some resources on statechains are:

* In the Mercury Layer design, the Statechain entity is signing blinded messages — it does not know whether these are Bitcoin transactions or something else entirely ([image source](https://medium.com/@RubenSomsen/statechains-non-custodial-off-chain-bitcoin-transfer-1ae4845a4a39)).

* Ruben Somsen’s [blog](https://medium.com/@RubenSomsen/statechains-non-custodial-off-chain-bitcoin-transfer-1ae4845a4a39) is the most up-to-date source of accessible information about statechains principles and design.

* Later design ideas proposed a “channel” based escrow that relies on Eltoo introduced by Christian Decker, Rusty Russell and Olaoluwa Osuntokun. This version of Statechains does not require a Statechain Entity. However, it can’t work with Bitcoin now since it requires an activation of SIGHASH_NOINPUT opcode to enable Eltoo. This opcode was proposed in a [draft BIP-118](https://github.com/bitcoin/bips/blob/1e663c21500979451c97da390ddd3052866af101/bip-0118.mediawiki) in 2017 and unlikely to be activated soon on the Bitcoin blockchain.

Statechain Entity-based designs achieved peak performance, in terms of privacy, via the Mercury Layer implementation. A detailed description of the protocol is available in the repository, in [the documentation section](https://github.com/commerceblock/mercurylayer/tree/dev/docs). Whereas receivers still have to verify the history of the statecoin, the Statechain Entity has no insight into the statecoin thanks to blind signing of the UTXO transfer.

Another protocol flavor called Spark, which was launched in 2025, addressed non-fungibility of statecoins with its “[leaf splitting](https://docs.spark.money/learn/technical-definitions#splitting-a-leaf)” technology. This way, users could spend not just whole UTXOs on the statechain, but use smaller amounts which greatly improved user experience (when compared to Commerce Block’s Mercury Layer protocol). To our knowledge, Spark operators do **not** use blind signing, thus they have a record of every transaction that takes place in the Spark protocol. Leaf splitting also suggests there is a very large amount of pre-signed transactions that have to be constantly updated, adding significant overhead for the operators. Finally, the launch of Spark support in Wallet of Satoshi [highlighted](https://x.com/evankaloudis/status/1975997003680108908) that all Lightning payments for WoS users may be publicly visible via Spark’s block explorers.

Spark and Mercury layer are production ready protocols based on the statechain paradigm. A research implementation from Super Testnet possibly provides the simplest implementation of statechains; [statechainjs](https://github.com/supertestnet/statechainjs). An interesting feature of this project is its reliance on relative timelocks. Relative timelocks do not require statecoin updates at fixed block heights, as opposed to absolute timelocks that force users to come onchain prior to the timelock expiry. They defer this inevitable “reset” of the backup transaction to an undefined time horizon that depends on the usage of the corresponding coin. However, *any prior holder may broadcast their pre-signed exit transaction that “starts the clock,” but they can’t withdraw til after the latest holder gets a chance to do so, so prior holders have limited incentive to try and cheat the current user – unless they think the “real holder” is offline, in which case they may be able to steal from them. Due to this possibility, the latest holder has to watch the chain “just in case,” and be ready to broadcast their withdrawal transaction “first.”*

Super Testnet’s experimental implementation also lacks support for the important concept of [Adaptor signatures](https://conduition.io/scriptless/adaptorsigs). Therefore this design relies significantly on incentives to work cooperatively and not withholding key shares during statecoin transfers.

# ZK Execution Layer for Private Transfers

Somsen’s original idea implied a degree of freedom when it comes to statecoin “ledger” that registers state coins transfers for coordination purposes. It practically addresses technical issues that stem from the ability of statechain users to look up original deposit transactions, track transitory keys, timelocks and verify backup transactions.

We propose a combination of statechain exit mechanics with a zero-knowledge transaction layer (ZK Ledger) as the main ledger that is responsible for the validation of “key changes” and simultaneous relative timelock decrements corresponding to onchain Bitcoin transactions. Our idea relies [on previous work](https://polybase.github.io/zk-rollup/whitepaper.pdf) related to the ZK Rollup design for Payy Network. We found that the notion of private “payment links” to be potentially synergetic with statechain-style protocols.

When connected to statechains, the enhanced programmability of the ZK Ledger allows for simplified backup transaction updates. In this model, the receiving party verifies a zero-knowledge proof as a part of the UTXO transfers, in addition to normal ways to verify bitcoin transactions. It is our view that this programmability makes the protocol simpler and closer to Super Testnet’s demo case.

![image|513x295](upload://x794W2gVnWvmvH1nfkm74Ecnrih.png)

Transmitting notes P2P

## Protocol explainer

Let’s dive into more details about proposed architecture. On the highest level, the Sender and Receiver facilitate a peer-to-peer transfer. For the Sender to generate a valid zkSNARK proof and transfer ownership of the UTXO to the recipient, they must have the corresponding the note’s data (note being the private, offchain representation of an onchain UTXO). This zkSNARK proof is transferred to the user client-side, and only the hash commitment of the note is published to the operator within the ZK Ledger. Senders coordinate with the Statechain entity to pass ownership of the UTXO to a recipient. Upon validating the validity of the UTXO, recipients receive the coin and trust the operator to not collude with previous owners to double spend them. For the remainder of this post, we will refer to the Statechain entity as the “Operator”.

## Protocol roles

The protocol has three actors; the operator, the sender and the receiver who operate the ZK Ledger. Each of these parties are responsible for specific items during a statecoin transfer.

*ZK Ledger:* The ZK Ledger is a sovereign network with its own consensus rules and implementation. It enables token transfers that are proven valid through zkSNARKs and minimal information leakage related to participants in a transaction.

*Operator*: The operator is responsible for maintaining a ledger of note commitments that nullify a sender’s ability to double-spend a recipient (i.e. run a ZK Ledger full node). Upon verifying a sender’s proof to spend a UTXO, and ensuring the newly generated output commitment does not exist within the list of previous commitments, the operator co-signs the new presigned transaction. The Operator is meant to be a Statechain Operator which is an independent role to validators in the ZK ledger.

*Sender*: The sender is a user in the protocol that wants to transfer funds to another party. To transfer these funds, they construct a new presigned transaction that designates the recipient as the recipient for the payout transaction. This presigned transaction has a lower timelock than that of their presigned transaction.

*Receiver*: The receiver is responsible for initiating transfers within the protocol and verifying their validity. When they want to receive funds, they generate a secret key that is passed to the sender, which the sender then uses to pre-sign a new pay out transaction (related to the bitcoin UTXO locked in the onchain multisig) specifying the recipient as the previous owner. After the sender and the operator pre-sign a new payout transaction, and this information is communicated with the recipient, the recipient verifies that the payout transaction is for a valid bitcoin UTXO and that their pay out transaction timelock is lower than that of the sender.

### Payment flow

The payment flow can be described as the following: a user deposits funds into a 2-2 Musig address where they and the Operator each hold a keyshare. This deposit transaction is mirrored on the ZK Ledger and credits the user’s balance with an amount of bitcoin-denominated tokens that is equivalent to that of tokens held in the multisig address. The Operator can be a single signer or their key can be sharded across multiple signers using some Threshold Signature Scheme. When depositing funds into the protocol, the user works with the Operator to generate a presigned transaction that enables them to unilaterally exit the protocol in case of an Operator liveness failure.

To send funds to a recipient, the sender is responsible for communicating her UTXO in the form of “input note’s” data to a receiver via short “payment links”. Such payment links may be shown as QR code or sent via communication channels. This way the receiver “pulls” payment and constructs her new UTXO. SuperTestnet suggested a way that is [featured by some eCash projects](https://thebitcoinmanual.com/articles/send-ecash-tokens-nostr/) when bearer tokens are sent via Nostr relays. If implemented, it would mean achieving the goal of “push” payments via a combination of creating a “pull” note by the Sender, encrypting it for the Receiver using the Receiver’s Nostr public key, and broadcasting the message via a dedicated Nostr relay.

To fully facilitate a payment, the sender (Bob) first proposes a transaction within the ZK Ledger. He creates a payment link for a note (UTXO) and broadcasts the payment to this link to the network’s validators. If newly created UTXO satisfies rules for normal send, mint or burn type of transaction, the network validators confirm the payment as valid and add its corresponding commitments to the ZK Ledger chain. It is the purpose of the future research to evaluate potential options for note contents and corresponding rules. For example, a transitory key may be handed over along with a nullifier inside the note or alternatively note could commit to a key that was used for creating Bob’s backup address, i.e. when handing over secret in the payment link automatically invalidates previous timeout spending branch. This “InputNote” created by Bob has to be handed over to the recipient (Carol).

As mentioned, in the ZK Ledger, the sender (Bob) has created a note to transfer funds to the recipient (Carol). To mirror this transfer on Bitcoin, we need to transfer ownership of the spending key in the onchain bitcoin multisig from Bob to Carol if it wasn’t transferred via note. To do this, the sender (Bob) works with the operator to create a new pre-signed exit transaction for the new recipient (Carol) as depicted in Figure n (“blind statechains”). This procedure establishes the recipient (Carol) as the new current owner. As with all statechain-like designs, the timelock on this new exit transaction expires before the timelock of the previous exit transaction belonging to Bob. After this procedure is complete, the sender (Bob) communicates the secret key for X the recipient (Carol).

Upon downloading the link, the recipient can then claim the note and create new UTXO through a second ZK Ledger transaction. Carol does this and broadcasts said transaction to the networks’ validators who confirm their claim. A second commitment is added to the chain, finalizing the transfer within the ZK Ledger. This transfer is also reflected on bitcoin via the statechain-like ownership transfer procedure. Carol now holds a note within the ZK Ledger and a copy of a presigned transaction for the 2-2 multisig on bitcoin for optional unilateral exit.

In this protocol, the transfer of ownership is proven valid by a zkSNARK proof which the recipient validates client-side. The zkSNARK proves, per some zero-knowledge circuit logic, that the sender constructed a presigned payout transaction with a lower timelock than the one they previously received and that the note they’ve received on the ZK Ledger is valid and has an available output commitment. To prove that the sender has not previously transferred ownership to another user within the ZK Ledger, the receiver checks the generated output commitment against all commitments through its ZK Ledger full node or a third-party node. If the output commitment is unique, and the proof is valid, the recipient validates the ZK proof and finalizes the transaction. After verifying the transfer’s validity, the recipient locally stores the backup transaction, but does not broadcast it. The recipient is now the current owner of the statecoin. Note, that in our protocol, a history of backup transactions is not passed to the recipient. The zkSNARK that they validate locally is proof that the note they are receiving in the ZK Ledger is valid, and that this transfer of ownership is updating the bitcoin multisig’s view of said transaction.

A successful verification means that that locktime committing to the output hash was accepted along with his Shielded note secret key hash in private inputs of corresponding SNARK proof and it was lower than locktime in previous transactions.

#### Note structure

Within the example above, you may note that the sender is responsible for two operations: 1) create a payment in the ZK Ledger and engage in a key reassignment procedure with the operator to transfer ownership of the bitcoin multisig to a recipient. This is necessary, but user complexity can likely be mitigated through application UI abstraction.For the receiver, we can construct the ZK Ledger’s note structure and corresponding ZK-Circuits to verify the validity of a transfer pertaining to funds in the bitcoin multisig, offchain. It is important for Receiver’s security that upon receipt of the pre-signed backup transaction, the Receiver must obtain a finalized transaction that spends from an existing Bitcoin UTXO and contains a timelock that is lower than all previously generated backup transactions.

In previous protocols, the timelock decrement could be enforced via verification of the entire history of statecoin transfers client side. In our protocol, we address that in a way that allows compressed representation via SNARK while providing privacy as a by-product.

![image|624x429](upload://mJKzT9W982RkvarLKQV7KWoD6dh.png)

Updated UTXO note structure

When included into note structure, the timelock validation becomes a part of the dedicated UTXO circuit which validates privately if Receiver’s timelock in the output satisfies a condition to be less than timelock in the input with given delta. The timelock and the authentication key become two entities that connect a shielded pool note with Bitcoin. The authentication key may be a valid libsecp256k1 private  key that corresponds to the backup transaction and is not known to the next receiver until signing of the new backup is complete.

In the less private setup after obtaining the payment link with the previous transitory key along with private key to backup address, the receiver includes them as input into a new shielded transaction, validates it, generates proofs and sends the new transaction to the Operator. However, albeit less private (details are shared with counterparty), this way may be considered also as most secure since the next shielded coin owner also has a private key from the previous backup transaction. This disincentivizes the previous owner from broadcasting an old backup.

In a more private setup the Receiver can only obtain Sender’s public key in case of an attempt to spend funding UTXO and because Receiver already has a transitory key after completion of the statecoin transfer, she may convince Operator to punish dishonest sender and co-sign to the new “recovery” address of the owner of valid Shielded note.

# Crime and Punishment

In the current design, we propose a more straightforward way to facilitate backup UTXO transfer. However, it still allows Spenders to *attempt* a double-spend. In a scenario where the Spender hands over payment links with the note private key encoded, broadcasting an old backup does not make sense since the Receiver may know the private key for the backup output from the link (i.e. it becomes a way to revoke backup). Alternatively, when the Sender publishes their previous back up transaction onchain, and the Receiver already has a backup transaction, the recipient reconstructs the commitment and expects confirmation in the ZK Ledger which only happens if the timelock in the Output was lower than the timelock in the Input. Here, double spend is impossible too, unless the Operator colludes and secretly co-signs a new transaction with lower timelock without submitting it into ZK ledger.

However, this situation may be detected by external observers and fraud is provable the same way it is done in the original statechains. When defrauded, the Receiver may obtain a transaction that spends funding input of the state coin, extract public key, amount and timelock and obtain commitment that should have been submitted into statechain ledger in the honest scenario. If the existence of the output commitment is not confirmed, it means that the operator colluded with dishonest Sender.

When the Operator entity is represented by FROST federation, each participant may track funding UTXO and verify if new spends correspond to shielded UTXO. Besides detecting potential collusion, this defines a unilateral exit scenario when UTXO must be burned.

# Fungibility of Notes in Shielded Pool

Backup transactions may lead to privacy leaks which will make all efforts on improving it on the statecoin ledger side fruitless. Therefore an important feature of the Operator should be blind signing of backup transactions similarly to the Mercury layer. However it also means that schemes with splitting of shielded coins denominations with simultaneous backing similar to Spark’s are likely impractical when joined with blind signing. In this situation we propose to preserve privacy benefits while addressing shielded coins’ fungibility via market based mechanics when circulating shielded notes may be merged and exchanged privately inside a shielded pool with support of pre-programmed circuits implementing dedicated smart contract functionality.

The transaction structure of the original ZK Rollup architecture assumes 2-input, 2-output transactions and allows merging and splitting notes with change by design. For notes of “statecoin” kind rules may define 1 padding “zero” note in the input and the output correspondingly and such simple rules could be enforced via dedicated UTXO circuit which could be programmable.

For an illustration, let’s consider an existing code snippet, a simple check for various types of notes: if note facilitates regular transfer, mint (gets into the pool) or burn (exits shielded pool).

```

    if (kind == 1) {

        //SEND

        assert(input_value == output_value, "Input and output totals do not match");

    } else if (kind == 2) {

        // MINT

        // Assert mint utxo is balanced:

        //   - \`output_value\` is checked above

        //   - \`input_value\` is checked as it must have previously been an output value

        //   - \`msg_value\` is checked above (but also using that to overflow would be detrimental to the

        //      attacker)

        assert(output_value == input_value + msg_value, "Mint output must match value message");

        // Assert mint hash

        assert(mint_hash == msg_hash, "Mint hash must match message");

        // Assert note kind

        assert(output_notes\[0\].kind == msg_note_kind, "Mint note kind must match message")

    } else if (kind == 3) {

        // BURN

        // Prevent frontrunning the txn and changing the evm address

        assert(pmessage4 == burn_addr, "messages\[4\] must match private input");

        // Assert burn hash

        assert(burn_hash == msg_hash, "Burn hash must match message");

        // Assert burn utxo is balanced:

        //   - \`output_value\` is checked above

        //   - \`input_value\` is checked as it must have previously been an output value

        //   - \`msg_value\` is checked above

        assert(input_value == output_value + msg_value, "Burn output must match value message");

        // Assert burn kind

        assert(input_notes\[0\].note.kind == msg_note_kind, "Burn note kind must match message")

    } else {

        assert(false, "Invalid kind");

    }
```

This way we could define more types and more checks. More importantly, we envision building a market place for peg-outs where whole pre-signed UTXO could be traded against shielded fungible tokens. Marked based approach assumes the existence of notes that may be detached from the corresponding backup transactions and therefore requiring less interactive process for regular private transfers.

# Nostr-based Relaying

[The original Moore’s and Gandhi work](https://polybase.github.io/zk-rollup/whitepaper.pdf) mentions “Encrypted Registry” which should help sending payments over the network when the Sender and Receiver are not online at the same time. During their absence, the transaction data could be stored in the Encrypted Registry, and this is an optional component of the network.

As mentioned earlier, using Nostr allows achieving a “push” payments user experience while not changing the existing method for sending coins. On a higher level, sending a payment to a selected Nostr public key specifies a resilient and sufficiently decentralized way to broadcast an encrypted transaction and possibly gain additional privacy because financial transactions will be relayed along with tons of different encrypted data, including regular text messages to this user.

Besides that, relaying a message broadly via several Nostr relays may allow duplication of the original message. Because Nostr is becoming a modular sophisticated protocol, the goal of *a sharded decentralised encrypted registry, where the encrypted data is split into chunks and blindly stored across multiple nodes* may be achieved independently from ZK Ledger development goals.

# Conclusion

Previous works around Bitcoin Statechains suggest a degree of freedom when it comes to implementing a ledger for tracking statecoin transfers and keeping the Operator accountable. We have attempted to replace implied UTXO-based ledger with the simplest possible flavor of the ZK Rollup that was previously developed for Ethereum. The eventual design becomes a rather zero-knowledge based smart contract execution layer for Bitcoin, with opportunities for unilateral exit and censorship resistance stemming from leveraging ZK technology stack.

The proposed architecture occupies an intermediate position between Statechains and ShieldedCSV or ZKCoins. The latter omits on/off ramping into shielded pools while the former lacks privacy. However, options for using BitVM as for bridging coins back and forth are mentioned both in ShieldedCSV paper and [ZKCoins](https://gist.github.com/RobinLinus/d036511015caea5a28514259a1bab119) as well. It makes the ZK execution layer an interesting opportunity to experiment with programmable market-based mechanics for on/off ramping into shielded pools. The closest example here might be the “pragmatic rollup” design implemented in [Signet](https://signet.sh/) and completely unknown to the Bitcoin community. Its design suggest instant swaps and atomicity for cross-chain transactions which when applied to ZK Statechain would mean “one in, one out” rule facilitated with the help of “privacy arbitrators” who are ready to post UTXO for exits or take ownership and possibly lift timelocks for entering participants.

**Acknowledgments**

I greatly appreciate the thorough review by [Janusz](https://x.com/januszg_) and Gus Gotoski. I also thank SuperTestnet and fiatjaf for their valuable feedback.

-------------------------

instagibbs | 2025-12-17 14:26:34 UTC | #2

Focusing on just transfers for a second, can you make a concrete claim about the privacy enhancements afforded this scheme?

-------------------------

evd0kim | 2025-12-17 21:17:23 UTC | #3

Let’s consider such example. I did some request by TxId to public node. It returned a proof, and some hashes. In the bigger picture we more info in backup transactions, so while proposed architecture does improve privacy in statechain like design, all trade-offs must be analyzed more thoroughly.



```
curl http://63.176.138.198:8091/v0/transactions/19094a66fc30f775d7cc3279862c3c166caa70061dd8a1cd51613e91fb442ca6 | jq
```
returned.

```json
{
“txn”: {
“proof”: {
“proof”: “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E2svfhQGZSoLG0Qsbw0UBUywDGkWnNyyT5L9qwKAjeq+cCgEBQ4AbgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA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AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA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”,
“public_inputs”: {
“input_commitments”: \[
“27f51a143ee66f749d726e0e869eab5c6f3e5c163c479a74687b913fc926144d”,
“0000000000000000000000000000000000000000000000000000000000000000”
\],
“output_commitments”: \[
“0f5a5d07cf62a090d9386e6fdec42c17314c29abf2808421a86861747863b7a7”,
“0000000000000000000000000000000000000000000000000000000000000000”
\],
“messages”: \[
“0000000000000000000000000000000000000000000000000000000000000001”,
“0000000000000000000000000000000000000000000000000000000000000000”,
“0000000000000000000000000000000000000000000000000000000000000000”,
“0000000000000000000000000000000000000000000000000000000000000000”,
“0000000000000000000000000000000000000000000000000000000000000000”
\]
}
},
“index_in_block”: 0,
“hash”: “19094a66fc30f775d7cc3279862c3c166caa70061dd8a1cd51613e91fb442ca6”,
“block_height”: 2123171,
“time”: 1765893819
}
}
```

-------------------------

