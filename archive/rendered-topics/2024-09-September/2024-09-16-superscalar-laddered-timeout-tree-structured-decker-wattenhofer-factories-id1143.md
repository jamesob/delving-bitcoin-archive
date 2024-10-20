# SuperScalar: Laddered Timeout-Tree-Structured Decker-Wattenhofer Factories

ZmnSCPxj | 2024-09-16 20:08:19 UTC | #1

Subject: SuperScalar: Laddered Timeout-Tree-Structured Decker-Wattenhofer Factories

# Introduction

We introduce the LSP Last-Mile Problem:

* New users receiving their first ever Bitcoins over the Lightning Network must pay for incoming liquidity.  
  - The cost of blockchain operations to get that liquidity must be amortized across multiple new users to keep the cost of each one low.

In addition to the above problem, our solutions must also have the following constraints:

* We must ensure that the LSP cannot steal funds, i.e. it is not sufficient to have one-honest-member security assumption, unless every end-user can be their own single honest member.  
* We must do so without any blockchain consensus changes.  
  - The Bitcoin blockchain has ossified in practice. Due to the scheduled halvening every 4 years causing a sudden increase in Bitcoin price, leading to a sudden increase in interest in Bitcoin, large batches of new users join the set of entities with an interest in Bitcoin consensus. As the previous batch becomes convinced of some consensus change, the new batch enters Bitcoin, which must itself be convinced of the same consensus change. If a consensus change cannot achieve practical consensus within a time shorter than a halvening period, it will not ever happen, thus leading to ossification in practice (smaller and more easily-digestible changes may still push through, but more complex changes, like covenants, will never reach consensus).  
* We must be resilient against some or a few end-users being offline when the LSP has to reallocate funds.  
  - As the number of users sharing a single UTXO increases, the probability of one or more of them being offline increases. Thus, when scaling to large numbers of end-users, resilience against some of the end-users not being able to come online is a necessity.  
  - It turns out that software running on mobile phones can be made to come online via Android or iOS application notification mechanisms, thus in practice the onlineness of mobile clients can be reasonably high. Nevertheless, mobile phones may occassionally drop off network at times, so their uptime is still not as good as non-mobile devices. Thus, the mechanism should be resilient against a few users being offline.

All of the above constraints immediately rule out Ark and BitVM2 bridges. Both have a one-honest-member security assumption without covenants, and covenants will not achieve consensus in practice, as pointed out above (`OP_CTV` was largely finalized in 2020, and has not achieved consensus in 2024, thus will never achieve consensus due to exceeding the halvening period; `SIGHASH_NOINPUT` has even worse prognosis). Both can get around the one-honest-member assumption only if all end-users are simultaneously online at the moment at which an LSP needs to reallocate funds, which breaks the final constraint above.

In this writeup, I provide a construction, SuperScalar, which is effectively laddered timeout-tree-structured Decker-Wattenhofer channel factories.

# The Ingredients

First, I provide a hopefully gentle introduction to the three different components I combine to form this single construction:

1. Decker-Wattenhofer decrementing-`nSequence` offchain mechanisms.  
2. Timeout trees, particularly the variant that uses everyone-signs to emulate `OP_CTV`, which I call timeout-sig-trees.  
3. Laddering.

Feel free to skip sections you are already familiar with.

## Decker-Wattenhofer

The Decker-Wattenhofer decrementing-`nSequence` mechanism is an offchain cryptocurrency system which allows for a set of interested users to get consensus on some state change, without requiring that each state change be published on the blockchain layer (hence "offchain"). This is like the Poon-Dryja mechanism, however with the following differences:

* Number of parties:  
  - Poon-Dryja is strictly two-party.  
  - Decker-Wattenhofer decrementing-`nSequence` can have any number of parties.  
* Number of state changes:  
  - Poon-Dryja can theoretically have an infinite number of state changes (practical deployments, as in the BOLT specification of the Lightning Network, have a limit on number of state changes, but the limit is in the billions, thus practically unlimited).  
  - Decker-Wattenhofer decrementing-`nSequence` have a small number of state changes. Single unchained constructions would be able to provide a small number of state changes each (less than 100, and practically, 4 or 2 per chained mechanism). The typical proposal is to chain multiple such constructions together, effectively multiplying the number of available state changes for each chained construction (e.g. chaining 3 constructions with 4 state changes each will get 4 x 4 x 4 \= 64 state changes).  
* Size and convenience of unilateral closure:  
  - Poon-Dryja only requires a single commitment transaction for unilateral closure, as well as additional transactions to reclaim funds. The time delay until funds can be recovered is constant.  
  - Decker-Wattenhofer decrementing-`nSequence` have one kickoff transaction to begin the unilateral closure process, and one delayed transaction per layer. Increasing the number of state changes by chaining multiple layers of this construction will increase the number of additional transactions, so that the blockspace is practically O(log N) where N is the number of state changes supported. Each layer transaction has varying time delay until funds can be recovered.

Both Decker-Wattenhofer and Poon-Dryja are implementable in current Bitcoin without consensus changes.

Like Poon-Dryja, Decker-Wattenhofer decrementing-`nSequence` mechanisms have an onchain UTXO that serves as the "funding outpoint". The outpoint is a simple n-of-n multisignature of all signatories who sign off on state changes.

For a single-layer Decker-Wattenhofer decrementing-`nSequence` mechanism, there are two transactions:

1. A "kickoff" transaction, spending the funding outpoint, and outputting to a n-of-n multisignature.  
2. A "state" transaction, spending the output of the kickoff transaction, and which has an `nSequence` that requires a specific relative locktime. Its outputs are the state that is agreed upon by the consensus signatories.

For the initial state, the "state" transaction has an `nSequence` that is the maximum designed relative locktime. For instance, for a 4-state design, it would be reasonable to start with 432 blocks of relative delay.

When the state is changed, and a new state is agreed upon by the signatories, a new state transaction spending the kickoff output is created, with a smaller `nSequence` relative delay. For example, for a 4-state design:

| State Index  | Relative Locktime (Blocks) | Comment                             |
|--------------|----------------------------|-------------------------------------|  
| 0            | 432                        | Initial state                       |  
| 1            | 288                        |                                     |  
| 2            | 144                        |                                     |  
| 3            | 0                          | Final; mechanism can only be closed |

Thus, it is called a "decrementing-`nSequence`" mechanism; each time a new state is ratified by the signatories, the new state transaction has a smaller `nSequence`, until it reaches an `nSequence` encoding 0\.

Because the latest state has a lower relative locktime than any previous state, it can confirm earlier than earlier states. This mechanism ensures that, assuming the blockchain layer is not congested, the latest state is the one that gets confirmed in a unilateral exit case.

The difference in `nSequence` relative locktimes should be large enough that it is reasonable to assume that, even in congestion situations, the latest state can confirm before the previous state could potentially be confirmed. Thus, the example above uses a step of 144 blocks between states.

A single such mechanism, as demonstrated above, has a very limited number of state changes before it reaches the "final" state and has to be closed. Decker-Wattenhofer thus practically recommends chaining multiple such mechanisms. The first mechanism in the chain has a single output, which is an n-of-n of the signatories, which serves as the input to the next mechanism, and so on. Only the last mechanism has the state transaction have multiple outputs, the actual final state of the funds in the mechanism.

When chaining Decker-Wattenhofer mechanisms, only the "state" transactions need to be duplicated; the "state" transaction of the first in the chain serves as the "kickoff" transaction for the next mechanism in the chain. For example, if we chain 3 Decker-Wattenhofer mechanisms, we have a single "kickoff" transaction, plus 3 "state" transactions, for a total of 4 transactions for a unilateral close.

In effect, the "state" transaction of the previous mechanism is cut-through with the "kickoff" transaction of the next mechanism.

Chained Decker-Wattenhofer mechanisms work similarly to multi-digit countdown counters. Whenever a new state is ratified, the last chained mechanism \--- the one furthest from the funding transaction \--- gets decremented. However, if the last "digit" has counted down to 0, then the second-to-the-last one is decremented instead, and the last one is reset to the maximum `nSequence`. Similarly, if the second-to-the-last mechanism has counted down to 0, then the third-to-the-last is decremented and the succeeding mechanisms are reset to the maximum `nSequence`. And so on.

In effect, the last mechanism in the chain is the most likely to be changed whenever state is updated (it is always changed at each state update), while the first mechanism is the least likely to be changed at a state update.

```
Initial state
                         nSequence        nSequence
           +----+------+   +-----+------+   +-----+-----------+
funding -->|    |n-of-n|-->| 432 |n-of-n|-->| 432 |...state...|
           +----+------+   +-----+------+   +-----+-----------+
            kickoff tx       state tx          state tx

======>
                         nSequence        nSequence
           +----+------+   +-----+------+   +-----+-----------+
funding -->|    |n-of-n|-->| 432 |n-of-n|-->| 288 |...state...|
           +----+------+   +-----+------+   +-----+-----------+
            kickoff tx       state tx          state tx

======>
                         nSequence        nSequence
           +----+------+   +-----+------+   +-----+-----------+
funding -->|    |n-of-n|-->| 432 |n-of-n|-->| 144 |...state...|
           +----+------+   +-----+------+   +-----+-----------+
            kickoff tx       state tx          state tx

======>
                         nSequence        nSequence
           +----+------+   +-----+------+   +-----+-----------+
funding -->|    |n-of-n|-->| 432 |n-of-n|-->|  0  |...state...|
           +----+------+   +-----+------+   +-----+-----------+
            kickoff tx       state tx          state tx

======>
                         nSequence        nSequence
           +----+------+   +-----+------+   +-----+-----------+
funding -->|    |n-of-n|-->| 288 |n-of-n|-->| 432 |...state...|
           +----+------+   +-----+------+   +-----+-----------+
            kickoff tx       state tx          state tx
```

## Timeout Trees

Timeout trees are a mechanism that combines `OP_CTV`\-trees with a timeout.

Without `OP_CTV`, a variant can be created where the `OP_CTV` covenant is enforced by consensus signing of all participants. This variant (which I call timeout-sig-trees) is what I will focus on here. In addition, we consider the case where a singular LSP provides such a mechanism to its clients.

In timeout-sig-trees, an LSP `L` provides Lightning channels to a set of clients. A single confirmed onchain UTXO backs multiple channels to different clients.

On construction, the LSP creates a tree of transactions. Non-leaf nodes have outputs where the clients involved in the branch of that tree, plus the LSP, have an n-of-n multisignature. This is standard tree transaction structures, but timeout trees also add an alternative spending condition: the LSP can spend by itself after a particular timeout. The same alternative condition also exists on the funding outpoint.

For example, if the LSP `L` has 8 clients `A` to `H`, then it might form a timeout tree. The funding outpoint, which is confirmed onchain, would have the conditions:

1. `A & B &`...`& H & L`  
2. `L & CLTV`

Then the tree would look like this:

```
                                              +--+---+
                                              |  |A&L| LN channel
                                            +>|  +---+
                                            | |  |B&L| LN channel
                            +--+----------+ | +--+---+
                            |  |  (A&B&L) |-+ 
                            |  |or(L&CLTV)|   +--+---+
                          +>|  +----------+   |  |C&L| LN channel
          +--+----------+ | |  |  (C&D&L) |-->|  +---+
          |  |(A&..&D&L)| | |  |or(L&CLTV)|   |  |D&L| LN channel
          |  |or(L&CLTV)|-+ +--+----------+   +--+---+
funding-->|  +----------+
          |  |(E&..&H&L)|-+ +--+----------+   +--+---+
          |  |or(L&CLTV)| | |  |  (E&F&L) |   |  |E&L| LN channel
          +--+----------+ | |  |or(L&CLTV)|-->|  +---+
                          +>|  +----------+   |  |F&L| LN channel
                            |  |  (G&H&L) |   +--+---+
                            |  |or(L&CLTV)|-+
                            +--+----------+ | +--+---+
                                            | |  |G&L| LN channel
                                            +>|  +---+
                                              |  |H&L| LN channel
                                              +--+---+
```

The timeout condition forces all clients to come online before the timeout and exit the mechanism. Exit can be unilateral or with cooperation of the LSP.

In a cooperative exit, the client simply sends out all its funds in the channel hosted inside the timeout tree over Lightning, possibly using a swap service to get onchain funds, or to a new timeout tree with the same LSP, or to another LSP.

A unilateral exit simply means publishing the path from the root to their Lightning channel output, then unilaterally exiting the Lightning channel (expected to be a Poon-Dryja) as well.

The advantage of using a tree is that unilateral exit is small; it is only O(log N) transactions for a single client to exit, and most of the other clients can remain in the tree. If not using a tree, then a single client performing a unilateral exit would cause all clients to perform unilateral exits.

The advantage of the timeout condition is that it encourages all clients to exit simultaneously, hopefully cooperatively, and the LSP only needs to perform a simple single-input transaction to recover the funds from the construction, via the `L & CLTV` spending condition on the funding outpoint. Even if some clients also perform unilateral exit, much of the funds can be recovered via the `L & CLTV` spending conditions on each intermediate output.

## Laddering

Many financial institutions offer a kind of financial contract wherein a depositor puts funds into a contract, and cannot withdraw, even partially, until some specific future date, at which point the depositor is given the original funds, plus an interest payment. The contract is also non-transferable. Such contracts are known by various names:

* Certificate of Deposit (United States)  
* Guaranteed Investment Certificates (Canada)  
* Term Deposit or Time Deposit (other countries)

Such contracts are inflexible; as noted, it is impossible to withdraw or transfer the contract until the end of its term. However, savvy investors instead split up their investable funds into multiple such contracts, set up so that their termination dates are staggered by one month or one year to each other. This technique is called "laddering".

For example, an investor might have three such contracts, terminating in December 2024, December 2025, and December 2026\. On December 2024, the first contract terminates, and the investor may decide to withdraw part of the funds and re-invest the remaining in a new contract that terminates on December 2027, or to add more funds to invest in the new contract, or to start closing the ladder by not starting a new contract.

