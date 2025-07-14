# Building Secure and Watchtower-efficient Bitcoin Payment Channels with BitVMX

SergioDemianLerner | 2025-07-14 12:23:32 UTC | #1

Hi! 
I'm copypasting here an article I wrote about how to create Watchtower-efficient payment channels with BitVMX.
The original article is here: https://bitvmx.org/knowledge/building-secure-and-watchtower-efficient-bitcoin-payment-channels-with-bitvmx.

## Intro

This article presents a sample payment channel that uses to support efficient revocations. It can be easily extended to function as a state channel and support Hash-Time Locked Contracts (HTLCs) for lightning-like routing. However, we aim to keep it simple so that it serves as a clear example of BitVMX’s capabilities and how it can be integrated as a subcomponent of other Bitcoin protocols. In this example, BitVMX is used to handle state revocations and enable efficient watchtowers that store only O(1) data per payment channel.



Each state of our toy payment channel has a sequence number *i*, which increments with every state update. When the parties agree to move to the next state (e.g., rebalance the shared account), they must co-sign three transactions. They can use a multisig or MuSig2 scheme to reduce the on-chain footprint. Then, they create two revocation messages—one for each party—signed with Schnorr signatures. A revocation message contains the new state sequence number *i* and represents the message: "I revoke all states prior to *i*." In practice, the message is simply the number *i*, signed with a private key designated solely for this purpose.

When Alice wants to unilaterally close the channel, she can only do so by signing the sequence number *i* using a one-time signature (OTS) scheme. This sequence number defines the state that Alice claims should determine the final balance distribution between her and Bob. If Alice uses an outdated state—say she signs *j*, with *j < i*—a dispute mechanism is triggered to penalize her. Bob simply needs to present the OTS-signed value j and the Schnorr-signed message “I revoke all states prior to *i*,” which Alice previously issued and which Bob has stored. A Bitcoin script verifies the OTS signature, while a RISC-V program executed via BitVMX verifies the Schnorr signature and checks whether *j < i*. If this condition holds, Bob wins the dispute and can claim all funds in the channel.

---

## Transaction DAG

The figure below illustrates the transaction directed acyclic graph (DAG) created by the protocol. Only the *SetupTx* is published on-chain; other transactions are published only as needed.

* **Violet boxes**: Transactions published by Alice
* **Pink boxes**: Transactions published by Bob
* **Yellow boxes**: A sequence of transactions issued alternatively by Alice, Bob, or their watchtowers
* **Orange boxes**: Transactions re-created with every state update
* **Gray box**: A transaction issued by a watchtower
* **Green box**: The initial funding transaction, which either owner can issue

