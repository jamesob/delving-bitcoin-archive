# Conditional Message Transfer Contract To Solve Jamming

ariard | 2026-08-07 02:30:12 UTC | #1

**Abstract**. In this post, we present the prolegomenas for a formalization to a new mitigation paradigm of the channel jamming, a cheap of denial-of-service attack affecting the Lightning Network.

Our proposed mitigation relies on leveraging the bitcoin script capabilities, in its present state, to allow channel counterparties to unequivocally agree off-chain that a deterministic message has been previously exchanged or not among them at a given blockchain time, a practical universal clock.

This proof of message transfer constitutes a wider solution to the channel jamming attack, where counterparties should be able to decide for how long a HTLC or PTLC has been withheld over a channel and to charge fees in consequence.

**Introduction**. A brief informal re-statement of the problem.

"*The holy grail? is indeed charging fees as a function of the time the HTLC was held. As for now, we are not aware of a reasonable way to do this. There is no universal clock, and there is no way for me to prove that a message was sent to you, and you decided to pretend you didn't. It can easily happen that the fee for a two-week unresolved HTLC is higher than the fee for a quickly resolving one*".

**Conditions of validity**. A solution is valid if at the end of the CLTV expiry for the HTLC, one of the following state is reached:

* Bob revealed a preimage at time T and paid the withhold fee corresponding to time T
* Alice failed to be offline to receive Bob's preimage transfer
* Bob failed to deliver the preimage or has been offline

For the two last state, either Alice wins all the withhold fee or Bob wins all the withhold fee.

**The solution**. The solution is articulated around a serie of special scripts leveraging adaptor signatures and timelocks and an off-chain exchange of messages.

We start by the presentation of the conditional message transfer contract (a CMTC). In this post, we present an on-chain version and we leave it to future work how to uplift it in an off-chain channel, as additional script in the `tapscript_root`, the difficulty is not there.

The channel counterparties must agree on a discrete temporal window, for each temporal point in this window they pickup an "oracle-time" adaptor point.

The execution of the contract is conditional on multiple and alternative proof paths:

* the "message transfer success" proof path
* the "liveliness challenge" proof path
* the "message transfer failure" proof path

The "message transfer success" is used if the preimage has been transferred from Bob to Alice, and this reception has been cryptographically acknowledged by Alice, Alice and Bob can agree on a withhold fee corresponding to the delivery time.

The "liveliness challenge" is used if Alice has been offline, and therefore Alice has not been able to counter-sign the preimage or scalar delivery, whatever the reason. With a 2 stage process, Bob can exit the CMTC and get back all the money locked for the withhold fee.

The "message transfer failure" is used if Bob has failed the message transfer or has been offline, whatever the reason. With a 2 stage process, Alice can exit the CMTC and get back all the money locked for the withhold fee.

This contract is encoded in a bitcoin script with the following tapscript form, per BIP342 semantic:

```
OP_IF
"proof of message transfer success"

<alice_pubkey> OP_CHECKSIG <bob_pubkey> OP_CHECKSIGADD <2> OP_NUMEQUALVERIFY

OP_ELSE

    OP_IF

            "liveliness challenge"

            <alice_pubkey> OP_CHECKSIG <bob_pubkey> OP_CHECKSIGADD <2> OP_NUMEQUALVERIFY <cltv_expiry> OP_CHECKLOCKTIMEVERIFY

    OP_ELSE

            "proof of message transfer failure"

            <alice_pubkey> OP_CHECKSIG <bob_pubkey> OP_CHECKSIGADD <2> OP_NUMEQUALVERIFY <cltv_expiry + safety_delay> OP_CHECKLOCKTIMEVERIFY

    OP_ENDIF

OP_ENDIF
```

The "message transfer success" path can be spent by an encrypted contract execution transaction, where each contract execution transaction is encrypted under a adaptor point, itself corresponding to a temporal point in the window.

Each CET has a payout corresponding to the withhold fee and each CET is locked by a double adaptor point. The first point a PTLC `z + y0` and the second point a Alice-as-time-oracle point.

The "liveliness challenge" path can be spent by a challenge transaction by Bob, and if this transaction is not challenged by Alice, after a timelock, Bob can exit the path with the withhold fee minus an equilibrium penalty fee.

The Alice response transaction is locked under an adaptor point for Bob-as-time-oracle point.

The "message transfer failure" path can be spent by a challenge transaction by Alice, and if this transaction is not challenged by Bob, after a timelock, Alice can exit the path with the withhold fee.

The Bob response transaction is locked under an an adaptor point for Alice-as-time-oracle point.

Once the CMTC has been setup among the two parties at init time I, every time Bob reaches by his local clock a temporal point in the window, he's asking Alice a signature of M="time T" to finalize the corresponding CET and provide first his decryption secret for his own "oracle time" point.

Alice should reply by a reveal of her own decryption secret key for her own "oracle time" point.

After learning the tweak `z + y0` for the PTLC, Bob can reveal it off-chain to Alice. If Alice refuses to settle the PTLC in the channel off-chain, Bob can claim the PTLC and use the CTMC path to get back in part the withhold fee.

If Bob never delivers the preimage, Alice can exit the CTMC based on the latest decryption secret revealed by him.

**The validity of the solution**. The solution assigns to each temporal point a withhold fee distribution.

Alice cannot know the "real" time when Bob has discover the decryption key for his adaptor point. Lack of answering the liveliness message means Bob is able to exit the contract with the most advantageous withhold fee position.

Therefore, incentive-wise, she should engage in the liveliness challenge for each temporal point in the window, as such earning back chunks of the withhold fee.

If Bob fails to kickstart the liveliness challenge protocol, he will be at loss of the equilibrum penalty fee encumbered on the "liveliness challenge" path.

The conditions of validity of this solution deserves to be further analyzed on its cryptographic correctness and cryptoeconomic equilibrium.

**Generalization**. This is left as an open research question if a variant of this protocol can be used to solve more elegantly and in a griefing-free approach [the 3-party DLC transfer problem](https://web.archive.org/web/20241005213811/https://suredbits.com/transferring-discreet-log-contracts/), and beyond other decentralized financed bitcoin contract.

**Conclusion**. This post is a first formalization [on the idea](https://groups.google.com/g/bitcoindev/c/HxvRfLToCho) previously presented on the mailing list to solve channel jamming with a ping-pong protocol. After more brainstorming, it turns out that simple version might not necessitate blind signature, an oblivious protocol or garbled circuit.

-------------------------

AdamISZ | 2026-08-08 15:37:40 UTC | #2

Very interesting line of thinking!

I'm trying to grok the specifics, a couple of initial questions:

* We need a shared clock to be able to create a protocol that punishes non-response (the classic ack - ack .. infinite regression problem). So we use onchain/offchain contracts as a proxy, where we can fallback to clauses that say "such and such happens after a timelock" using the bitcoin blockchain's clock. Would you agree with that? Does this imply that this can only work on relatively slower timescales?
* Most of the description here doesn't seem to address the problem of onion routing, but then I thought: ah, maybe what he's saying is, message routing failure punishments can be chained the way that ordinary payments can: if Bob loses withhold fee to Alice, maybe he also gains the same from Carol? I think that makes sense, but I didn't see it described, maybe I missed it. And then it would make sense that you don't need crypto shenanigans in order to deal with routing privacy.

-------------------------