Laddering provides investors the ability to change the amount of investment, and to add or reduce their investment into these contracts, once a month or once a year, depending on the laddering. Thus, even though the base contracts are inflexible, laddering allows investors to regain a little bit of flexibility, while retaining the advantages of long-term certificates of deposit.

# The SuperScalar Mechanism

Laddered timeout-tree-structured Decker-Wattenhofer channel factories are simply the combination of the above three ingredients.

## Timeout-tree-structured Decker-Wattenhofer

First, let me demonstrate the combination of the first two ingredients; we shall add laddering in a separate subsection later.

Suppose an LSP, `L` has 8 clients, `A` to `H`. The funding outpoint then has the following two alternative spend conditions:

1. `A & B &`...`& H & L`  
2. `L & CLTV`

When setting up the mechanism, the LSP arrange the following transactions to be signed, with the funding transaction being an n-of-n of `A`..`H` and `L`:

```
                                                         nSequence
                                                           +---+---+
                                                           |   |A&L| LN channel
                                                           |   +---+
                                                       +-->|432|B&L| LN channel
                                                       |   |   +---+
                                                       |   |   | L |
                                                       |   +---+---+
                                                       |
                                                       |   +---+---+
                                                       |   |   |C&L| LN channel
                                            +--+-----+ |   |   +---+
                       nSequence            |  |A&B&L|-+ +>|432|D&L| LN channel
                         +---+----------+ +>|  +-----+   | |   +---+
                         |   |(A&..&D&L)| | |  |C&D&L|---+ |   | L |
         +--+---------+  |   |or(L&CLTV)|-+ +--+-----+     +---+---+
funding->|  |A&...&H&L|->|432+----------+
         +--+---------+  |   |(E&..&H&L)|-+ +--+-----+     +---+---+
          kickoff tx     |   |or(L&CLTV)| | |  |E&F&L|---+ |   |E&L| LN channel
                         +---+----------+ +>|  +-----+   | |   +---+
                          state tx          |  |G&H&L|-+ +>|432|F&L| LN channel
                                            +--+-----+ |   |   +---+
                                             kickoff   |   |   | L |
                                             tx        |   +---+---+
                                                       |
                                                       |   +---+---+
                                                       |   |   |G&L| LN channel
                                                       |   |   +---+
                                                       +-->|432|H&L| LN channel
                                                           |   +---+
                                                           |   | L |
                                                           +---+---+
                                                            state tx
```

Basically, the rules for building the tree from a set of clients, using a leaf-to-root construction order, are:

1. First, distribute the clients into leaf nodes, where multiple clients go into leaf nodes, according to arity (in the above example, arity is 2).  
   - Their outputs are a channel between the LSP `L` and the respective client, and also an additional fund that is owned only by `L`.  
2. Leaf nodes are always state transactions.  
   - They have decrementing `nSequence`.  
3. Create parent nodes to the trees being built, depending on the desired arity.  
   - Parents of state transactions are kickoff transactions.  
     - Their outputs are simply the owners in the respective child in an n-of-n.  
   - Parents of kickoff transactions are state transactions.  
     - State transaction outputs have an `or (L & CLTV)` alternate condition in addition to the n-of-n of the owners in that branch.  
     - State transaction inputs have decrementing `nSequence`.  
4. Repeat 3 until you get a single root node.  
5. If the resulting single root node is a state transaction, add a single-input single-output kickoff transaction. Otherwise, just use the root directly as the first kickoff transaction.  
   - This rule is true for the above example where the tree arity is 2 and the number of clients is 8\.  
   - If the number of clients in the example were increased to 16, then the number of transactions needed in a unilateral close would still be 4 transactions, plus the Poon-Dryja unilateral close transaction.

The reason for alternating kickoff transaction layers of the tree with state transaction layers of the tree will be given in a later subsection.

## `A` Needs Inbound Liquidity, Badly\!

Suppose `A` runs out of inbound liquidity in the `A` and `L` channel that is funded in the above tree structure. What can the LSP `L` do to provide inbound liquidity to `A`, without having to drop onchain?

The LSP can wake up `B` \--- and only `B` (presumably `A` is already awake if it is requesting for more inbound liquidity) \--- in order to update the leaf node containing the `A&L` and `B&L` channels. Because the leaf node only contains funds owned by `A`, `B`, and `L`, only those three participants need to be online; thus, the mechanism is more resilient to some participants not being online at the time that `A` needs additional inbound liquidity.

Of course, if `B` cannot come online, the LSP does have to fall back to alternate methods, such as onchain. This can lead to onchain fees needing to be paid, which the LSP would pass on to `A`.

Suppose `B` does come online. In that case, `L` can move funds from the `L`\-only output to the `A & L` channel, without requiring that clients `C` to `H` be online.

Of course, it is possible that the `L`\-only output has been depleted, or the final state change (i.e. `nSequence = 0`) has been reached. In that case, the LSP can move up the tree to the next higher state transaction, waking up more clients in an effort to move funds from the other `L`\-only outputs to `A`, and reset the `nSequence` of the nodes until the leaf nodes. This increases the number of clients that need to be online at that time, but does not necessarily require that *all* clients are online.

In effect, whenever some client runs out of inbound liquidity, the leafmost state transactions are the ones that are more likely to be updated. However, if the `L`\-only outputs have been depleted, then earlier state transactions would also need to be updated, to allow the LSP `L` to move liquidity from other leaf nodes to `A`.

## Tank, `A` Needs An Exit\!

Suppose that `A` decides to unilaterally exit. This may occur for any number of reasons; the important thing is that any arbitrary client of the LSP is capable of performing unilateral exit, and for the effect of unilateral exit to be minimized. This assurance prevents the LSP from being able to rug clients, and assures their sovereignty and control of their money.

In order for `A` to exit, it must first have the `A & L` channel funding outpoint confirmed, then finally exit the `A & L` channel using standard Poon-Dryja unilateral exit. Thus, `A` must publish the path to its channel below:

```
                                                         nSequence
                                                           +---+---+
                                                           |   |A&L| LN channel
                                                           |   +---+
                                                       +-->|432|B&L| LN channel
                                                       |   |   +---+
                                                       |   |   | L |
                                                       |   +---+---+
                                                       |    state tx
                                                       |
                                                       |
                                            +--+-----+ |
                       nSequence            |  |A&B&L|-+
                         +---+----------+ +>|  +-----+
                         |   |(A&..&D&L)| | |  |C&D&L|
         +--+---------+  |   |or(L&CLTV)|-+ +--+-----+
funding->|  |A&...&H&L|->|432+----------+    kickoff
         +--+---------+  |   |(E&..&H&L)|    tx
          kickoff tx     |   |or(L&CLTV)|
                         +---+----------+
                          state tx
```

However, we should also take note that any outputs of a confirmed kickoff transaction ***MUST*** be consumed by a state transaction that gets confirmed. This is because state transactions with smaller `nSequence` relative delays need to be confirmed first before those with larger `nSequence` relative delays, or else the Decker-Wattenhofer mechanism gets broken (older states may get confirmed instead).

Thus, in the case that `A` wants to exit, not only are the clients on the same leaf transaction (`B`) are inadvertently exited, but also sibling clients on the next higher level, `C` and `D`.

```
                                                         nSequence
                                                           +---+---+
                                                           |   |A&L| LN channel
                                                           |   +---+
                                                       +-->|432|B&L| LN channel
                                                       |   |   +---+
                                                       |   |   | L |
                                                       |   +---+---+
                                                       |
                                                       |   +---+---+
                                                       |   |   |C&L| LN channel
                                            +--+-----+ |   |   +---+
                       nSequence            |  |A&B&L|-+ +>|432|D&L| LN channel
                         +---+----------+ +>|  +-----+   | |   +---+
                         |   |(A&..&D&L)| | |  |C&D&L|---+ |   | L |
         +--+---------+  |   |or(L&CLTV)|-+ +--+-----+     +---+---+
funding->|  |A&...&H&L|->|432+----------+    kickoff        state tx
         +--+---------+  |   |(E&..&H&L)|    tx
          kickoff tx     |   |or(L&CLTV)|
                         +---+----------+
                          state tx
```

This also explains why our layers of nodes need to alternate between "state" transactions and "kickoff" transactions, unlike the plain Decker-Wattenhofer where only the very first transaction is the "kickoff". If we imitated that in our scheme, then if any single client needed to exit, all the downstream state transactions, which have varying `nSequence` relative delays, ***MUST*** be published also, or else the Decker-Wattenhofer mechanisms risks being broken. The layers of "kickoff" transactions serve as a backstop, ensuring that a single participant doing unilateral exit only publishes O(log N) transactions, and not the entire O(N) tree.

This is also the reason why the outputs of kickoff transactions do not need the timeout branch `L & CLTV`. A kickoff transaction output ***MUST*** be consumed by the latest state transaction for that output, and the state transactions would have the timeout branch anyway.

In the above example, `B`, `C`, and `D` can still transact HTLCs on their respective channels, but can no longer cheaply purchase additional inbound liquidity from `L`. Instead, `L` would need to splice in additional inbound liquidity onchain, which would be more expensive. (i.e. a single participant exiting does not cause *all* participants to exit; it still causes *some* other participants to partially exit, in that their channels remain open, but the ability to cheaply get inbound is lost)

However, the remaining clients `E` through `H` would still be able to purchase cheap inbound liquidity from inside the mechanism, as their part of the tree is not yet published onchain and can still be updated offchain. In addition, once the timeout period ends and the clients `E` through `H` have performed a cooperative exit over the Lightning Network (and have no more in-Lightning funds in their respective channels) then `L` can reap their output on the state transaction via the `L & CLTV` branch, in order to recycle those funds.

## Laddering

I now add the third ingredient.

From the point of view of the LSP `L`, the above mechanism is an investment. The hope of the LSP `L` is that it can earn a return on this investment, from various means, including:

* Lightning Network routing fees.  
* Selling of cheaper offchain liquidity.  
* Fees for maintenance of the overall mechanism.

In addition, because of the timeout branches, the LSP cannot easily recover its funds until the end of the term.

Thus, a single timeout-tree-structured Decker-Wattenhofer mechanism is very much like a term deposit, from the point of view of the LSP.

And as I pointed out earlier, savvy investors use laddering of multiple term deposit contracts in order to get a little more flexibility in how they allocate their funds.

The LSP itself can run multiple timeout-tree-structured Decker-Wattenhofer mechanisms, with different sets of clients, and with overlapping terms, as in a ladder of term deposits in traditional finance. As the term of one mechanism ends, the LSP can start a new mechanism, inviting the clients in the terminating mechanism to transfer their funds to the new mechanism, including charges for the transfer. Then the LSP can recover the funds from the terminating mechanism via the timeout branch, onchain. The LSP gets earnings from routing fees, selling offchain liquidity, and the privelege of transferring to a new mechanism, and those earnings remain in Lightning-locked funds.

For example, the LSP can run 30 such mechanisms, all expiring on different days. When one of the mechanisms is about to expire, the LSP can invite the clients on that mechanism into a new mechanism. The LSP funds the new mechanism from the mechanism that expires today, creating a new 30-day mechanism. Then, the clients can move their funds from the dying mechanism to the new mechanism. Once all clients have exited the dying mechanism, on the completion of the term, the LSP can simply claim the entire UTXO, resulting in a single output being spent.

In the concrete example below, the LSP has a ladder of 9 timeout-tree-structured Decker-Wattenhofer factories.  Each ladder has an "active period" of 7 days, and a "dying period" of 2 days.  During the dying period, clients can join one of the 2 factories built on the 2 days of the dying period, and transfer their funds to a new channel inside the new factory.  This allows clients a little leeway; if they miss the first day they can transfer, they have another chance on the succeeding day.  In actual deployments, I would mildly suggest an active period of 30 days and a dying period of 3 days; an LSP would then need to maintain 33 different factories at any time.  Ideally, there would be a single 1-input 1-output transaction per day, although if a client never comes online, the LSP would need to publish the path to its output, which increases the number of transactions needed onchain.

```
Legend: ===== Active Period
        ::::: Dying Period

Day | 01 | 02 | 03 | 04 | 05 | 06 | 07 | 08 | 09 | 10 | 11 | 12 | 13 | 14 | 15 | 16
       ===================================::::::::::   <---- each of this a factory
            ===================================::::::::::
                 ===================================::::::::::
                      ===================================::::::::::
                           ===================================::::::::::
                                ===================================::::::::::
                                     ===================================::::::::::
                                          ===================================::::::::::
               Client can move            ^    ===================================::::::::::
              funds from first -----------+----^    ===================================::::::::::
             factory to either of                   ^
                these two new                LSP uses the funds of the
                 factories                       first factory to
                                                build this factory
```

Unfortunately, due to a lack of covenants, clients need to come online on the specific times that the LSP is constructing the new factory.  If they miss the exact time, they need to try on the next day of the dying period; if they miss the last day of the dying period they ***MUST*** exit the mechanism, with all the costs implied by the exit.  `OP_CTV` would allow LSPs to preemptively add the client to the new factory without the client needing to be online immediately (the client can come online at *any* time within the dying period, unlike the non-covenants style where the client has to come at *one* of the specific times a channel is opened within the dying period).  However, this has the drawback that if the client actually *does* exit instead of going to the new factory, the LSP is forced to lock up its funds for the next active period, and since the client has left the LSP, will not be able to get a signature needed to reallocate funds inside the offchain mechanism, impacting neighboring clients in the tree (i.e. the *other* clients in the same tree cannot buy cheap liquidity offchain either, due to the LSP preemptively adding the client but the client exited the LSP and presumably wants nothing to do with the LSP or its service).

The requirement to move funds regularly from old factories to new ones also provides a convenient cadence for charging fees for managing liquidity to the client.

