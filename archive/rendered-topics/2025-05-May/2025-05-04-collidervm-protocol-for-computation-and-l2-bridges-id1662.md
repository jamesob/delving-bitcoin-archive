# ColliderVM protocol for computation and L2 bridges

victorkstarkware | 2025-05-04 12:08:34 UTC | #1

I would like to update you about an alternative protocol to BitVM that we have been working on, which we call ColliderVM. It uses the same trust assumptions as BitVM but without needing fraud proofs. By that it is eliminating the need for a fraud-proof time window. All in all, some additional work is needed to make this protocol practical (on which I elaborate below).

Perhaps some ideas here could also be of independent interest.

Here is the link to the paper:

https://eprint.iacr.org/2025/591

(Please note that we erroneously used an incorrect estimate for BLAKE3 script size, but this will be fixed in the next version which is coming in a few days):

We’ve also made a video explaining this protocol

[https://starkware.co/blog/avihu-levy-bitcoin-horizons-from-op_cat-to-covenants-hong-kong/
](https://starkware.co/blog/avihu-levy-bitcoin-horizons-from-op_cat-to-covenants-hong-kong/)

**What is ColliderVM**

ColliderVM is a protocol for stateful computation on Bitcoin, meaning the computation can persist across multiple transactions and user interactions. This is impossible to do directly in Bitcoin script without something like covenants.

In the paper, we also use it to sketch a trustless bridge design to a “layer two network”, with the purpose of scaling Bitcoin with a rollup-like design and enabling more functionality and privacy.

This protocol potentially allows STARK verification to be executed on top of it, which might not be viable to do on top of BitVM due to the long proofs used by STARKs.

**Stateful computation**

Stateful computation, as loosely defined above, can be split into two components:
1. Logic persistence

2. Data persistence

Logic persistence is about having Bitcoin locking scripts executed in a certain order or logical flow onchain. For example, suppose that tx1 implements locking script f1, and similarly, tx2 implements f2, and tx3 implements f3. Then, for “logic persistence” we want to make sure the order of execution onchain is

f1 -> f2 -> f3

Data persistence is about having some way to pass information between transactions (in the most simple case, think of some global memory storage). With logic persistence alone you can only guarantee transaction execution in some order, but there is nothing connecting the individual witnesses of the locking scripts involved. Data persistence gives you this capability.

In the most simplistic case, consider the situation where we would like f1,f2,f3 to be executed on the same witness x. This means that if tx3 was spent, it is necessarily the case that

f1(x)=f2(x)=f3(x)=1 (i.e. a single witness x was accepted by all 3 locking scripts)

**Why stateful computation might be desirable**

Two important instances are:

1. Long computation that doesn’t fit into Bitcoin’s 4MB script size limitation or the 1000 stack element limitation
2. A computation that is invoked across multiple points in time

We are particularly interested in building smart contracts and bridges to L2 networks on top of Bitcoin with this.

**Common solutions**

For perspective, I list here some common solutions for stateful computation on Bitcoin:

1. Multisig (t-out-of-n): This is the simplest (non-)solution. We trust a committee of n entities, such that as long as t of them sign the UTXO, it evolves to the next state. It achieves stateful computation trivially but is basically a centralized solution (unless n is very large).

More concretely, there is a tradeoff here between safety and liveness. If t is large, the multisig is more secure but has a liveness risk, and, conversely, for small t, the protocol is likely to be live but has a greater safety risk.

2. (Strong) covenants: As long as covenants are strong enough to achieve both logic and data persistence, we could get stateful computation without additional security assumptions. You can do it with OP_CAT as I discuss here
[https://delvingbitcoin.org/t/the-path-to-general-computation-on-bitcoin-with-op-cat/1106
](https://delvingbitcoin.org/t/the-path-to-general-computation-on-bitcoin-with-op-cat/1106)Possibly this is also doable with other proposed opcodes or a combination of them (as in https://en.bitcoin.it/wiki/Covenants_support).

We believe this is the desirable approach for stateful computation on Bitcoin, but it requires a soft fork and cannot be readily deployed.

3. BitVM: This approach sits somewhere in the middle in terms of trust assumptions, being a 1-out-of-n system. In other words, as long as one entity is honest, the protocol is guaranteed both safety and liveness. This is a desirable property because it allows an operator who trusts only themselves to make sure the funds are always safe and operable.

By analogy, covenants can be seen as a “0-out-of-n” solution here.

**Our security model**

We follow basically the same model as BitVM2, with two groups of entities

* n signers
* m operators

Safety is guaranteed if 1-out-of-n signers is honest, and liveness is guaranteed if 1-out-of-m operators is honest. These are the only assumptions we require, while BitVM2 also requires onlookers who will defraud faulty execution by the operator.

Hence, in a BitVM2-based L2 bridge, an operator needs to pay the user out of pocket when serving a withdrawal. Then, after the fraud proof time window closes, the operator gets reimbursed. In ColliderVM, since no fraud proofs are involved, the operator can get reimbursed immediately. This lifts the burden on the operator of staking high amounts of collateral.

We consider this feature a main selling point of our construction.

**How BitVM achieves stateful computation**

Logic persistence is achieved through presigning transactions by the n signers. This way, as long as at least one signer refuses to sign transactions that don’t follow the template, we can force a certain flow of transactions (as the txid of a previous transaction is encoded in one of the inputs of the next transaction)

To achieve data persistence the clever trick by Robin Linus is to use one-time signatures. Consider the simpler Lamport version:

Two seeds s0 and s1 are drawn and hashed as H(s0) and H(s1). We identify the “memory location” of a bit with H(s0) and H(s1). To set the value of the bit [H(s0),H(s1)] to b=0,1, we reveal the seed sb. If both s0 and s1 are known, it means the operator was malicious and needs to be slashed. Hence, this approach inherently requires fraud proofs to work.

**How ColliderVM achieves stateful computation**

For logic persistence we similarly use presigned transaction templates.

For data persistence, we replace the fraud proof-based one-time signature component with a hash collision-based one, somewhat reminiscent of ColliderScript (https://eprint.iacr.org/2024/1802), hence the name.

The basic idea is as follows: Suppose that instead of a single flow f1 -> f2 -> f3, we have, for some set D, |D| copies of such a flow. Each flow encodes a single element d in D, that is

(f1,d) -> (f2,d) -> (f3,d)

Then, if we had a lot of flows |D|, or the size of the witness x was small, we could just check that x=d in the execution of a particular flow. By the construction of the flows, it is guaranteed that the same witness x (which equals d) is computed on in all 3 locking scripts.

Because the size of the witness x can be large, we instead use some kind of hash function F (with short digest) and check that F(x,nonce)=d for a given x and some nonce. Note that for our approach to be viable we need F to be efficient to implement over 4 byte elements.

Depending on the parameters (see paper for details), we get the following complexity gap:

* An honest operator, who wishes to commit to data x, needs to find a preimage to a hash d in D
* A malicious operator, who wishes to commit to conflicting data x1 and x2, needs to find two colliding preimages to the same hash d in D

This separation underpins the security of our construction.

**Good properties of ColliderVM (and the L2 bridge you can build with it)**

ColliderVM enjoys notable attractive benefits, including:

* The aforementioned capital efficiency
* During deposit, interaction with an operator is not required
* A very simple presigned transaction flow (making analysis and implementation easier)
* Saving the amount of data needed to be passed across transactions (more details in the paper)

There are also other benefits we list in the paper.

**Bad properties of ColliderVM (and the L2 bridge you can build with it)**

A main drawback of ColliderVM is its overall cost (see the paper for details):

* Our computation is validity-based, hence all of it is present onchain. Thus, we require a hash function with a cheap Bitcoin script implementation
* Per deposit, we require signers to sign a moderate amount of (Schnorr) signatures, which the operators need to store
* To process a withdrawal, an operator needs to evaluate a moderate amount of hashes to find a hash preimage

**Future research**

We believe that our protocol is far from optimal, and this approach is fruitful for enabling new applications on top of Bitcoin. In particular, we identify the following avenues for further research (more details in the paper):

* Finding Bitcoin-friendly hash function implementations.
* Optimizing STARK verifier implementations on top of Bitcoin.
* Reducing the overall work of the signers.
* Reducing the amount of data needed to be stored on a hard drive.
* Increasing the computational gap between honest and malicious operators.

-------------------------

