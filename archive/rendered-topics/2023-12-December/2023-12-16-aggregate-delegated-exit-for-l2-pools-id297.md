# Aggregate delegated exit for L2 pools

salvatoshi | 2023-12-16 08:08:35 UTC | #1

It's often been claimed that "covenants don't solve the L2 exit problem" because it's always not economical to exit a layer 2 system for a user whose balance is too low to cover for the fees.

In this post, I will argue that fraud proofs might provide an interesting solution to this problem.

**TL;DR:** a withdrawal protocol based on fraud proofs can allow many parties that are individually _too poor_ to withdraw their balance from a pooled UTXO to cooperate off-chain and perform a single, larger withdrawal.

## Intro: fraud proofs

Readers who are familiar with the concept of fraud proofs can safely skip this section.

The idea of *fraud proofs* is to replace a (possibly expensive) computation in a smart contract with just a claim that a computation would give a certain result.

As long as the interested parties in a smart contracts have a way of disproving fraudulent claims - and a protocol that punishes any fraud attempt, then frauds (and fraud proofs) are never expected to end up on chain.

The saving generally comes not only from avoiding an expensive operation in Script, but also by *not* putting the entire witness data that the computation would need: a commitment to it is generally sufficient, as long as the counterparties are able to independently compute the correct data themselves.

The trade-off of using protocols based on fraud proofs (optimistic protocols) in comparison to a full on-chain validation of the contract transition are:
- fraud proofs require a challenge period, in comparison to direct computations that are confirmed in a single transaction; moreover, some capital lockup is generally needed during the challenge period;
- fraud proof protocols rely on the assumption that miners are not actively censoring the transactions of the challenger.

The most commonly used optimistic protocol in bitcoin today is in fact a Lightning channel: the "justice transaction" can is a fraud proof. 

**Remark**. While [MATT](merkle.fun) admits a generic fraud proof protocol for arbitrary computations (that scales logarithmically on its computational cost), there are many special cases for which the fraud proof protocol is a lot simpler. Fraud proofs on statements about the leaves of a Merkle tree (for example "if I change the $i$-th leaf of the Merkle tree with root $R$, the root of the new Merkle tree is $R'$" have fairly straightforward fraud proof protocols).

## The optimistic pool setting

Imagine a layer 2 system where a single UTXO stores the state as a list of `user:balance` for $n$ users (where `user` is a public key, and `balance` is an amount in satoshis), and the operator (or a Musig2 of all the participants) publishes new states according to some rules.

An important assumption in what follows is that _all the users completely know the state of the contract - that is, each user's balance_.

If the operator messes up (for example, fraud is proven, or the operator becomes unresponsive and does not publish the next state), then the UTXO can be spent towards an _unwinding_ state, where the only action allowed is getting your money out of the pool.

There are multiple constructions that might fit this model (rollups, coinpools, etc.). In the following, we only focus on the _unwinding_ stage of such constructions: something went wrong, and all the users have to withdraw their coins.

## The "Unwind" contract

Once the panic button is pressed, the pool is in the Unwind contract (a UTXO with a covenant): the only allowed action is "TTMAR" (Take The Money And Run). This UTXO contains a vector commitment of all the `user:balance` pairs; that is, the root of the Merkle tree of the vector.

High value users can exit directly and instantly by proving their balances (with a Merkle proof) – but low value users can't, as even publishing the Merkle proof is more expensive than their balance!

## Optimistic withdrawal

If the Merkle proof is large, one can think of a cheaper way: withdraw with a fraud proof.