The longer the dying period, the more extra funds the LSP has to dedicate to the construction.  These funds need to earn, thus increasing the dying period requires increasing the fees the LSP charges to clients, due to expected return on investment.  This is a simple convenience-vs-cost consideration.

To reduce the chance of a client having to fall back to unilateral exit, the LSP can offer a mutual exit where the client swaps funds inside the mechanism for onchain funds using standard HTLCs, i.e. a perfectly normal offchain-to-onchain swap.  This allows a client to exit to onchain without having to publish the unilateral exit; unilateral exits cause more UTXOs to be pushed onchain, which increases the cost on the LSP to manage onchain funds once the factory dies after its dying period.

# Practical Considerations

Now that I have hopefully given you a conceptual grasp of the laddered timeout-tree-structured Decker-Wattenhofer channel factory construction, let us now turn to practical considerations for this mechanism.

## Why Take The `L`?

In the proposal, each leaf has an output that is owned only by the LSP `L`, from which new inbound liquidity to clients can be dynamically allocated.

This brings up the thought: can we remove the `L` output?

For example, suppose that all leaves consist only of the channels. So `A` and `B` would have a leaf transaction that contains the channels `A & L` and `B & L`.

Suppose that `A & L` runs out of inbound liquidity towards `A`, but the channel `B & L` has inbound liquidity. Then the LSP can move that liquidity from the channel with `B` to the channel with `A`.

The problem here is that this is not incentive-compatible. `B` has no incentive to actually participate in the signing of the new state, because it loses something valuable: inbound liquidity\!

Suppose that `A` and `B` are both users with high volume of Lightning payments in both directions. If `B` signs off on the loss of inbound liquidity, simply because it had a spike of outgoing payments, and then later it gets a spike of incoming payments, then `B` has to buy more inbound liquidity from the LSP. If `A` has already used up the inbound liquidity it already got from that change, then the LSP has to go up the tree to get more liquidity from other clients. This requires more other clients to come online, which increases the chance that one of those clients cannot come online, forcing an expensive onchain fallback to get more inbound liquidity for `B`. Obviously, the higher expenses will be passed by the LSP to its client.

Now, perhaps you might propose that the LSP can simply pay `B` for the inbound liquidity it loses.

The problem with that is that the inbound liquidity was purchased by `B` from the LSP in the first place. The LSP paying `B` for the inbound liquidity towards `B` is effectively a refund of already-sold product. As you can imagine, a refund is always a bad customer interaction; as a seller of inbound liquidity, the LSP wants that ideally 100% of sold inbound liquidity remains sold and never refunded, just as any seller wants to never have to refund already-sold goods.

In particular, if each unit of inbound liquidity has a price, and the price is the same for `A` and `B`, then any refund for the inbound liquidity coming from `B` would be the same price that `A` pays for the same inbound liquidity. This means that the LSP itself has no incentive to even set up this kind of mechanism, because it cannot earn from selling inbound liquidity to `A` if it has to refund the price of inbound liquidity of `B` for the same unit of inbound liquidity.

If LSP pays `B` less than the price it charges for reselling that inbound liquidity to `A`, so that it can actually earn from the difference, then it forces `B` into a bad economic partnership. If `B` later needs to rebuy liquidity, then if the LSP charges the same, higher price, `B` effectively loses money in refunding its liquidity and then rebuying it later. This is simply a zero-sum game which none of the participants can win.

Thus, the only way for the LSP to provide actual inbound liquidity is to lock up funds in an `L`\-only output, effectively the "sales stock" of liquidity.

The important part is that `B` would never participate in a scheme where it loses inbound liquidity.

An alternative is for the LSP to ***not*** sell inbound liquidity towards clients; i.e. instead of the model "LSP sells inbound liquidity, charges 0 LN routing fees to clients" we use the model "the LSP charges non-0 LN routing fees to clients, and determines where to point its liquidity".  The latter is less ideal, as the clients presumably have more information on when they need inbound liquidity; for instance, a merchant that has some sales promotion, or a new product on sale, can expect to have to need more inbound liquidity, and this information can be signalled by buying inbound liquidity explicitly from the LSP prior to the event occurring; the client can wait around for all fellow clients in its part of the tree to come online in that case.

## Incentivizing Onlineness

Even with the separate `L` output that serves as the "inbound liquidity stock for sale", as `B` is required to come online, `B` should still be compensated somehow.

The most straightforward is that, for simply participating, `B` can be paid some small fraction of the price that the LSP charges for selling liquidity to `A`.

Alternatively, the LSP can simply offer a small amount of free inbound liquidity to `B`. Both are largely equivalent, as inbound liquidity is valuable, but the advantage here is that `B` can pre-emptively get more inbound liquidity while both it and `A` are online. Later, `A` may fail to come online when `B` needs inbound liquidity, forcing `B` to fall back to expensive onchain operations to get inbound liquidity, so `B` may prefer to get a little free inbound liquidity now (when it is sure `A` is online) as opposed to later (when `A` might have gone offline).

## Client Grouping

It is likely that some clients are powered off at some regularly daily schedule (e.g. local nighttime).

The LSP can monitor the uptime of clients, and bin them according to what time of a 24-hour day they are most likely to be online. Then, when constructing trees for a new timeout-tree-structured Decker-Wattenhofer mechanism, the LSP can group clients with similar "most active" times together in the same leaf nodes and in adjacent leaf nodes.

This makes it more likely that if any particular client needs to get inbound liquidity, other clients in the same leaf node are also online, and even if the leaf node runs out of `L`\-only funds, nearby leaf nodes have clients that are also likely to be online.

In particular, if an LSP has clients globally, grouping them by timezone would be helpful, as clients near each other by timezone are more likely to be simultaneously online as well.

## Arity and Tree Structuring Decisions

The best arity for leaf nodes is 2, as this means that leaf nodes can be updated with only three online participants: the two clients on the leaf, and the LSP. This makes such operations more reliable.

Kickoff nodes may also have an arity of 1. Since all outputs of a kickoff transaction ***MUST*** be spent if the kickoff transaction is spent, this reduces the number of affected clients if one client wants to unilaterally exit. This also reduces the number of clients that have to be awoken in case a leaf node has run out of `L`\-only funds for funding liquidity.

Unfortunately, low arity implies greater tree height:

* Greater tree height means more transactions published onchain in a unilateral exit case.  
* Greater tree height means more `nSequence` relative locktime delays before funds can be recovered in a unilateral exit case.  
  - The relative locktime delay involved also forces HTLCs terminating at the client to have their minimum final CLTV delta (`min_final_cltv_expiry_delta` in BOLT11) be higher by the largest possible `nSequence` delay along the path to their channel than what the client would deem "safe" for itself. Thus, this delay is very important to keep low, as this delay is also the worst case that an HTLC on the public network ends up locking funds on unresolved HTLCs, reducing public network capacity.

To mitigate these:

* The tree can have low arity near the leaves, then increase the arity when building nodes a few levels away from the leaves.  
  - If a leaf update is unable to provide additional inbound liquidity to a client, then the LSP would need to "go up a level" and wake up more clients anyway.  
  - "Going up a level" means multiplying the number of clients that have to be online, proportionally to the node arity.  
  - If the group that needs to be awoken is big enough, the probability all of them come online is low enough that you could double or quadruple the size of that group with little impact \--- the chances all of them are online is low enough that you would not bet on it anyway and would just fall back to onchain.  
* Beyond a few layers away from the leaves, we could entirely remove state transactions (i.e. those with decrementing `nSequence`s). In effect, it would be a root timeout-sig-tree that backs multiple timeout-tree-structured Decker-Wattenhofer mechanisms, which itself backs actual channels to clients.  
  - Again, if the LSP has to actually "go up a level" by more than one or two state tx layers from the leaves, then the group of clients that need to be awoken can be large enough that it is very unlikely all of them are online anyway, so you may as well reduce the delays of unilateral delay by not having more than a few layers of state transactions.  
  - Transactions that are not state transactions do not have a relative timelock, thus would not cause additional time delays in exit.  
* The LSP can group together clients with high uptime and put them into higher-arity nodes.  
  - Such clients would get better service (they would be grouped with other high-uptime clients which would be likely to be online as well when they need inbound liquidity) and cheaper and shorter unilateral exits (higher arity implies lower tree height implies less transactions on unilateral exit).  
  - Low-uptime and new clients would have to get more arity 2 and arity 1 nodes, increasing the cost of their unilateral exits, but this also gives better isolation against their sibling clients being offline.

-------------------------

cryptoquick | 2024-09-16 21:37:03 UTC | #2

Apologies if you've already answered these, but the information is a bit dense, and I have a few questions to help me put this work into context... Say, practical takeaways.

Could one use case for this be to help people in developing nations receive small amounts of bitcoin instantly and with low transaction fees? I suppose the question is, can this work well for small payments as well as for large ones?

Additionally, how critical is liveness for trustlessness? Could this work on a mobile app for example?

What are the worst case scenarios, especially if the LSP is shut down by the authorities?

How much do you think this can scale with 1MB blocks? Does this essentially solve the billions of Lightning users metric without the need for changes to bitcoin?

Finally, are there any implementations underway? Could this be compatible with LND or CLN, say as a plugin or layer, or would an entire new node be needed, such as one built with LDK?

-------------------------

ZmnSCPxj | 2024-09-16 23:54:39 UTC | #3

> Could one use case for this be to help people in developing nations receive small amounts of bitcoin instantly and with low transaction fees? I suppose the question is, can this work well for small payments as well as for large ones?

Hopefully, yes. The idea is that the cost of management of a smaller number of UTXOs is amortized by the LSP for a larger number of clients.

>Additionally, how critical is liveness for trustlessness? Could this work on a mobile app for example?

For *trustlessness*, you need to come online at least once in the lifetime of the factory. The proposal is that a factory lasts for about a month, at the end of which you must have made a decision whether you exit or move to a new factory for the next month.  As long as you come online and do a *unilateral* exit within the lifetime of the factory, you are safe.  However, if you do *not* do a *unilateral* exit, you *must* come online during the "dying period" of the factory (a grace period offered by the LSP, and enforced onchain so that the LSP cannot rug you during the active period+dying period, but *CAN* rug you after it) at which point you ***MUST*** decide to unilateral exit, assisted exit (swap offchain to onchain with LSP cooperation; this can be indistinguishable from paying out using your entire funds if you use e.g. Boltz offchain-to-onchain), or move to a new factory (with LSP cooperation, possibly to a new LSP if your current LSP allows it or if we design the move to be indistinguishable from a payment using all your funds).

The mobile app is ***the*** target use-case here.  It turns out that mobile apps in iOS and Android have the ability to be notified, which gives them a small amount of CPU and network connectivity during which they could potentially perform a few important crypto operations, possibly including signatures.  We already know that Phoenix wallets can do enough operations while awakened using the notification system to receive payments while the phone is "asleep" in your back pocket, and that it is reliable enough in practice that it reduces pressure for the need for asynchronous receive.

To an extent, if you already use Lightning on a sometimes-online wallet like Electrum, Phoenix, etc. you already can handle the once-a-month-online in practice.

>What are the worst case scenarios, especially if the LSP is shut down by the authorities?

In such case all clients have to unilaterally exit, with their funds forced onchain (possibly into a UTXO that is below standardness dust limit meaning it will outright go to fees, or below practical onchain usage if onchain fees are high), and timelocks coming into play (which are potentially larger than timelocks already on normal Poon-Dryja Lightning channels).  This requires publishing O(N) transactions, where N is the number of fellow clients in the factory; the constant factors depend on arity of the tree (arity of 2 is 2x more, arity of 4 is 1.3333x more, etc; arity means the number of children at each node of the tree).

>How much do you think this can scale with 1MB blocks? Does this essentially solve the billions of Lightning users metric without the need for changes to bitcoin?

My own ***hopeful*** estimation is that this will 10x our effective channel-opening capacity, so is equivalent to approximately increasing block size to 10Mb (40 Mweight) in effect.  Not sure if we can reach *billions* yet but maybe the dozen millions is a bit more reachable with this, without sacrificing trustlessness just yet (beyond what Lightning already imposes on users, i.e. you have to come online periodically to ensure your counterparty has not performed a theft attempt).

>Finally, are there any implementations underway? Could this be compatible with LND or CLN, say as a plugin or layer, or would an entire new node be needed, such as one built with LDK?

There are no implementations underway.  I thought of this literally just last week, while my family was forcing me to ride YET ANOTHER roller coaster.  FML.  On the client side it would probably be an entire new node software, possibly taking some LDK code for managing of the Poon-Dryja channels at the leaves of the tree, but with novel code to handle the tree itself.  On the LSP side you would need to integrate into an existing node software; I think CLN has enough hooks that you could write a plugin so that forwarding can be implemented to and from this scheme to the "actual" LN node, by hooking into `htlc_accepted` for public-to-client forwardings, and just using `sendpay` for client-to-public forwardings; the LSP-side plugin can then use the same code as the client (including LDK dependency) to implement the factory construction as well, plus some automation to perform the periodic movement for the laddering scheme.  I expect LND would have similar enough hooks to implement this as well.

-------------------------

ariard | 2024-09-17 07:28:54 UTC | #4

[quote="ZmnSCPxj, post:1, topic:1143"]
The timeout condition forces all clients to come online before the timeout and exit the mechanism. Exit can be unilateral or with cooperation of the LSP.
[/quote]

This does not seems to works well if you have mass off-chain force-closure as more or less described in the “Forced Expiration Spam” section of the lightning paper: https://lightning.network/lightning-network-paper.pdf

Timeout condition only gonna provoke a mass on-chain fallout of unilateral exit competing for scarce blockspace, and no guarantee that the LSP is online when timeout matures for batching, at least at the protocol-level.

-------------------------

ZmnSCPxj | 2024-09-17 21:55:37 UTC | #5