![](upload://5IuINqxtg1Dauh381UTGjbPZH7S.jpeg)

The transaction Direct Acyclic Graph (DAG) for a Simple Payment Channel

When Alice publishes *StartTx(j)*, she commits to the sequence number *j*. Either Alice or Bob can then OTS-sign the value *j* in *AssertTx(j)*. Normally, Alice signs this transaction. However, we must consider whether Bob could disrupt the protocol by signing it himself. If Bob detects that *j* is not the latest state *i*, he should refrain from signing *AssertTx(j)*—he lacks Alice’s signed sequence number for *j* and thus cannot initiate a valid dispute. If *j = i*, Bob can safely sign and publish *AssertTx(i)*, which won’t affect the protocol’s integrity.

## Setup

There are two main parties in this protocol: Alice and Bob, referred to as the **owners** of the channel. Optional auxiliary parties, *Wa* and *Wb*, act as watchtowers for Alice and Bob, respectively.

### Initialization

1. Each owner chooses a hidden pre-image (*Pa* by Alice and *Pb* by Bob) and sends the other the corresponding hash (*Ha = H(Pa)* and *Hb = H(Pb)*).
2. Each owner generates an OTS key pair (pair *A1* for Alice and *B1* for *Bob*) for signing sequence values *i*, using 32-bit unsigned integers to allow up to 4 billion updates. Public keys are exchanged.
3. Both parties construct a DAG of co-signed transactions, including:

  * *SetupTx*
  * BitVMX Dispute Channels
  * *AliceWinsTx()*, *BobWinsTx()*
  * *TimerForBobTx*, *TimerForAliceTx*
  * *TimeoutForAliceTx*, *TimeoutForBobTx*
4. They create the initial transactions: *PayTx(0)*, *AssertTx(0)*, and *StartTx(0)*. These define an initial exit path where each owner can withdraw their initial deposit. No revocation messages exist at this point, so no challenges are possible.
5. Each owner generates another OTS key pair (*A2* by Alice, *B2* by Bob) for signing inputs to dispute channels.
6. If watchtowers are used, they provide their OTS public keys to the owners, who create additional dispute channels that only the watchtower can sign.

---

## Advancing to the Next State

When a payment occurs (e.g., Alice pays Bob), both parties must advance to the next state and revoke the previous one. Without loss of generality, let’s assume Alice is paying Bob. Bob has an incentive to complete the protocol.

1. **Alice and Bob sign** *PayTx(t)* and *AssertTx(i)*, exchanging signatures.
2. **Bob signs** *StartTx(t)* and sends it to Alice. Now Alice can continue with either state, but Bob can only use the old state. Since he received funds, Bob is motivated to proceed.
3. **Alice gives Bob** a revocation message *Ma(i)*, which is a Schnorr signature on *i*, representing: “I, Alice, revoke all states prior to *i*.” Now Alice can only use the new state.
4. **Alice co-signs** *StartTx(t)* and sends it to Bob. Alice can only use the new state; Bob could use either, but the new state benefits him more.
5. **Bob forwards** Alice’s revocation to his watchtower and issues his own revocation message *Mb(i)* to Alice. At this point, neither party can safely use the old state.

---

## Liveness Guarantees During State Transitions

To prevent a party from stalling the protocol mid-update, the channel includes a timeout mechanism. If Alice issues *StartTx(j)* but delays or refuses to issue the corresponding *AssertTx(j)* in a timely manner, Bob can respond by publishing *TimerForAliceTx*. This transaction initiates a countdown, giving Alice a limited window to publish *AssertTx(j)*. If Alice fails to do so before the timer expires, Bob may publish *TimeoutForAliceTx*, which awards him all funds in the channel as a penalty for Alice’s non-cooperation.

The timer is triggered using the preimage *Pa*, which Alice reveals when she publishes *StartTx(j)*. This ensures that only Alice can start this process, and only Bob can enforce the timeout based on her actions.

---

## Closing the Channel

If both parties agree to close the channel, they co-sign a payment transaction that spends directly from the SetupTx, distributing the funds accordingly. If funds remain in intermediate connector outputs (used for pre-paying fees or managing dust outputs), they can also be collected and shared.

If one party becomes uncooperative, the other performs a **unilateral close**:

* Publish the latest *StartTx* and *AssertTx*
* Wait for a potential dispute
* Publish *PayTx*, which is timelocked to allow dispute resolution

The *PayTx* timeout can be shortened when no dispute occurs, it doesn’t need to match the dispute worse case time, but this complexity is omitted here to keep the transaction DAG simple.

---

## Watchtowers

Watchtowers receive periodic revocation messages *M(i)* for a given payment channel, identified by a monitoring ID (*MoId*). The *MoId* is derived from the channel’s funding transaction ID and the owner’s public key. If the same watchtower monitors both parties, each will have a distinct *MoId*.

When a watchtower receives a new revocation *M(i)*, it can safely discard all prior revocations for the same *MoId*. This is a significant improvement over Lightning Network watchtowers, which must store a revocation key for every state update. Thanks to BitVMX’s ability to verify complex logic off-chain, revocations become more storage-efficient and secure.

---

## HTLCs

(This section is available in the original article since it has more images and this forum restricts to 1 image per post)

---

## Summary

This article introduced a sample Bitcoin payment channel that leverages BitVMX to enable efficient state revocations and support for compact watchtower implementations. By encoding state revocation logic through Schnorr-signed messages and verifying them using BitVMX, the protocol achieves robust dispute resolution with minimal on-chain data. The payment channel uses a directed acyclic graph (DAG) of transactions, with one-time signatures (OTS) enforcing state commitments. Watchtowers benefit from an O(1) storage requirement by tracking only the latest revocation message per channel, improving upon the standard Lightning Network approach. We also show how to add HTLCs, PTLCs and other new locking functions. While the design is simplified for clarity, it highlights the extensibility of BitVMX in constructing secure and efficient off-chain Bitcoin protocols.

-------------------------

