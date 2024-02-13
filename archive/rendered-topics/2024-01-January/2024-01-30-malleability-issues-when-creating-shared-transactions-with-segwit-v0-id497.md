# Malleability issues when creating shared transactions with segwit v0

t-bast | 2024-01-30 09:32:10 UTC | #1

This issue is already known among some people, but I couldn't find any article about it and it took me a while to figure out the details, so it's worth making a post for future reference.

Layer 2 contracts require building transactions that include inputs from multiple participants that don't trust each other. In most cases, participants pre-sign a refund transaction that can be used to get their funds back in case some of the other participants stop collaborating, to ensure their funds aren't locked forever in the contract. It is important that the `txid` of the shared transaction cannot be malleated, because the refund transaction would otherwise become useless, exposing users to blackmail to get their funds back.

We have this kind of usecase in lightning for interactive-tx transactions, used in dual-funding and splicing (https://github.com/lightning/bolts/pull/851). I will focus on that specific feature to explain the issue.

Each participant adds inputs to the transaction using the `tx_add_input` message. This message contains a `prevtx` field that contains the transaction the participant is spending from. The question is thus: why is that necessary? Can't we simply transmit the corresponding `outpoint`, its `amount` and `scriptPubKey`?

We unfortunately can't, because that would let participants use non-segwit inputs, which allows them to malleate the `txid`. Let's look at it in more details. Alice and Bob want to collaboratively fund a lightning channel without transmitting the whole `prevtx` in their `tx_add_input` messages:

1. Alice sends `tx_add_input` with `outpoint_A` and a p2wpkh `scriptPubKey`

2. Bob sends `tx_add_input` with `outpoint_B` and a p2wpkh `scriptPubKey`

3. Alice and Bob exchange signatures for the first commitment transaction spending that 2-inputs funding transaction

4. Bob sends the witness data for his input of the funding transaction

5. But Alice's input was actually using p2pkh instead of p2wpkh: she signs her input and publishes the funding transaction

At that point the published funding transaction has a different `txid` than the one that was used for the commitment transaction, since the p2pkh signature is included in the transaction hash. The commitment transactions are thus useless, and Bob's funds are stuck in a contract with Alice. Alice can then blackmail Bob if he wants to get his funds back.

Alice was able to perform this attack because Bob's signature doesn't commit to the `scriptPubKey`s of the other inputs of the transaction. Bob's signature was thus still valid, even though Alice lied about which `scriptPubKey` she was using.

This changes with segwit v1: [BIP 341](https://github.com/bitcoin/bips/blob/deae64bfd31f6938253c05392aa355bf6d7e7605/bip-0341.mediawiki#signature-validation-rules) added the `amount`s and `scriptPubKey`s of all inputs of the transaction to the signed hash. But it doesn't mean we can drop the `prevtx` in every case: participants are only protected **if they include one of their own segwit v1 input to the transaction**. In the example above, if Bob still uses a p2wpkh input, Alice could claim to use a segwit v1 input while still replacing it with a non-segwit input at broadcast time, and Bob would still be exposed.

That means that for lightning, we should only drop the `prevtx` field if:

- this is a splice transaction of a taproot channel (there is a shared input that uses taproot and requires signatures from both participants)

- or each participant adds at least one taproot input to the transaction

-------------------------

instagibbs | 2024-02-13 16:04:47 UTC | #2

https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-April/017801.html
https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2017-August/014843.html

two related emails on the subject for historical background

-------------------------