If most unilateral exit costs are borne by the LSP, it has an incentive to (1) provide very good service so that clients have no desire to unilateral exit and instead do assisted exit, (2) to come online ASAP to batch at timeout maturity, to reduce the risk of accidental or deliberate unilateral exit by the LSP, and (3) be picky about its clients so that clients do not just unilaterally exit to drain the LSP of funds.

There is no magic bullet, only tradeoffs.

-------------------------

t-bast | 2024-09-18 14:37:42 UTC | #6

Thanks for this detailed post, this is well explained and the diagrams are very clear.

There are two cases that I'd like to understand better:

- how exactly can the LSP add liquidity when its `L` leaf outputs are depleted
- the state of the factory after a unilateral exit

### Adding liquidity when the `L` leaf outputs are depleted

[quote="ZmnSCPxj, post:1, topic:1143"]
Of course, if `B` cannot come online, the LSP does have to fall back to alternate methods, such as onchain. This can lead to onchain fees needing to be paid, which the LSP would pass on to `A`.
[/quote]

I don't see how that works: to add on-chain funds, the LSP needs to modify the root transaction to add more funds to it, right? Which means it has to modify the whole tree to propagate those added funds to some of the leaf outputs, which requires waking up every node in the tree? Or am I misunderstanding how that would work?

Another possibility is that when this happens, the LSP simply doesn't honor the liquidity request, and will only honor it when other participants can come online to transfer from within the factory, or when moving to a new factory (and thus broadcasting a new on-chain root transaction).

### State after a unilateral exit

[quote="ZmnSCPxj, post:1, topic:1143"]
Thus, in the case that `A` wants to exit, not only are the clients on the same leaf transaction (`B`) are inadvertently exited, but also sibling clients on the next higher level, `C` and `D`.
[/quote]

Can you tell me whether the following is correct in this scenario:

- B, C and D have been exited, which means they now have a "plain" lightning channels whose funding output is confirmed on-chain: this is fine, they can enroll into a new factory using a splice on that existing channel?
- E, F, G and H are still in a factory that is one level smaller than the previous one and consists of the subtree with those 4 nodes: the number of available state transitions has been reduced (since we lost one level of the tree), but apart from that nothing has changed for them?

### Synchronization issues with concurrent liquidity requests

It seems to me that there are non-trivial synchronization issues when moving liquidity inside the factory. If A wants liquidity and B isn't online, the LSP may reach out to C and D to use their leaf node. If that happens, we still need B to come online as well to exchange all the signature needed to complete the liquidity allocation, right?

While we're waiting for one of those nodes to come online, we may have other, conflicting liquidity allocation requests happening. I'm not sure how we can resolve them and avoid being in a huge mess where software need to track concurrent, incompatible asynchronous operations? Unless the LSP imposes a strict ordering, which may take a very long time to complete to get all the signatories online in the right order?

-------------------------

ZmnSCPxj | 2024-09-18 16:17:04 UTC | #7

>I don’t see how that works: to add on-chain funds, the LSP needs to modify the root transaction to add more funds to it, right?

No, I mean the LSP takes *other* LSP-single-sig funds outside of the mechanism and then adds a JIT channel to the client.

> * B, C and D have been exited, which means they now have a “plain” lightning channels whose funding output is confirmed on-chain: this is fine, they can enroll into a new factory using a splice on that existing channel?
> * E, F, G and H are still in a factory that is one level smaller than the previous one and consists of the subtree with those 4 nodes: the number of available state transitions has been reduced (since we lost one level of the tree), but apart from that nothing has changed for them?

Both are correct.

> It seems to me that there are non-trivial synchronization issues when moving liquidity inside the factory. If A wants liquidity and B isn’t online, the LSP may reach out to C and D to use their leaf node. If that happens, we still need B to come online as well to exchange all the signature needed to complete the liquidity allocation, right?

Yes, in that case LSP does **not** reach out to `C` or `D`, but does an onchain fallback (i.e. open a JIT channel with onchain LSP-single-sig funds).

> While we’re waiting for one of those nodes to come online, we may have other, conflicting liquidity allocation requests happening.

A simple heuristic is that if the client leaf partner is offline, the LSP just falls back to opening an onchain JIT channel.  Similarly, if `A` and `B` are online but the leaf has depleted the `L` liquidity, try to wake up `C` and `D` and take the `L` liquidity from their leaf, but if that fails due to offlineness of `C` or `D` then just fall back to opening an onchain JIT channel.

A more sophisticated "waiting" strategy can probably use research on e.g. optimistic locking implementations of transactional memory.  Basically, every deferred decision to give more liquidity to some client will decide on some set of clients it needs to "lock", and some scheduler can decide the order of client locking using similar implementations to transactional memory.  Once you have locked a client (i.e. checked it is online and reserved its participation in this transfer) you can then check if you can do the liquidity movement using those clients, then once the needed signatures have been provided by the clients you can release the locks on those clients so that other requests can continue.  Most requests will only lock a client and its leaf partner, so this will have high concurrency.

-------------------------

t-bast | 2024-09-18 16:29:35 UTC | #8

Thanks that's very clear.

[quote="ZmnSCPxj, post:7, topic:1143"]
No, I mean the LSP takes *other* LSP-single-sig funds outside of the mechanism and then adds a JIT channel to the client.
[/quote]

So basically whenever the LSP wants to use on-chain funds that aren't in the factory, they just keep this fully separate from the factory and import them into normal channels.

This is a drawback, because it means the LSP would need to have multiple channels per-user, which defeats the goal of using less utxos for more users. But LSPs could opt to never use normal channels and only work inside factories, with the downside that liquidity requests sometimes cannot be satisfied...

-------------------------

ZmnSCPxj | 2024-09-18 17:22:08 UTC | #9

[quote="t-bast, post:8, topic:1143"]
This is a drawback, because it means the LSP would need to have multiple channels per-user, which defeats the goal of using less utxos for more users.
[/quote]

The intent is that the LSP makes a probabilistic bet that most users, most of the time, can have their hardware awakened so the LSP can use offchain funds to provide liquidity. This also means that we probably need to handle at most 2 channels per client: one inside this mechanism, and one onchain (which you JIT-open or JIT-splice to if the other clients cannot be awakened inside this mechanism).  So you might have one UTXO backing multiple clients (the one that backs this mechanism), and for some fraction (< 100%) of those clients, have an additional UTXO for the optional onchain channel (1 + FRACTION x N where the LSP bets that FRACTION < 100%).  This is still better than having 100% x N UTXOs for N users.

The onchain channel itself can be infected with a `CLTV` branch for the LSP so that after the period, the client has to move to a new factory and consolidate both channels into a single channel inside the next timeout-tree-structured mechanism.

-------------------------

ZmnSCPxj | 2024-09-23 06:13:27 UTC | #11

***ANYWAY*** in Lightning proto dev summit, someone pointed out that we might be able to have some kind of sign-only-once scheme that could be used here.

(also pointed out that the number of bonds have to be equal to the number of clients; so for example, for a 2-client leaf, the LSP can allocate 1/3 of its funds in the leaf while only one client is online; more generally, for an N-client leaf, the LSP can allocate 1/(N+1) of its funds if only one client is online).

The following pointer was given regarding possible constructions that ***might*** be workable for sign-only-once today without `OP_CAT`: https://eprint.iacr.org/2024/025

-------------------------

ZmnSCPxj | 2024-09-23 06:57:54 UTC | #12

Another idea proposed in the Lightning proto dev summit is with LSP-assisted exit.  If a client exits from one factory with no intention to ever return to that factory (either by exiting onchain, or to a new factory during the dying period of the previous factory) then the client can use PTLCs to offer the private key it uses inside the factory.  For example, if client `A` wants to permanently exit using an LSP-assisted exit to an onchain address, then `A` can offer an in-factory PTLC to the LSP, where the payment point is the public key of `A`. Then the LSP has to make an onchain PTLC with the same payment point, which is claimable by `A`.  `A` can then claim the onchain PTLC into its own onchain address with a fresh keypair, revealing the scalar behind the payment point, which is equal to the private key of `A`.

The advantage of this private key turnover is that if `A` and `B` are on the same leaf, `A` has performed this assisted exit and never comes online again, the LSP can, with `B` and its private key copy of `A`, sign a new leaf state, without `A` ever talking to the LSP ever again.  The LSP can even use the funds of the `A`-`L` channel to provide additional liquidity to the remaining client `B`!

If the same mechanism is used for transferring from a dying factory to the next laddered factory, then if all clients exit the dying factory, the LSP is now in possession of all the private keys and can claim using the `A & B & ... & G & H` branch. This can be a Taproot keyspend path, which is just a single signature without revealing a special `OP_CLTV` script, and which is indistinguishable from many other P2TR spends, giving better privacy to the LSP (and by implication, its clients).  If all clients exit before the actual end-of-life of the channel factory (i.e. before the end of the dying phase) then the LSP can recover its funds "early", even, which might be strategically important in funding decisions.

Because of this, the LSP may now have an incentive to cover some of the cost to perform an assisted exit --- that is, the assisted exit can be cheaper, for the client, than the unilateral exit case, because the LSP has an advantage that it can sockpuppet the exiting client for the rest of the lifetime of the individual factory.  The LSP can offer to pay for the privilege of sockpuppeting the client when it assisted-exits, by reducing the cost (on the client) for exiting onchain.

-------------------------

ZmnSCPxj | 2024-09-26 04:30:47 UTC | #13

Okay, some implementation notes.

## Tree Nodes Pay Fees

Obviously `nVersion=3` > `nVersion=2`. And we will soon be able to have transactions pay 0 fee and still propagate, as long as it has an attached child transaction that pays for it.

However, for the implementation of SuperScalar, I intend that all tree nodes pay non-0 fee.

The big issue with 0-fee transactions is that **they require exogenous funds to pay for them**.  And the goal of SuperScalar is to be able to onboard people, possibly people who do not have an existing UTXO they can use to pay exogenous fees.

Thus, I propose that tree nodes pay a small amount of fees.  Because reaping a timeout tree means that the tree nodes will not exist and therefore will not pay the fees, the fees paid by tree nodes are recovered if the LSP reaps the UTXO without publishing the entire tree. The net effect is that ***tree node fees are default-paid by the LSP***.  The result is that the tree has endogenous fees.

As noted above, some tree nodes are state transactions that are modified.  When modifying a node, we reduce the `nSequence`.  In addition, we can **also** increase the feerate of that node.  Thus, even in a high feerate environment, we have a minor assurance that the latest state version is the one that is most likely to confirm.

We can add P2A outputs to each tree node as well.  However, by itself it is unsafe.  Consider the following attack scenario:

* The LSP sells some inbound liquidity to a client within the SuperScalar mechanism.  The effect is that there are two version of a leaf node, one which has more funds in the LSP side.
* The client uses that inbound liquidity, such that loss of the channel capacity is loss of funds.
* The LSP waits for a high-fee period.
* The LSP initiates a unilateral exit for the victim client, confirming tree nodes until the parent of the leaf containing the victim.
* The client attempts to broadcast the latest leaf node, but because it has no external funds, it cannot feebump the latest leaf node above the prevailing confirmable rate.
* The LSP waits for the ***previous*** leaf node to be broadcastable (it has a later `nSequence` than the latest one) and attaches a higher feerate to it via the P2A output.
* The previous leaf node gets confirmed, the LSP claws back its funds, the client loses access to all its funds.

The above inducts over every state transaction in the tree, thus we need to have state transactions without P2A outputs if we are worried about such an attack.

To protect against this, we can observe:

* Within the SuperScalar mechanism, all state transfers are of the form:
  * fully owned by `L` -> a channel between a client `A` and `L`.
  * This is because the LSP sells inbound liquidity to clients.  The LSP wants to ensure that all sales are final, in that it never refunds inbound liquidity that has already been sold.

Within the SuperScalar mechanism, we do not intend to ever have the liquidity go back from the `A`-`L` channel to solely owned by `L` --- as noted, once the LSP has sold some unit of inbound liquidity, it wants to ***not*** take back that liquidity and refund the client for the inbound liquidity.

What we can thus do is to recognize that all `L`-owned funds are unidirectional, they will always go down.  So instead of having a "plain" `L` script, we can have the script `<secret> | L`. 
 Whenever we invalidate a leaf node (and if we change a non-leaf node, we also invalidate all children down to the leaf nodes) the LSP shares the `<secret>` of every `L` liquidity stock output of the OLD version to each client that signs off on the change.  For the new version, the LSP generates a new secret for the `L` liquidity stock output.  This can use the same shachain mechanism we already use for Poon-Dryja BOLT Lightning channels.

If the LSP uses the P2A output to confirm an older version during a high-fee environment, then the `L` outputs have an alternate `<secret>` path that the clients can use.  The clients can then retaliate by burning the `L` output to miner fees.  Thus, the LSP is incentivized to always ensure that the latest version of every state transaction gets confirmed before the older version can be broadcastable (because the clients also have a copy of the older version and can broadcast them for confirmation and subsequent burning of LSP funds).

With this, we can have P2A on all tree nodes as well, and incentive for the LSP to always assist in confirming the latest version of a tree node if a client wants to unilaterally exit.

-------------------------

ariard | 2024-09-29 22:59:07 UTC | #14

[quote="ZmnSCPxj, post:5, topic:1143"]
If most unilateral exit costs are borne by the LSP, it has an incentive to (1) provide very good service so that clients have no desire to unilateral exit and instead do assisted exit, (2) to come online ASAP to batch at timeout maturity, to reduce the risk of accidental or deliberate unilateral exit by the LSP, and (3) be picky about its clients so that clients do not just unilaterally exit to drain the LSP of funds.
[/quote]