User Will who is withdrawing claims "my balance is X, and the remaining pool is Unwind(all-users-minus-Will)"; he additionally attaches 1 ₿ as a bond, and the money is locked for the challenge period. If he lied, whoever proves the fraud can take $(1+X)/2$ ₿ (half of Will's money), while $(1+X)/2$ ₿ is burned. Otherwise, at the end of the challenge period, Will can take out his money, and his bond, and produce the output Unwind(all-users-minus-Will)

This optimistic withdrawal might be cheaper as it does not require a Merkle proof, lowering the balance threshold for which withdrawals are economically feasible.

However, users who want to withdraw their 10000 sats are still out of luck.

## Optimistic aggregated withdrawal

The idea is to generalize the optimistic withdrawal to an arbitrary subset of pool users, and add an intermediary Ingrid.

A set $S$ of users of users wants to delegate to Ingrid the right to withdraw their "aggregate balance". Each of them signs a delegation message with their pubkey in the pool saying "I authorize Ingrid to withdraw my balance".

Ingrid then spends the Unwind UTXO, claiming "I'm authorized to withdraw a total of X on behalf of the users in $S$ = {s_0, s_1, ..., s_k}; the new state will be Unwind(all users, except the ones in $S$)" (note: only the _index_ of each user in $S$ is posted); the output also contains the Merkle root of all the delegations, and is timelocked to allow fraud proofs, and Ingrid also posts a juicy 10 ₿ bond.

If Ingrid is lying about the total aggregate balance of $S$, or about the new state of the contract (with the updated Merkle root), any user in the system can challenge Ingrid: since they know the full state of the system, a fraud proof protocol on the Merkle tree of the UTXO state will allow to expose the lie. upon winning the challenge, they will 5 ₿ from Ingrid's bond and burn the other 5 ₿; the withdrawal attempt is reversed.

Any user in $S$ can challenge Ingrid if they didn't sign a delegation (but they also have to put a bond, and if Ingrid proves they _did_ sign a delegation, Ingrid pockets half of their bond).

If there is no challenge, Ingrid takes the money of all the users in $S$, and gets back her bond.

Ingrid then can (custodially) return the funds in some other way - on lightning, on Liquid, in fiat, etc.

### Properties

Summing up, the properties of the system are such that:
- the intermediary can be chosen at any time _after_ the Unwind utxo is created; users can cooperate at any time with other users to find an intermediary who still didn't withdraw (and they have an incentive to do so);
- the cost for Ingrid to claim the money is (almost) constant (except for the list of indices that grows with the user indices, that is pretty small anyway);
- any unsuccessful fraud loses money;
- any set of users with sufficient aggregate balance can withdraw using the optimistic approach.

## Improvements

### Non-custodial Ingrid
For simplicity, we said that Ingrid "custodially" helps $S$ above, but one can build on top of this scheme: for example, the delegation could already specify some other UTXO(s) where the money has to go to once the challenge period is over – maybe a new pool with a different operators agreed upon by the users in $S$).

OP_CHECKTEMPLATEVERIFY or OP_TXHASH would provide an efficient to pre-program the future of the coins after the aggregated withdrawal.

### Constant withdrawal cost? (open problem)

In the above, the index of each user who is withdrawing is part of the initial aggregate withdrawal message. While that doesn't require too many bytes, it still make the transaction size increase linearly with the number of users withdrawing.

For a pool with $n$ users, one could use 1 bit per user (listing for each of the n users whether they are in $S$ or not), so the total cost of withdrawing is fixed at $n + O(1)$ bits; this might be good enough in practice.

It would be great to find a protocol with constant cost independent on $n$ or $|S|$.

### Parallel optimistic withdrawals? (open problem)

With an opcode like OP_CHECKCONTRACTVERIFY that allows cross-input introspection, the optimistic withdrawal protocol (and possible challenge protocol) could be performed on a separate UTXO instead of spending the pool UTXO; only at maturity (after the challenge period is over), both UTXOs could be spent together. However, it's not clear if there is a clean/easy way to make one withdrawal not interfere with other ones happening in parallel.

# Script requirements
- MATT (e.g. OP_CHECKCONTRACTVERIFY + OP_CAT)
- Amount introspection (equality checks should be enough)

**Nice-to-have**s:
- OP_CHECKSIGFROMSTACK to sign delegations (there are [workarounds](https://gist.github.com/bigspider/041ebd0842c0dcc74d8af087c1783b63#pre-signed-state-update-utxos) to do it without, but it's just awkward)
- 64 bit arithmetic

# Conclusions

There are many details to figure out, but the approach seems sound to me. I look forward to your comments.

-------------------------

