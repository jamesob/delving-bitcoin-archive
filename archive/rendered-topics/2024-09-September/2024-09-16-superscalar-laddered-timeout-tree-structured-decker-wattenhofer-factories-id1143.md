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

ZmnSCPxj | 2024-09-30 00:02:24 UTC | #15

>Also any assisted exit sounds a double-edged sword mechanism as it could be leveraged to kick out a user, at the worst time e.g when the on-chain fees are full.

An assisted exit is a PTLC in the in-factory construction from user to LSP, followed by an onchain PTLC from LSP to user, with the scalar being the private key that client is using for the in-factory constructions.  It is ***not*** the LSP initiating a unilateral exit for a user and assisting it by paying for onchain fees.  The client is the one that initiates this; it has to actually offer a PTLC inside the in-factory construction before the LSP can safely put the swap PTLC onchain.  Thus, the client can choose to ***not*** initiate an assisted exit during high-fee times.  The LSP has no incentive to eject clients during high fees.

An assisted exit like this allows the LSP to recover its in-factory funds earlier. If all clients perform assisted exits (including assisted exits from the current factory to a newer factory) then the LSP can recover funds directly from the funding outpoint, ***and*** can get those funds ***immediately***, before the end of the timeout period (which is safe since all the clients have exited; the entire fund is now solely owned by the LSP). This is in contrast with evicting clients by unilaterally exiting them: the LSP has to wait for some time due to the extra Decker-Wattenhofer steps along the way, and has to actively pay for fees if the LSP does it during high-fee periods.  That is, the LSP has an incentive to keep clients in the tree until the clients perform an assisted exit, because the assisted exit lets the LSP use less blockspace to recover its funds.  It has nothing to do with the LSP fee policy, simply the pragmatic realization that the private key of the client is valuable (and the client can safely hand it over in exchange for getting its funds out of the construction and into another one, either onchain or to a new factory) because it allows the LSP to close the construction earlier and with just a simple 1-input 1-output tx with a single signature.

-------------------------