To gauge better the fault-tolerance of this construction, I think you should have an estimation of the on-chain weight units costs for a range of less than perfect interactivity situations (e.g m-of-m, m-of-m - 1, m-of-m - 2). Also any assisted exit sounds a double-edged sword mechanism as it could be leveraged to kick out a user, at the worst time e.g when the on-chain fees are full.

There is no guarantee that the LSP respects its own liquidity policy on that regard.

> In order for `A` to exit, it must first have the `A & L` channel funding outpoint confirmed, then finally exit the `A & L` channel using standard Poon-Dryja unilateral exit. Thus, `A` must publish the path to its channel below:

Another question - How do you prevent `A` to unilaterally downgrade the construction fault-tolerance for all the subset of users intersecting with its path to its channel ? I.e A to D being partitioned from E to H.

-------------------------

ZmnSCPxj | 2024-09-30 00:10:27 UTC | #15

>Also any assisted exit sounds a double-edged sword mechanism as it could be leveraged to kick out a user, at the worst time e.g when the on-chain fees are full.

An assisted exit is a PTLC in the in-factory construction from user to LSP, followed by an onchain PTLC from LSP to user, with the scalar being the private key that client is using for the in-factory constructions.  It is ***not*** the LSP initiating a unilateral exit for a user and assisting it by paying for onchain fees.  The client is the one that initiates this; it has to actually offer a PTLC inside the in-factory construction before the LSP can safely put the swap PTLC onchain.  Thus, the client can choose to ***not*** initiate an assisted exit during high-fee times.  The LSP has no incentive to eject clients during high fees.

An assisted exit like this allows the LSP to recover its in-factory funds earlier. If all clients perform assisted exits (including assisted exits from the current factory to a newer factory) then the LSP can recover funds directly from the funding outpoint, ***and*** can get those funds ***immediately***, before the end of the timeout period (which is safe since all the clients have exited; the entire fund is now solely owned by the LSP). This is in contrast with evicting clients by unilaterally exiting them: the LSP has to wait for some time due to the extra Decker-Wattenhofer steps along the way, and has to actively pay for fees if the LSP does it during high-fee periods.  That is, the LSP has an incentive to keep clients in the tree until the clients perform an assisted exit, because the assisted exit lets the LSP use less blockspace to recover its funds.  It has nothing to do with the LSP fee policy, simply the pragmatic realization that the private key of the client is valuable (and the client can safely hand it over in exchange for getting its funds out of the construction and into another one, either onchain or to a new factory) because it allows the LSP to close the construction earlier and with just a simple 1-input 1-output tx with a single signature, without even having to show the `L & CLTV` script tapleaf.

Even if not all clients perform an assisted exit (say, a client never comes online again because the underlying phone hardware was destroyed or irretrievably lost) the LSP would still prefer an assisted exit from the *other* clients, because each node has a `A & B & C & ... & Z & L` branch and once a subset is reached that is a strict subset of the clients that *have* performed assisted exit, the LSP can recover the rest of its funds from that output using the "everyone signs" branch without publishing that subtree, because it is now in possession of everyone's keys.  This is still cheaper than the LSP preemptively evicting non-responsive clients; the LSP can always hold out and hope that the  client comes back online and performs an assisted exit (which it would need to perform to transfer from one tree to the next).

Private key handover is ***AWESOME***.

-------------------------

ZmnSCPxj | 2024-10-02 16:41:04 UTC | #16

Wide Leaves With Pseudo-Spilman Multiparty (N>2) Channel Factories
==================================================================

Consider the following sub-tree:

                                        nSequence
                                         +-----+---+
                                         |     |A&L| LN channel
                            +--+-----+   |     +---+
      nSequence           +>|  |A&B&L|-->| 432 |B&L| LN channel
       +-----+----------+ | +--+-----+   |     +---+
       |     |  (A&B&L) |-+              |     | L | Liquidity stock
       |     |or(L&CLTV)|                +-----+---+
    -->| 432 +----------+               nSequence
       |     |  (C&D&L) |                +-----+---+
       |     |or(L&CLTV)|-+              |     |C&L| LN channel
       +-----+----------+ | +--+-----+   |     +---+
                          +>|  |C&D&L|-->| 432 |D&L| LN channel
                            +--+-----+   |     +---+
                                         |     | L | Liquidity stock
                                         +-----+---+

The above is not quite what I showed earlier, but the only real
change is that the kickoff hosting the leaves have an arity of 1
rather than 2, which I suggested earlier to provide better isolation
of the pair `C` `D` from the pair `A` `B`.
We do have the option of having the kickoff transactions have an
arity of 2 or more for higher up in the tree, just as we might have
higher arities for state transactions at nodes higher up the tree.

For the above sub-tree, we have the following operations that can
be done without touching the blockchain:

* `A` can buy liquidity from the LSP if `B` and `L` are online.
* `B` can buy liquidity from the LSP if `A` and `L` are online.
* `C` can buy liquidity from the LSP if `D` and `L` are online.
* `D` can buy liquidity from the LSP if `C` and `L` are online.

Now, let me present an alternative implementation of the above
sub-tree, that also allows the above operations, again without
touching the blockchain.
The advantage of the below is *the removal of one Decker-Wattenhofer
layer*, which translates to a reduction in the maximum total
relative locktime (`nSequence`).

       +-----+----------+
       |     |    A&L   | LN channel
       |     +----------+
       |     |    B&L   | LN channel
       |     +----------+
       |     |  (A&B&L) | Liquidity stock (A&B)
       |     |or(L&CLTV)|
    -->| 432 +----------+
       |     |    C&L   | LN channel
       |     +----------+
       |     |    D&L   | LN channel
       |     +----------+
       |     |  (C&D&L) | Liquidity stock (C&D)
       |     |or(L&CLTV)|
       +-----+----------+

How does `A` (`B` `C` `D`) buy liquidity from `L` in the above
scheme?
Like so:

       +-----+----------+
       |     |    A&L   |
       |     +----------+
       |     |    B&L   |   +--+----------+
       |     +----------+   |  |    A&L   |
       |     |  (A&B&L) |-->|  +----------+
       |     |or(L&CLTV)|   |  |  (A&B&L) |
    -->| 432 +----------+   |  |or(L&CLTV)|
       |     |    C&L   |   +--+----------+
       |     +----------+
       |     |    D&L   |
       |     +----------+
       |     |  (C&D&L) |
       |     |or(L&CLTV)|
       +-----+----------+

The new transaction takes the `A & B & L` branch.
(In the case that the LSP incentivizes `B` to join the signing
session by providing a small amount of free inbound liquidity,
we "simply" add another `B & L` output as an extra channel.)

The above is remniscient of Spilman channels; the output here is
semantically owned by `L`, and `L` may reduce its allocation of
funds to give more funds (or in this use-case, inbound liquidity)
to the other participants.

However, a difference with the above vs ***real*** Spilman channels
is when `A` buys liquidity again:

       +-----+----------+
       |     |    A&L   |
       |     +----------+
       |     |    B&L   |   +--+----------+
       |     +----------+   |  |    A&L   |   +--+----------+
       |     |  (A&B&L) |-->|  +----------+   |  |    A&L   |
       |     |or(L&CLTV)|   |  |  (A&B&L) |-->|  +----------+
    -->| 432 +----------+   |  |or(L&CLTV)|   |  |  (A&B&L) |
       |     |    C&L   |   +--+----------+   |  |or(L&CLTV)|
       |     +----------+                     +--+----------+
       |     |    D&L   |
       |     +----------+
       |     |  (C&D&L) |
       |     |or(L&CLTV)|
       +-----+----------+

In a ***real*** Spilman channel, `L` would have simply replaced
the previous transaction with a new one.
However, direct replacement is unsafe when there are more than 2
participants, because the other participants could actually be
sockpuppets of `L`.
In that case, `L` can get a complete set of signatures for all
versions of the transaction, including older versions of the
transaction, and can bribe a miner to confirm the older version.
Thus, this scheme is not a true Spilman channel factory.

Instead, the above scheme forces the LSP to only spend
each of the liquidity stock UTXOs at most once, and protects
against double-spending by clients simply refusing to sign for a
transaction that spends an already-used liquidity stock UTXO.
Even if the other participants are sockpuppets of `L`, as long
as the buying participant is a signatory, it can ensure the
correct operation of the scheme.

The major advantage of this wide leaf is that it provides the
same basic liquidity purchase operations as the first sub-tree
shown earlier, but with one less Decker-Wattenhofer layer.
Each layer of Decker-Wattenhofer is a liability even if the
construction never hits the blockchain, because each
Decker-Wattenhofer layer adds more relative locktimes.
Any hosted HTLCs or PTLCs would need a CLTV-delta that is at
least the total maximum Decker-Wattenhofer locktime, plus any
margin deemed necessary by the receiving side;
if the remaining time in a pending HTLC or PTLC is less than
the current Decker-Wattenhofer locktime plus margin, participants
***MUST*** drop the construction onchain, which is expensive and
time-consuming due to the larger participant set.
Thus, reducing the worst-case Decker-Wattenhofer locktime
is always an advantage, as it helps avoid unilateral closes.

If a client has ever bought liquidity, then its unilateral
close involves more bytes on the blockchain.
This is a distinct disadvantage over the original sub-tree.
In addition, because the new inbound liquidity is in a new
channel, forwarded HTLCs from the LSP to client may need to
be split at that hop (i.e. local multipath).

Further, this additional complexity with managing liquidity
compounds.
If the wide leaf is replaced entirely (for instance, if the
liquidity stock for `A` and `B` runs out and the LSP wants to
move some of the stock from `C` and `D`), it would be advantageous
to cut-through all the dependent transactions that give
additional channels to the buying client `A`, creating just a
single channel.
This means that any hosted HTLCs would need to be put in the
same single new channel in the new state, effectively merging
the pending HTLC sets.

Against Internal Splicing
-------------------------

In this scheme, as noted, the LSP creates a new channel to the
client instead of increasing the capacity of the existing channel
inside the construction.
This adds complications, in that HTLCs forwarded from the LSP to
the client may need to be split among the multiple channels.
We might wonder if we could instead splice, so that there is only
one liquidity "bag" for the HTLCs.

Suppose that the LSP has the policy that incentivizes `B` to
participate in signing sessions by giving `B` a small amount of
free inbound liquidity whenever it is online to assist the
purchase of `A` of inbound liquidity.
In that case, the splice transaction would spend the `A & L` and
`B & L` outputs.

However, suppose in addition that `B` is actually a sockpuppet of
the LSP.
In that case, the LSP can trivially invalidate the splice transaction
by simply spending from the `B & L` output, as it is actually in
possession of both keys.

The same may happen even if the LSP does ***not*** have a policy
of incentivizing signing sessions by offering free inbound
liquidity.
If `B` purchases inbound liquidity first (or rather, the LSP emulates
`B` purchasing inbound liquidity) and *then* `A` buys inbound
liquidity, then again the splice of the `A & L` channel is dependent
on the `B & L` output, and if `B` is actually a sockpuppet of the LSP,
then this is still a way for the LSP to clawback the sold liquidity.

Thus, it is not safe to use internal splicing inside offchain
mechanisms; we should use cut-through instead when transaction
replacement is possible (which is not true in this scheme; it is
dependent on a parent node being able to perform replacement).
Splicing in general is only safe if confirmed onchain.

Incentivizing LSP-pays-mining-fee
---------------------------------

An important model here is that clients may not have onchain
UTXOs with which to pay for unilateral exit.
One may consider this scheme, as well as related schemes, as ways
for a client to build up their Bitcoin HODLings without having an
onchain UTXO, but with an assurance that the service provider has
no incentive to rug their funds until they have accumulated enough
to own their own unique UTXO.

The unilateral exit is, in effect, a proof-of-liabilities that is
publishable onchain, to prevent the LSP from rugging the client.

Thus, this scheme also provides an "assisted exit" where, in
exchange for the client keys in this construction, the LSP assists
the client to get an onchain UTXO, or to move to a new factory in
the ladder.
This is done by using PTLCs to swap, with the scalar being the
private key.

The assisted exit is part of the protocol so that the LSP has an
incentive to retain clients in the mechanism, until the clients
have performed an assisted exit.

* If all clients have performed an assisted exit from one of the
  factories, the LSP can immediately reclaim the funds in the
  factory, even before the timeout.
  * Even if some clients do not perform an assisted exit, the LSP
    can, at its option (i.e. if onchain fees are low) perform a
    partial unilateral exit to reclaim funds from subsets of the
    participating clients.

Our goal is that, if the client is unsatisfied with the performance
of the LSP, the client can perform a unilateral exit at the
expense of the LSP.

Crucially, as we update offchain, the LSP has an incentive to
claw back its funds in case of a unilateral exit, by waiting for
older states to be publishable onchain (by the Decker-Wattenhofer
mechanism) and then paying to have that confirmed.
If clients do not have an existing onchain UTXO with which to
pay fees, and all their Bitcoins are held in a shared UTXO with
the LSP, then the clients will not be able to have the newer states
confirmed.

Our exact scheme for the not-really-Spilman factory is thus:

* All clients in the liquidity stock sign *two* transactions
  spending the liquidity stock:
  * One is the transaction we showed above, funding new client
    channel(s) and an optional change output that is the new
    liquidity stock.
    - This transaction is `nVersion=3`, has 0 fee, and has a
      P2A output for feebumping.
    - Call this the "state" transaction.
  * Another version is a `nLockTime`d transaction, with timelock
    equal to the timeout of the tree, and with a single `OP_RETURN`
    output.
    - Care must be take to respect minimum transaction size
      (65 bytes in txid format).
    - Also has `nVersion=3`, but its fee is effectively the entire
      value of the liquidity stock.
    - Call this the "burn" transaction.

The liquidity stock output of the wide leaf has effectively two
states:

* The original starting state where it is owned semantically by
  the LSP, but is claimable at the end of the tree timeout.
* The new state where part of the fund is allocated to a new
  channel with a client.

The LSP may attempt to claw back the original state by simply
not feebumping the 0-fee state transaction.
However, if so, the clients can retaliate by simply publishing
the burn transaction.
Due to its small size, it is very difficult for the LSP to
replace it with a higher-fee transaction; it can only do so by
effectively spending more than the actual original liquidity
stock.
If so, the LSP has incentive to always publish --- and pay for
confirmation of --- the correct latest state, as it ends up
saving more money that way.

As practical consideration, the actual `L & CLTV` locktime should
be a little later than the formal end of the timeout tree, while
the burn transaction should have the `nLockTime` of the timeout
tree.

-------------------------

ariard | 2024-10-02 22:48:07 UTC | #17

[quote="ZmnSCPxj, post:15, topic:1143"]
An assisted exit is a PTLC in the in-factory construction from user to LSP, followed by an onchain PTLC from LSP to user, with the scalar being the private key that client is using for the in-factory constructions
[/quote]

In the worst-case, how do you guarantee the user to guarantee PTLC-to-LSP and PTLC-to-LSP to settle at the same time ? And the user to have enough funds to confirms those 2 transactions under few blocks of relapse. 

See my point above the “Forced Expiration Spam”: https://lightning.network/lightning-network-paper.pdf 3 I don’t see how you’re making things better here.

-------------------------

ariard | 2024-10-02 22:55:05 UTC | #18

[quote="ZmnSCPxj, post:16, topic:1143"]
Thus, this scheme also provides an “assisted exit” where, in exchange for the client keys in this construction, the LSP assists the client to get an onchain UTXO, or to move to a new factory in the ladder. This is done by using PTLCs to swap, with the scalar being the private key.
[/quote]

How can you trust the LSP to fee-bump just-on-time for the end user and the LSP won’t maliciously abort the private key handover ?

“Fair exchange” of a secret is an unsolved problem since 90s's distributed system literature.

-------------------------

ZmnSCPxj | 2024-10-03 02:17:32 UTC | #19

> In the worst-case, how do you guarantee the user to guarantee PTLC-to-LSP and PTLC-to-LSP to settle at the same time ? And the user to have enough funds to confirms those 2 transactions under few blocks of relapse

Honestly, I do not have an answer for you here.  What I can point out is that while it does not work in theory, it does work in practice; lots of users are able to swap offchain to onchain, and vice versa; lots of people can still pay over Lightning, over multiple hops, even during high onchain fees.  It somehow still all works, and this is just another application of same constructions.  Puzzling, but sometimes I just have to move on and get stuff done.

-------------------------

ariard | 2024-10-04 01:08:45 UTC | #20

[quote="ZmnSCPxj, post:19, topic:1143"]
it does work in practice; lots of users are able to swap offchain to onchain,
[/quote]

It doesn’t work in practice. There is no guarantee in bitcoin that an off-chain state is reconciliated with the on-chain state of the blockchain, before expiration of the safety timelocks in a world where users are in a fees competition to confirm off-chain states.

> Puzzling, but sometimes I just have to move on and get stuff done.

It’s easy to get stuff done, I can do it too. Just ship completely broken stuff to users.

-------------------------

ZmnSCPxj | 2024-10-07 18:44:30 UTC | #21

Addenda

# CLTV Locktime Extended By Decker-Wattenhofer Relative Delays

A side-effect of Decker-Wattenhofer use is that HTLCs hosted (directly or indirectly, as in the channel factory case) have a minimum CLTV-delta that is equal to the current Decker-Wattenhofer `nSequence` state delay, plus any safety margin needed by the node for downtime or forwarding purposes.  Note that this happens for *any* use of `nSequence` to provide timeouts for resolving the latest state --- for example, Poon-Dryja **could** have had the same drawback, **if** the `CSV` timelock had been placed before HTLCs instead of as an alternate branch (the tradeoff here is that if the BOLT Poon-Dryja specification had put the `CSV` delay *before* HTLCs, it would have allowed Poon-Dryja implementations to completely drop historical HTLC data --- note that dropping the resolved and failed HTLC data is something the implementation *can* do but which actual code could potentially still keep for surveillance purposes anyway, so this tradeoff was not taken).

However, it also means that the LSP has to unilaterally close "early" if clients have not done an assisted exit from their current tree to the next tree, or to onchain, and the specific SuperScalar tree is approaching the current Decker-Wattenhofer delay to the timelock.

Thus, for actual implementations, the timelock of the timeout trees would be active period + dying period (i.e. a grace period for clients to move from older tree to newer trees) + maximum Decker-Wattenhofer delay.

This increases pressure for the LSP to provide proper assisted exit, as the additional delay from the Decker-Wattenhofer mechanisms would also represen additional time that the LSP funds are locked.

I am currently wondering if the Decker-Wattenhofer delay can be mashed with the dying period, just as the Decker-Wattenhofer layers were mashed with the timeout-tree layers in SuperScalar.

# Asymmetric Onchain Fee Schemes For Poon-Dryja

In the most recent Lightning Network protocol dev summit, it was decided that the "next step" would be 0-fee commitment transactions, with a common P2A output that either side can use to fee-bump.

This has the drawback that it requires exogenous fees.  If our base assumption is that clients do not afford their own UTXO (which is why they are sharing one with many other clients and the LSP in SuperScalar), then 0-fee commitment transactions are not good for clients.

An alternative idea given to a few people at the summit, was to have multiple commitment transactions per commitment state with different feerates (dependent transactions such as HTLC-success and HTLC-failure would have the same feerate as the commitment transactions).  This is kept incentive compatible by taking advantage of the asymmetry of Poon-Dryja commitment transactions: the side that holds that commitment transaction is the one whose funds are deducted from for fees of that commitment transaction.  Obviously this means that clients with 0 funds in the channel cannot pay for the commitment transaction, but if the client has 0 funds, it does not matter if the client cannot unilaterally close anyway.  The extended drawback, however, is that the maximum fee the client can pay is limited by how much money they have in the channel (HTLCs they offerred may be garnished similarly).

An extension of this asymmetry would be to asymmetrize the onchain fee scheme:

* For the LSP side commitment transaction, use a P2A output so that onchain fees are paid exogenously --- the LSP is presumed to have ready access to onchain funds with which to CPFP via the P2A output.
  * This allows the LSP to reduce its reserve requirement per channel to only provide security to the client in case the LSP tries to cheat.
* For the client side commitment transasction, use multiple versions with various feerates, funded endogenously by the client-side owned funds and possibly the HTLCs they offered.

-------------------------

ZmnSCPxj | 2024-10-07 18:52:54 UTC | #22

[quote="ariard, post:20, topic:1143"]
It’s easy to get stuff done, I can do it too. Just ship completely broken stuff to users.
[/quote]

I object to "completely".  Consider that custodial Bitcoin wallets are even worse, in that the offchain state is merely the trustmebro of the custodian; with Lightning and SuperScalar, the offchain state at least has the (unguaranteed) possibility of resolving with the onchain state if the LSP becomes uncooperative (unlike the case of custodial Bitcoin, where if the custodian refuses to cooperate, it can simply 0 out your account under you).  Custodial Bitcoin is significantly more broken than Lightning or similar schemes.  Until you can present a perfect scheme that allows Bitcoin to have offchain state perfectly resolved 100% of the time, I would respectfully ask you to point your attention at custodial schemes instead.  Improving something is still better than waiting for the perfect thing.  Sometimes you have to accept gray in a gray vs black fight.

-------------------------

instagibbs | 2024-10-08 14:11:40 UTC | #23

[quote="ZmnSCPxj, post:21, topic:1143"]
For the client side commitment transasction, use multiple versions with various feerates, funded endogenously by the client-side owned funds and possibly the HTLCs they offered.
[/quote]

This is in effect Peter Todd's suggestion to make fees endogenous(and moreover, a single transaction).

It begets potentially weird behavior in that the layer 1 chain can not actually know what the latest state is, so we don't know what "their own funds" are, to spend as fees. During the challenge period, we just don't know. It relies much heavier on the penalty mechanism, and ignores the fact that a miner could be in cahoots with your counter-party, robbing the LSP by simply playing an old state that had lots of fees attached to it.

If we had "non-contentious" funds inside the tree, ala the Timeout Tree paper(off-chain funds, I think it was called), these could directly be used without weirdness, but that's liquidity per-user at least, and I honestly didn't quite understand the stated construction.

-------------------------

ariard | 2024-10-08 21:40:46 UTC | #24

> This is standard tree transaction structures, but timeout trees also add an alternative spending condition: the LSP can spend by itself after a particular timeout.

> Private key handover is AWESOME.

The security model of time-sensitive contracting protocol is to be able for any counterparty to broadcast and unilaterally fee-bump its off-chain states, before expiration of the safety timelocks.

This construction is so broken...The arity of the tree inflates the branch of transactions in weight units that a counterparty can have to fee-bump in the worst-case scenario.

"Fair secret exchange" i.e the private key out-of-band swap to make an assisted exit cannot work, as there is no guarantee that the LSP _complete_ the key exchange on time (under classic physics), before the safety timelocks of the tree expire. That way letting the LSP to rug pull the end users.

> An alternative idea given to a few people at the summit, was to have multiple commitment transactions

> per commitment state with different feerates (dependent transactions such as HTLC-success and HTLC-failure

> would have the same feerate as the commitment transactions).

> This is in effect Peter Todd’s suggestion to make fees endogenous(and moreover, a single transaction).

Sure making the fees endogenous is an idea known for years, with rainbow ranges of pre-signed replacement lightning states. But I think I should still go to explain to Peter Todd, why it doesn't work as soon as you have one or two bitcoin stacked in your lightning channels.

> I object to “completely”. Consider that custodial Bitcoin wallets are even worse, in that the offchain state

> is merely the trustmebro of the custodian; with Lightning and SuperScalar, the offchain state at least has the

> (unguaranteed) possibility of resolving with the onchain state if the LSP becomes uncooperative (unlike the case

> of custodial Bitcoin, where if the custodian refuses to cooperate, it can simply 0 out your account under you).

> Custodial Bitcoin is significantly more broken than Lightning or similar schemes. Until you can present a perfect

> scheme that allows Bitcoin to have offchain state perfectly resolved 100% of the time, I would respectfully ask

> you to point your attention at custodial schemes instead. Improving something is still better than waiting for the

> perfect thing. Sometimes you have to accept gray in a gray vs black fight.

Can I ask you a simple question ? Are you the ZmnSCPxj which whom I've already been on the same panel at some bitcoin conference to talk about the subject of off-chain scaling eyes in the eyes ? And if yes are you realizing this work about SuperScalar as part of your paid time as an [employee](https://github.com/ZmnSCPxj-jr) at TBD's the Jack Dorsey’s Block Inc's subsidiary ? I respect your privacy, just not using a pseudonym as an excuse for shameless corporate lobbying about open-source.

Apart of that, what you're saying about custodial wallet vs non custodial wallet is gibberish, and one is better to put its money at Silicon Valley Bank, than within a SuperScalar off-chain construction.

Back to the more technical conservation about scaling and off-chain constructions, if one goes the way of using short-paced timelocks and Decker-Wattenhofer Factories are just that, one as to find a solution at the consensus-level for the “*Forced Expiration Spam*" problem as described in the lightning whitepaper (section 9.2).

I'm not the one who came to describe this problem in the bitcoin community. Tadge Dryja and Joseph Poon did it back in 2015. And as far as I know, since then there has been no research emanating from the academic or industry pointing out that this problem is not a real issue for this approach of bitcoin scalability, of which factories and payment channels clearly belong.

-------------------------

ZmnSCPxj | 2024-10-08 22:10:19 UTC | #25

> Can I ask you a simple question ? Are you the ZmnSCPxj which whom I’ve already been on the same panel at some bitcoin conference to talk about the subject of off-chain scaling eyes in the eyes ?

Yes

>And if yes are you realizing this work about SuperScalar as part of your paid time as an [employee](https://github.com/ZmnSCPxj-jr) at TBD’s the Jack Dorsey’s Block Inc’s subsidiary ?

Yes.

>And as far as I know, since then there has been no research emanating from the academic or industry pointing out that this problem is not a real issue for this approach of bitcoin scalability, of which factories and payment channels clearly belong.

Feel free to attack the actual Lightning Network to demonstrate the problem. Clearly, you believe:

>one is better to put its money at Silicon Valley Bank, than within a SuperScalar off-chain construction.

If so, you can demonstrate this, today, with the actual Lightning Network.

-------------------------

ZmnSCPxj | 2024-10-09 01:05:29 UTC | #26

addendum

# Inversion of Timelock Default

Existing timeout tree discussion has the timeout default in the favor of the LSP.

However, we should note that the construction needs a timeout (in order to provide a well-defined scope for how long the LSP needs to provide service to the clients), but does not actually need to have the timeout default to the LSP.

If we assume that the LSP is in the business of selling liquidity, we can assume that the LSP has large amounts of funds available onchain to pay for onchain fees if the timeout tree needs to be published onchain.  What we need is a way to force the LSP to handle unilateral closes and pay for them if the client is so unsatisfied with LSP services that it decides a unilateral close is more palatable.

Instead of having an `L & CLTV` branch at the transaction outputs of Decker-Wattenhofer state transactions, we can instead have the signatories sign an `nLockTime`d transaction that sends the funds to the clients, with the timelock being the timeout of the tree.  Thus, each node output that goes would have gone to an `(A & ... & Z & L) or (L & CLTV)` would instead have just `A & ... & Z & L` and ***two*** transactions signed:

* The node as shown in the main post.
* An alternate transaction, locked at `nLockTime`, which distributes the funds so that the initial channels of `A`...`Z` are given solely to the respective client, and with `L`-funds (i.e. the liquidity stock) split evenly among all clients.
  * Each node output that eventually leads to the client channel must, by necessity, include the total value of the client channel, plus any channel reserve imposed by clients on the LSP, plus any fellow client channels, plus the liquidity stock the LSP is holding ready for sale to clients.
  * As the clients have unilateral control of the outputs, they can trivially fee-bump this alternate timeout transaction to any level.

Then, if a client decides it wants to unilaterally exit, it can force the LSP to pay for unilateral exit by simply never performing an assisted exit from the current tree and waiting until `nLockTime`.  If the blockheight approaches the `nLockTime` of the tree, the LSP ***must*** initiate the unilateral exit itself, *and* pay for the confirmation of those nodes, or else it risks loss of all funds still locked in that part of the sub-tree.

If a client has performed assisted exit (i.e. a PTLC-based swap that exchanges the client private key used in the tree for onchain funds, or for funds in next laddered timeout-tree) then the LSP does not need to fully perform a unilateral exit; it only needs to publish enough nodes until it reaches an output with `(A & ... & M & L)` where it already got the client private keys `A`...`M` via assisted exit.

This means that the LSP is very incentivized to provide assisted exit.  For instance, for an onchain assisted exit, the client can wait for the PTLC output to be deeply confirmed, and if onchain feerates have changed enough, can require the LSP to re-sign a new PTLC-claim transaction at a different feerate, and the LSP has incentive, up to the cost of onchain fees to perform a unilateral exit from the tree, to cooperate.  The client can abort this assisted exit, and it would not be much different from the client simply refusing to perform an assisted exit and forcing the LSP to perform a unilateral exit from the tree.

-------------------------

ZmnSCPxj | 2024-10-09 14:04:01 UTC | #27

For the "Inversion of Timelock Default" case, the following question may be raised: if the client performs unilateral exit by passively waiting for the timelock default so that the LSP is forced to perform (and pay for) unilateral exit on behalf of the client, how can the client enforce HTLC timeouts for outgoing payments?

And the simple answer is: it does not have to (as long as the client is not secretly a forwarder, i.e. if the client is really just an end-user that never forwards payments)!

If the HTLC timeout is shorter than the timeout tree timeout, then the LSP can very well simply not fail the HTLC and not drop unilateral exit.  However, the LSP gains no advantage; either it knows the preimage or not.  If the LSP did not successfully forward to the final destination and refuses to fail the HTLC, this is equivalent to the LSP refusing service to the client; then the client can simply force a passive unilateral exit and recover its funds, including the outgoing HTLC.  If the LSP still forwards the payment (in order to learn the preimage) after the client-offered HTLC has timed out, it is at the risk of the LSP: the client ***might*** still have onchain funds to pay for exogenous fees, and the nodes have P2A outputs that the client can use (a truly destitute client might go to a competitor LSP, give them the signatures of the tree nodes, and let the competitor LSP punish the misbehaving LSP).  Thus, the incentive for the LSP is to respect the timeout of the HTLC, and to fail the HTLC before the timeout.  The LSP can only lose a customer if it misbehaves this way.

This is risky if the client is forwarding: the upstream node may enforce the upstream timeout.  However, the expectation is that the client is a plain end-user that never forwards.

For HTLCs offered by the LSP, we have already given the assumption that the LSP is capable of paying for onchain fees to do unilateral closes via exogenous fees.  Thus, even though the LSP is only a forwarder, it can enforce the timeout on HTLCs it offered to the client.

-------------------------

ariard | 2024-10-09 19:26:22 UTC | #28

Thanks for the clarification.

Let's recall some fact for the public bitcoin community:

- few months agos, a block inc employe A is making an [invitation list](https://github.com/lightning/bolts/issues/1171#issuecomment-2391532275) for a so-called Lightning Dev Summit and apparently deliberately excluding people who are scientifically critical to their technical ideas
- during the time that meeting happening, you, a block inc employe B publish out of nowwhere, without any peer review that one expect at academic or industry venue, that SuperScalar construction
- during that meeting happening, another block inc employe C, [shows the light](https://github.com/lightning/bolts/issues/1171#issuecomment-2365479410) on SuperScalar construction, making the call to discuss that construction
- after that meeting, you, a block inc employe B and another block inc employe D are using the fact that this construction was discussed at that recent Lightning Dev Summit, as an argument of authority, or at least I'm under that impression, that this construction has been peer-reviewed in any meaningful way

Correct me, if I'm wrong on the fact, though I think it's a fair description of the fact.

I don't know if you're familiar with the history of Bitcoin protocol development (here the Wikipedia [link](https://en.wikipedia.org/wiki/Bitcoin_scalability_problem)). During the block size war, in 2016, there was something called the Hongkong Agreement, where some devs attended and industry participants, and while some more senior devs discourage other folks to attend, there was some pow-mow style agreement born out of that on the future of bitcoin scalability.

That agreement, which was done in a complete opaque fashion for the rest of the bitcoin community and maybe without considering the maths, economics and network aspects of discussed topics with distant minds, gave the birth years latter to many controversies and was used as an argument to plan the Segwit2x hard fork.

In the spirit of avoiding future controversies in the community, I think it would be great if you
could highlight that this SuperScalar constructions represents only the view of the block inc employes. Or worst if  has been also vetted in a pow-mow fashion by the other lightning protocol devs present at this summit, and they’re weighing their experts opinions on it. Far from me to accuse block inc to engage in a embrace, extend, extinguish stragegy a la microsoft about the lightning protocol, but many in the communities can have real doubts. At least, when
blockstream folks published their landmark paper about sidechains in 2014, there were more vocal about their vested interest in the adoption of this blockchain technology (— and the company CEOs were old cypherpunks themselves).

I don't think the lightning community has put so much years of works in getting a decentralized network of payment, with instant and private settlements, to fall into some very ill-designed
off-chain construction, with a lot of trust in the LSP. If block inc is designing a product only for the US market, where there are legal framework in case of issues, it's better to be more verbose about it (-- I'm not going to say that Jack Dorsey's tweets are misleading between what he's saying and what block inc is really doing, but why TBD hasn't released yet a LSP ?).

Edit: corrected  s/defamatory/misleading/g. it’s not the same.

Personally, I'm more interested in a decentralized network where a lambda user in El Salvador
or Oman can pay another user in Seoul or Singapor, while relying only on the trust-minimized
assumptions of the protocol. A protocol working well in war zones and developing countries,
where there is not always a stable nation-state authority and a legal remedy is not an option.

> Feel free to attack the actual Lightning Network to demonstrate the problem. Clearly, you believe:

As far as I know, I've never seen any CVE or responsible disclosure associated to your name in the bitcoin field. You might have done so in the past in another industry as an infosec researcher, and if so you're free to communicate such realizations.

In the lack of such information, I don't really think you're familiar with ethical disclosure as a security researcher when end-users money are at risk, and as such I'll let your naive remark
here. Sorry, not sorry.

> If so, you can demonstrate this, today, with the actual Lightning Network.

I don't think you're understanding how the "Forced Expiration Spam" laid out in the lightning whitepaper of 2015 characterize a systemic risk in a world where the bitcoin blockchain space is limited and the number of coins limited to 21 millions.

Let's me re-give you an explanation.

Post-segwit, blocks are limited to 4MB in weight units. Under current lightning protocol safety timelocks of 144 blocks in average and assuming commitment transactions of 300 weight units in average, the block chain can only process 13k channels per bloc. That means the blockchain can only process 2M of channel per day in the worst-case scenario of everyone force-closing at the same time.

Failing to confirming on-time a commitment transaction and the corresponding HTLC-preimage or HTLC-timeout leads to loss of funds for one of the lightning counterparty.

This is the best scenario, I'm not considering fully charged lightning channels with `max_accepted_htlc` limits number of HTLCs, neither second-stage transactions and usage of lower than 144 blocks, as done in practice by lightning implementation. We have only 44k public channels in lightning today, though probably another order of magnitude of private channels, so ~500k. Massive force expiration spam could already be an issue for today's Lightning Network, if we see such forced expiration spam happening (or in other words a bank run).

Your SuperScalar proposal, even if it does not implies a consensus change, only makes the problem worst, as now all the combinations of the full tree of transactions might have to be confirmed on-chain. Corresponding fee reserves have to be provisioned by the lightning routing nodes, or the end-users lightning wallet, who ever is paying the fee. So if you assume timeout tree with a k-factor of 2 and 8 users, that's 12 transactions that have been to confirm in the worst-case. 12 transactions, that's more than the 8 commitment transactions that have to be confirmed in the worst-case scenarios, rather than 8 commitment transactions, and there is no, or only small compression gain as the k-factor has to be materialized in the fan-out output at each intersection of your timeout tree.

So, I'll re-say what I've already said the SuperScalar construction is broken and only makes the systemic risk that is already happening with open lightning channels, in a world where the size of the block is statically limited. Pleasure to discuss the mathematics and the economics, if there is still some persistent wondering.

I think that problem has been understood for years by many bitcoin protocol experts, and again it's described in the lightning whitepaper section 9.2

-------------------------

ZmnSCPxj | 2024-10-09 22:23:15 UTC | #29

>Correct me, if I’m wrong on the fact, though I think it’s a fair description of the fact.

Sure --- people who work in the same company tend to work together on the same stuff.  SuperScalar was released in a limited venue internal to Block, before we presented it at the summit.

The initial SuperScalar was lousy --- it was just laddered timeout trees, without the Decker-Wattenhofer.  This was developed a few months ago, internally to Block, and not released publicly because it ***sucked***.  A few weeks ago, while looking over @instagibbs work on P2A, I realized that P2A handled the issues I had with Decker-Wattenhofer --- in particular, the difficulty of having *either* exogenous fees (without P2A, you need every participant to have its own anchor output, increasing the size of each tx with an output per participant) *or* mutable endogenous fees (because not every offchain transaction is changed at each state change, earlier transactions cannot change their feerates when you update the state for a feerate change), which is why I shelved Decker-Wattenhofer constructions and stopped work on sidepools, which used Decker-Wattenhofer.  However, with P2A, I realized that Decker-Wattenhofer was actually more viable --- and thus can be combined with timeout trees, creating the current SuperScalar.  I then rushed to create a quick writeup, got it reviewed internally, and got permission to publish it on Delving, so we could present this at the summit.  @moneyball believes it provides a possible path to wider Lightning onboarding, so this encouraged me to focus more attention on it and figuring out its flaws and limitations.

(You can track my thinking around laddered timeout trees by my Twitter posts, incidentally --- again, I only started posting about timeout trees, and musing on laddering them, in the past 6 months. Thinking takes time.)

Adding Decker-Wattenhofer to laddered timeout-trees was an idea that occurred to me literally a few weeks before the summit.  Again, remember that I stopped working on sidepools because Decker-Wattenhofer ***sucked*** (exogenous fees are out because you need an output per participant, endogenous fees cannot be meaningfully changed because each state transition only changes a subset of offchain transactions), and I only returned my attention to Decker-Wattenhofer after @instagibbs could release P2A for Bitcoin Core 28.  Without thinking about Decker-Wattenhofer, I cannot combine Decker-Wattenhofer with timeout-trees, obviously.  The timing was coincidental, not planned.  Thinking takes time.

I have not made any representation that the construction has been peer-reviewed meaningfully, even internally in Block.  Given the addenda I have been writing, it is very much a work-in-progress, one that I am taking all of my time to work on rather than anything else.  If anyone can poke holes into it, they can do so in this very thread, that is the whole point of making this thread and presenting it at the summit.  I am in fact taking the time here to allow people to respond.  All I did was present this to the people at the conference, and I believe it to be worth my time to think about and refine.

> If block inc is designing a product only for the US market, where there are legal framework in case of issues, it’s better to be more verbose about it 

I have been advised to avoid mentioning of legal or regulatory stuff, as I am not a lawyer and anything I say about regulations, legal frameworks, or regulatory bodies would not be expert advice on it.  Let me ask my supervisor about this.

> So if you assume timeout tree with a k-factor of 2 and 8 users, that’s 12 transactions that have been to confirm in the worst-case. 12 transactions, that’s more than the 8 commitment transactions that have to be confirmed in the worst-case scenarios, rather than 8 commitment transactions, and there is no, or only small compression gain as the k-factor has to be materialized in the fan-out output at each intersection of your timeout tree.

We are aware of this issue and this issue was also presented at the Lightning Proto Dev Summit.  However, the tree structure does allow for subsets to sign off on changes instead of requiring the entire participant set to be online to sign; this reduces the need for large groups to come online. @adiabat last year in TABConf had a talk about how onlineness (specifically, coordination problems with large groups) will be the next problem; tree structures allow us to reduce participant sets, regardless of consensus change, and tree structures do still require a multiple of N data to publish N leaves.  Full exits of `OP_TLUV`-style trees are even worse, as they require O(N log N) instead of O(N) data (m * N data to be specific, where `m` is reduced by higher arity).  `OP_TLUV`-style trees also cannot change subset without changing the root, which means they do not gain the ability of SuperScalar to have subsets sign off on changes; this is because `OP_TLUV`-style trees use Merkle trees, unlike timeout trees which use transaction trees which allow sub-trees to mutate if you combine it with Decker-Wattenhofer.

I have been refining SuperScalar to shift much of the risk to the LSP, precisely to prevent risks on clients.  You may not agree that it goes far enough, but I think it can be done in practice such that it is more economical for the LSP to not screw over its clients, just as typical capitalism works.  You cannot earn money from a dead customer, which is why capitalism works.

The thing about SuperScalar is that timeout trees suck because of the large numbers of transactions that need to be put onchain in the worst case, Decker-Wattenhofer sucks because of the large numbers of transaction that need to be put onchain in the worst case, but when you mash them together they gain more power while ***not*** increasing their suckiness --- their suckiness combines to a common suckiness instead of adding their suckiness together.  Timeout trees get the ability to mutate due to combining with Decker-Wattenhofer, while Decker-Wattenhofer gets the ability to mutate using partial participant sets.  The whole is better than the sum of its parts, and I think it is worth my while to investigate if this is ***good enough in practice***.

-------------------------

ariard | 2024-10-11 22:34:21 UTC | #30

Thanks again for the clarifications

> Sure — people who work in the same company tend to work together on the same stuff. SuperScalar was released in a limited venue internal to Block, before we presented it at the summit.

Be sure, I don't question a private company to seek profit, neither people working at the same company working together on the same stuff, that's all right.

I'm questioning if all of that it's not a bit of corporate capture of the communication channels and venues dedicated to the Lightning protocol. Such protocol has been developed in common by different implementations since the Milan meeting around Scaling Bitcoin in 2016, and all the protocol specifications have been released under Creative Common License 4.0 since then.

Such meetings have been usually reserved to discuss matters related to the lightning protocol, and not to present commercial products in exclusivity. E.g, the folks at Lightning Labs, has never used that to talk about their Lightning Pool product, that they released in late 2020, and I think it would have been inappropriate for them to do so.

Moreover, and here I'll recall the example of Blockstream in 2014, when they released the sidechain paper, this was explicitly licensed into the public domain. Concerning the SuperScalar construction, given it has been developed internally at Block Inc as you said so, there could be patent protection in application. As far as I can see, there has been no such mention when you published your post few weeks ago, and this forum has moderation rules noticing about “deceptively encouraring the use of patent-encumbered techniques" as documented here:
https://delvingbitcoin.org/t/flagging-posts-topics/1085

Beyond, and here it's more concerning the lightning summit attendees, has any Block Inc employe explicitly said to the non-Block attendees that the SuperScalar construction was a Block Inc product, before to make a presentation about it ? If yes, have they give opportunity to
the non-Block attendees to leave the room or not attend the session if they didn't wish to discuss SuperScalar ? For the ones who did attend a session on it, as all materials presented to them being explicitly put into the public domain or a creative common license ?

As a reminder, it's not liked "closed-door" meetings, where only some developers were invited have raised many controversies in the past, of which the Hong Kong Agreement of 2016 is a good example.

So far there has been no answer on a public forum or channel from TheBlueMatt, a Block Inc employee, how the invitation list has been composed: https://github.com/lightning/bolts/issues/1201 and if any technical criterias has been followed. 

Some in the community could have doubt if it wasn't just opportunistically to lobby some
lightning devs about BlockInc's SuperScalar product ?

Concerning lightning summits, I think Rusty Russel sets a good standard in the past in matters of open-source protocol meetings, where the invitation for the Adelaide meeting of 2018 were announced on the mailing list: https://lists.linuxfoundation.org/pipermail/lightning-dev/2018-April/001168.html

I certainly don't wish to accuse TheBlueMatt of hypocrisy or double-standard on a public forum in matters of open-source, neither that he would forget his open-source standards everytime he's changing of corporate employer. After all, I contributed with him for years on the `rust-lightning` open-source project.

> The initial SuperScalar was lousy — it was just laddered timeout trees,
> without the Decker-Wattenhofer. This was developed a few months ago,
> internally to Block, and not released publicly because it sucked. A few
> weeks ago, while looking over @instagibbs work on P2A, I realized that
> P2A handled the issues I had with Decker-Wattenhofer — in particular,
> the difficulty of having either exogenous fees (without P2A, you need
> every participant to have its own anchor output, increasing the size of
> each tx with an output per participant) or mutable endogenous fees (because
> not every offchain transaction is changed at each state change, earlier
> transactions cannot change their feerates when you update the state for
> a feerate change), which is why I shelved Decker-Wattenhofer constructions
> and stopped work on sidepools, which used Decker-Wattenhofer. However,
> with P2A, I realized that Decker-Wattenhofer was actually more viable —
> and thus can be combined with timeout trees, creating the current
> SuperScalar. I then rushed to create a quick writeup, got it reviewed
> internally, and got permission to publish it on Delving, so we could
> present this at the summit. @moneyball believes it provides a possible
> path to wider Lightning onboarding, so this encouraged me to focus more
> attention on it and figuring out its flaws and limitations.

I don't know if you're reading the mailing list, though the timeout trees concept was presented there. If I remember numerous limitations were pointed out. Beyond, it's not like the P2A sounds to be broken too, as a fee-bumping scheme for off-chain counterparties with competing interest.

> (You can track my thinking around laddered timeout trees by my Twitter
> posts, incidentally — again, I only started posting about timeout trees,
> and musing on laddering them, in the past 6 months. Thinking takes time.)

I'm not on Twitter. Social medias culture can only make you dumb, and
more inclined to follow the madness of the crowd. Good to read again the
philosopher Hannah Arendt, her writings on the crisis of culture or some
of her essays on the essence of totalitarism.

> Adding Decker-Wattenhofer to laddered timeout-trees was an idea that occurred
> to me literally a few weeks before the summit. Again, remember that I stopped
> working on sidepools because Decker-Wattenhofer sucked (exogenous fees are out
> because you need an output per participant, endogenous fees cannot be meaningfully
> changed because each state transition only changes a subset of offchain transactions),
> and I only returned my attention to Decker-Wattenhofer after @instagibbs could
> release P2A for Bitcoin Core 28. Without thinking about Decker-Wattenhofer, I
> cannot combine Decker-Wattenhofer with timeout-trees, obviously. The timing was
> coincidental, not planned. Thinking takes time.

The main issue with Decker-Wattenhofer, whatever the elegance of the construction, is that for each state update, the relative timelocks are decremented and when they reach near of expiration, you can suddenly have massive surface of transactions to fee-bump, at the worst time of block demand.

On the other hand, the long safety timelocks can only make the construction very burdensome for the user, as they would have to wait until their on-chain expiration if there is an early exit.

> I have not made any representation that the construction has been peer-reviewed
> meaningfully, even internally in Block. Given the addenda I have been writing, it
> is very much a work-in-progress, one that I am taking all of my time to work on rather
> than anything else. If anyone can poke holes into it, they can do so in this very thread,
> that is the whole point of making this thread and presenting it at the summit. I am in
> fact taking the time here to allow people to respond. All I did was present this to the
> people at the conference, and I believe it to be worth my time to think about and refine.

Again, see all the comments above about making the timing of this publication at the same time than the summit and that being explicitly pointed out by another Block Inc employe in the github thread about the lightning dev summit.

In 2017, the way the extension block proposal was brought into the public conversation too raise controversies, when it was published on a Medium post, rather than usual communication venues.

> I have been advised to avoid mentioning of legal or regulatory stuff, as I am not a
> lawyer and anything I say about regulations, legal frameworks, or regulatory bodies
> would not be expert advice on it. Let me ask my supervisor about this.

Why your supervisor cannot comment here on this public forum on this name ? That’s only sounds more like corporate capture of the lightning protocol, when questions are asked on how a "closed-door" meetings, did happen. Some folks it was just pure hazard about the timing of publication and meetings happening, and when asked more questions someone falls into the "let me ask to my boss.." as they were just doing Silicon Valley-style bad public relationships…

Corrected: s/this name/its name/g - English can be hard.

Personally, I'm fine too to talk about technical and legal matters in a public fashion, and I think few other bitcoin and lightning protocols devs are versed too in legal matters.

Again, it's not like your top-down hierarchical supervisor, Jack Dorsey might freely have given in the past his contacts to some devs in that space and some people can go to ask in private explanations if someone wishes to know more. But if a supervisor at Block Inc is not able to explain on a public why there are some irregularities in the organization of a "closed-doors" meeting about an open-source protocol, this can only raise doubts among the bitcoin community what that SuperScalar product is all about.

> We are aware of this issue and this issue was also presented at the Lightning
> Proto Dev Summit. However, the tree structure does allow for subsets to sign off
> on changes instead of requiring the entire participant set to be online to sign;
> this reduces the need for large groups to come online.

If some participants are signing off the changes, the construction is broken,
as they cannot verify that ulterior state changes are correct, and the LSP
and another subset of the group colludes to double-spend the balance. The
whole lightning security model is about not trusting the channel counterparty
with loss of funds style risk, or the Lightning Service Provider, for what
the LSP is worth.

> @adiabat last year in TABConf had a talk about how onlineness (specifically,
> coordination problems with large groups) will be the next problem; tree structures
> allow us to reduce participant sets, regardless of consensus change, and tree
> structures do still require a multiple of N data to publish N leaves. Full exits
> of OP_TLUV-style trees are even worse, as they require O(N log N) instead of O(N)
> data (m * N data to be specific, where m is reduced by higher arity). OP_TLUV-style
> trees also cannot change subset without changing the root, which means they do not
> gain the ability of SuperScalar to have subsets sign off on changes; this is because
> OP_TLUV-style trees use Merkle trees, unlike timeout trees which use transaction
> trees which allow sub-trees to mutate if you combine it with Decker-Wattenhofer.

Lol, Tadge saying that onliness and coordination problems with large groups will
be the next problem. It's not like something that the OG working on Lightning
have known for years...and even some more.

Sure, OP_TLUV-style trees cannot changes subset without changing the root, however
_**if done correctly**_ they do not require trust in any of the other counterparty about
integrity of the balance.

> I have been refining SuperScalar to shift much of the risk to the LSP, precisely to
> prevent risks on clients. You may not agree that it goes far enough, but I think it
> can be done in practice such that it is more economical for the LSP to not screw over
> its clients, just as typical capitalism works.

That's missing the point about the discussion about levels of security risks. What
you're presentation of this SuperScalar product is saying, is that the LSP can
rug pull at anytime one of the user, so it's a loss of balance security risk. Not
a simpler risk like a delay in processing due to the bitcoin CSV timelocks.

In the real world, there is something call "bank run" and as one put it
into the bitcoin blockchain years years ago, "Chancellor on brink of
second bailout for banks”.

> You cannot earn money from a dead customer, which is why capitalism works

I'll let you dig into the etymology of mortgage, a financial instrument underpinning
many of business operations in modern capitalism.

> The thing about SuperScalar is that timeout trees suck because of the large numbers
> of transactions that need to be put onchain in the worst case, Decker-Wattenhofer
> sucks because of the large numbers of transaction that need to be put onchain in
> the worst case, but when you mash them together they gain more power while not increasing
> their suckiness — their suckiness combines to a common suckiness instead of adding their
> suckiness together. Timeout trees get the ability to mutate due to combining with
> Decker-Wattenhofer, while Decker-Wattenhofer gets the ability to mutate using partial
> participant sets. The whole is better than the sum of its parts, and I think it is
> worth my while to investigate if this is good enough in practice.

That SuperScalar construction still does not solve the "Forced Expiration Spam”
about a massive number of off-chain state transactions hitting the blockchain,
as whoever is paying the on-chain fees, be it the LSP or the channel counterparty,
there is a limited blockchain space (4MB) and a limited number of coins (`MAX_MONEY`)
that can be used to pay the fees. The LSP can inflate the timeout trees, with actually no fee-bumping reserves to back them up, so it's clearly trusted and the LSP can steal the participant sets at anytime.

But sure, what you're doing or what Block Inc is doing you it's up to you guys.

The rest of the lightning community, they will prefer something that scales bitcoin in a more trust-minimized fashion.

-------------------------

ZmnSCPxj | 2024-10-14 01:08:18 UTC | #31

I can state that Block has no intention of patenting SuperScalar or otherwise limiting the use of SuperScalar. The entire point of publishing the damn thing on delving and in presenting it to the summit was to bring it out to the public, WTF. There are no copyright claims or similar because a lot of these is just initial design notes.  I am designing it in public, right here, on delving.

I have been advised to not engage you in anything that is non-technical.  Given that your technical expertise has not raised anything that has not been raised before, I am also no longer engaging you in anything technical, either.

-------------------------

ariard | 2024-10-17 22:42:34 UTC | #32

> I can state that Block has no intention of patenting SuperScalar or otherwise limiting the use of > SuperScalar. The entire point of publishing the damn thing on delving and in presenting it to > the summit was to bring it out to the public, WTF.

If all what Block Inc did in the organization of this summit where SuperScalar was presented in “exclusivity” to selected devs, and respectuous of how Lightning development has been done in the open-source fashion, since 2016 and after since then There is still no answer from one of the Block Inc employe on how it was really organized.

This is not like I've myself organized open-source protocol dev meetings in the past, making abstraction of people backgrounds or the organizations (be it for-profit, non-profit or whatever) there were representing to concentrate the discussion on purely technical matters.

This silence from Block Inc is very speaking in itself…

Going back to SuperScalar, and the chain economics and deep technicals here.

Let's say you have the initial transaction with the LSP and all the users, i.e the root transaction.

Under, Decker-Wattenhofer update mechanism, channel factories have two stages: kick-off and state transaction, spending the root transaction. Each state transaction has a decrementing timelock and attached to this state transaction, there is a timeout tree, where after a timelock X, either the use should have come back online to interact with the LSP to update the tree, or the LSP (+ some users to sign the multisig of rhe state transaction) can evict out of the tree the user.

There is a k-factor at each output of the state channel factory transaction, to branch off and fan-out the users in the subtrees.

So if you assume a k-factor of 2 (at each branching of the timeout tree) and 8 users, that's 12 transactions that have to confirm in the worst-case. That means in the worst-case, either the user (if they wish to make a fully non assisted exit) or the LSP must have on-chain amounts available to confirm the 4 transactions constituting the path before the safety timelock expiration. For the LSP, it's even worst as they must have liquidity for all the combination of the timeout tree, and this when mempool networks might be full.

So, I'll re-say what I said above in one of my previous post, under current block size (4 MB) and
the maximum number of bitcoins, that can be used to pay the fees, I don't see how SuperScalar works at all under "Forced Expiration Spam" as described in the section 9.2 lightning whitepaper. As a reminder, that problem was well-described by protocol experts before I was involved in bitcoin dev, so don't take the shortcoming I'm pointing too about SuperScalar as ad hominem here.

-------------------------

