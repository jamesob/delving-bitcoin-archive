# SuperScalar: Laddered Timeout-Tree-Structured Decker-Wattenhofer Factories With Pseudo-Spilman Leaves

ZmnSCPxj | 2024-11-04 15:40:11 UTC | #1

Subject: SuperScalar: Laddered Timeout-Tree-Structured Decker-Wattenhofer Factories With Pseudo-Spilman Leaves

# Introduction

We introduce the LSP Last-Mile Problem:

* New users receiving their first ever Bitcoins over the Lightning Network must pay for incoming liquidity.  
  - The cost of blockchain operations to get that liquidity must be amortized across multiple new users to keep the cost of each one low.

In addition to the above problem, our solutions must also have the following constraints:

* We must ensure that the LSP cannot steal funds, i.e. it is not sufficient to have one-honest-member security assumption, unless every end-user can be their own single honest member (i.e. "your keys, your coins").
  - By not requiring a one-honest-member security assumption, a single LSP can provide liquidity to clients, independently of other entities, without needing there to be large numbers of entities that form federations to mitigate against the one-honest-member security assumption.
* We must do so without any blockchain consensus changes.  
  - The Bitcoin blockchain has ossified in practice. Due to the scheduled halvening every 4 years causing a sudden increase in Bitcoin price, leading to a sudden increase in interest in Bitcoin, large batches of new users join the set of entities with an interest in Bitcoin consensus. As the previous batch becomes convinced of some consensus change, the new batch enters Bitcoin, which must itself be convinced of the same consensus change. If a consensus change cannot achieve practical consensus within a time shorter than a halvening period, it will not ever happen, thus leading to ossification in practice (smaller and more easily-digestible changes may still push through, but more complex changes, like covenants, will never reach consensus).  
* We must be resilient against some or a few end-users being offline when the LSP has to reallocate funds.  
  - As the number of users sharing a single UTXO increases, the probability of one or more of them being offline increases. Thus, when scaling to large numbers of end-users, resilience against some of the end-users not being able to come online is a necessity.  
  - It turns out that software running on mobile phones can be made to come online via Android or iOS application notification mechanisms, thus in practice the onlineness of mobile clients can be reasonably high. Nevertheless, mobile phones may occassionally drop off network at times, so their uptime is still not as good as non-mobile devices. Thus, the mechanism should be resilient against a few users being offline.
* New users should be able to start out with 0 Bitcoins.
  - They may end up with substantial amounts of coin, but we desire to support new users that have no Bitcoin at all.
  - Even existing users with cold-storage wallets may be unwilling to move funds from their cold-storage wallets; they may prefer to be paid their salary (or other compensation) in Lightning, then move any extra funds to their cold-storage wallet.
    This makes such existing users look no different from 0-Bitcoin new users.

All of the above constraints immediately rule out Ark and BitVM2 bridges. Both have a one-honest-member security assumption without covenants. Both can get around the one-honest-member assumption only if all end-users are simultaneously online at the moment at which an LSP needs to reallocate funds, which breaks the third constraint above.
The last constraint rules out JohnLaw2 Tunable Penalties, as the penalty output ("`In[...]` transaction outputs in the tunable penalties paper) cannot be 0 amount, unless the LSP is willing to outright give money to clients (an exchange might be willing to onboard new users if they buy a minimum amount, with part of the bought amount being the penalty, and otherwise take the role of an LSP, but that is out of scope of our constraints as stated).

In this writeup, I provide a construction, SuperScalar, which is effectively laddered timeout-tree-structured Decker-Wattenhofer channel factories with pseudo-Spilman leaves.

# Digression: The LSP vs Client Difference

Our last constraint exists in order to support onboarding of new users to the Lightning Network.

Onchain, a new user only needs a fresh keypair, and can immediately receive Bitcoins.
On Lightning, however, the LSP Last-Mile Problem means that new users need to acquire incoming liquidity *first*, and then receive funds.

We can observe that the LSP and the client are not truly equal peers:

* The LSP likely has access to more Bitcoin liquidity than the client has.
* We expect that the LSP outright owns more Bitcoin than any single client, and is likely to own more Bitcoins than all clients.
  - Since onchain fees are based on bytes rather than Bitcoin value, the LSP has economies of scale in moving large amounts of Bitcoin onchain compared to smaller clients.
* Both of the above implies the LSP can more easily negotiate with miners, particularly on onchain feerate futures, which would allow LSPs to pay onchain fees that are, equivalently, flatter than public mempool fees.
  - End-users (clients) would prefer the more consistent feerate schedule, and the LSP can negotiate on behalf of its clients, passing on the flatter onchain fee it gets from negotiating onchain feerate futures with miners to the clients in the form of a fixed monthly subscription fee.

Thus, this writeup will also focus on forcing the LSP to provide onchain services on behalf of clients.
This enforcement means that the LSP may lose funds unless they provide onchain services, particularly unilateral exit, on behalf of clients.
The assumption is that LSPs are more capable of handling the risk inherent in onchain operation (varying feerates) than typical clients, and so the burden of unilateral close is placed on LSPs.

I call this an "asymmetric LSP-client scheme", as the LSP and its clients have different expected properties ("asymmetric").

# The Ingredients

First, I provide a hopefully gentle introduction to the four different components I combine to form this single construction:

1. Decker-Wattenhofer decrementing-`nSequence` offchain mechanisms.  
2. Timeout trees, particularly the variant that uses everyone-signs to emulate `OP_CTV`, which I call timeout-sig-trees.  
3. Pseudo-Spilman channel factories, a construction similar to Spilman unidirectional channels, but with N > 2 participants with some drawbacks compared to Spilman.
4. Laddering.

Feel free to skip sections you are already familiar with.
Do note that the Decker-Wattenhofer and Timeout trees are not "standard", and include additional sub-sections that are novel to this construction, and may be worth perusing.

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

### Old State Poisoning

In an asymmetric LSP-client scheme, there is an LSP that provides access to the rest of the Lightning Network to its clients.

When using Decker-Wattenhofer to host channels, we can arrange that the LSP is responsible for unilateral exit.
If so, the LSP may have an incentive to cheat by using old state.
In particular, if we assume that clients will most likely not have an external UTXO with which to pay for unilateral exit exogenous fees, then clients have no practical ability to publish the latest state.

To protect against the LSP attempting to use old state, we can instead make it so that old state becomes punishable.

As a concrete example, the state outputs for the rightmost state transaction, in a Decker-Wattenhofer used for a channel factory with the LSP `L` and two clients `A` and `B`, might look like:

```
    nSequence
   +-----+---------+
   |     |   A&L   | LN channel
   |     +---------+
-->| 432 |   B&L   | LN channel
   |     +---------+
   |     |  A&B&L  | LSP liquidity stock
   |     |or(L&CSV)|
   +-----+---------+
```

The "LSP liquidity stock" is an amount that the LSP has inside the factory, which it can use to add more liquidity to the channels with `A` and `B`.

Suppose that the above state transaction is replaced with a newer one with lower `nSequence`.
Then there is now a newer state transaction, like so:

```
    old state tx
    nSequence
   +-----+---------+
   |     |   A&L   | LN channel
   |     +---------+
+->| 432 |   B&L   | LN channel
|  |     +---------+
|  |     |  A&B&L  | LSP liquidity stock
|  |     |or(L&CSV)|
|  +-----+---------+
|
|   nSequence
|  +-----+---------+
|  |     |   A&L   | LN channel
|  |     +---------+
+->| 288 |   B&L   | LN channel
   |     +---------+
   |     |  A&B&L  | LSP liquidity stock
   |     |or(L&CSV)|
   +-----+---------+
    latest state tx
```

We can then "poison" the old state so that the LSP is at risk if it publishes the old state:

```
    old state tx        +--+---+
    nSequence        +->|  | A |
   +-----+---------+ |  +--+---+
   |     |   A&L   |-+  +--+---+
   |     +---------+  +>|  | B |
+->| 432 |   B&L   |--+ +--+---+
|  |     +---------+    +--+---+
|  |     |  A&B&L  |    |  | A |
|  |     |or(L&CSV)|--->|  +---+
|  +-----+---------+    |  | B |
|                       +--+---+
|   nSequence
|  +-----+---------+
|  |     |   A&L   | LN channel
|  |     +---------+
+->| 288 |   B&L   | LN channel
   |     +---------+
   |     |  A&B&L  | LSP liquidity stock
   |     |or(L&CSV)|
   +-----+---------+
    latest state tx
```

This poisoning means that the LSP has a strong incentive to ensure that old state is never published:

* The Poon-Dryja channels `A&L` and `B&L` on the old state have had all commitment transactions revoked.
  - The only non-commitment transaction available is the above shown "poisoning" transaction, that puts all the funds into `A` or `B` unilaterally.
    But because the LSP does not own any money in the outputs of those transactions, it does not want the "poison" transaction published.
  - The only other transaction the LSP can use would be one of its own commitment transactions.
    But all commitment transactions on the old Decker-Wattenhofer state have been revoked, so the LSP would also still lose all its funds in the channel, same as the above "poisoning".
  - The client `A` or `B` can always just publish the poisoning transaction.
* The LSP liquidity stock cannot be spent unilaterally by the LSP until the `CSV` condition is met.
  - However, the "poisoning" transaction which splits the LSP liquidity stock amongst all the clients is valid ebfore the `CSV` condition is met.
    Further, because each client has unilateral control of an output, any client can trivially CPFP the "poisoning" transaction.

Thus, if the LSP has a high-enough uptime, once it sees the Decker-Wattenhofer kickoff transaction, the LSP has a strong incentive to ensure that the latest, unpoisoned state is confirmed, and will be willing to pay onchain fees to ensure that.

Of note is that with nested Decker-Wattenhofer layers, only the rightmost state transaction (the one furthest from the funding outpoint) needs to have its output poisoned.

Another note is that because the poisoning is fixed, the protection it provides is limited.
For example:

* Suppose we start with the state below:
  * Channel with `A`: 10 units
  * Channel with `B`: 10 units
  * LSP liquidity stock: 20 units.
* Then when the old state is poisoned:
  * Channel with `A`: entirely goes to `A`.
  * Channel with `B`: entirely goes to `B`.
  * LSP liquidity stock: split:
    * 10 units go to `A`.
    * 10 units go to `B`.
  * In total, `A` and `B` each get 20 units of protection.
* Suppose `A` never buys inbound liquidity, but `B` buys a lot of inbound liquidity.
  - In that case, `B` might have > 20 units of funds inside the Decker-Wattenhofer factory.
  - In that case, `B` is at risk of the LSP actually sockpuppetting `A` and then publishing old state; `B` only gets 20 units, but the LSP (by sockpuppeting `A`) gets the rest of the funds, effectively letting it earn the funds.
  - In general, the LSP cannot really present any proof that `A` is indeed a genuine client that is not a sockpuppet the LSP actually controls, so for maximum security for `B`, it should assume that `A` is a sockpuppet of the LSP `L`.

Against the above case, we can point out that `B` has > 20 units, and can probably afford to put a few units into an onchain UTXO that it holds in reserve for paying exogenous fees on unilateral close, so that `B` itself can ensure that the latest state is published before the LSP `L` can attempt to publish old state.
Thus, while the above problem exists, the simple solution is for `B` to move some funds onchain and reserve it for paying for unilateral close fees.
This reserve is *not* locked, unlike, say, the BOLT-specified channel reserve --- if the amount `B` has in the factory drops to below the protection level it has, it can swap the onchain fund back to the factory, and then spend all its funds in Lightning.
This concept can be presented to end-users as "deposit insurance"; the default case is "limited deposit insurance" (in the worst case, the client can only get back up to the poisoned protection limit), but if the client has substantial funds in the factory, it can pay to move some funds into a reserve fund and get "unlimited deposit insurance" that protects all their funds in the factory.

A weakness with this is that if the client has no external UTXO at all, and the kickoff transaction is 0-fee with a P2A output (i.e. exogenous fees), then if the LSP refuses service (e.g. always fails HTLCs of the client), the client has no practical way to force the LSP to begin the exit by publishing the kickoff transaction.
This weakness is acknowledged, as adding the next ingredient removes this weakness.

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

### Inversion of Timeout Default

In the original scheme, `L & CLTV` is used to enforce the timeout of the timeout-tree.
In short, at the end of the timeout, the LSP can get all the funds, and clients must proactively exit the timeout-tree before the timeout ends (ideally by swapping out to a new timeout tree, or to onchain, but in the worst case by unilateral exit).

An alternative is to instead enforce the timeout *against* the LSP, so that the LSP must proactively allow exit of clients from the timeout-tree (ideally by swapping them out to a new timeout tree, or to onchain, but in the worst case by unilateral exit).

That is, instead of an `L & CLTV` branch at each output, we have an alternate `nLockTime`d transaction, with `nLockTime` being the timeout of the timeout branch.
The transaction distributes the funds among the clients, without any funds going to the LSP.

As a concrete example, suppose the timeout tree has four clients `A` `B` `C` `D` and LSP `L`.
The leafmost outputs of the timeout tree then have four channels between the clients and the LSP.

```
                                    +--+-----+
                                    |  | A&L | LN channel
                                 +->|  +-----+
                      +--+-----+ |  |  | B&L | LN channel
       +---------+    |  |A&B&L|-+  +--+-----+
funding|A&B&C&D&L|--->|  +-----+
       +---------+    |  |C&D&L|-+  +--+-----+
                      +--+-----+ |  |  | C&L | LN channel
                                 +->|  +-----+
                                    |  | D&L | LN channel
                                    +--+-----+
```

In addition to the above tree of transactions, we also have, at each transaction output of the tree, an alternate, `nLockTime`d transaction:

```
                     nLockTime
                      +--+---+     nLockTime
                      |  | A |      +--+---+
                      |  +---+      |  | A |
                      |  | B |   +->|TO+---+
                   +->|TO+---+   |  |  | B |
                   |  |  | C |   |  +--+---+
                   |  |  +---+   |
                   |  |  | D |   |  +--+-----+
                   |  +--+---+   |  |  | A&L | LN channel
                   |             +->|  +-----+
                   |  +--+-----+ |  |  | B&L | LN channel
       +---------+ |  |  |A&B&L|-+  +--+-----+
funding|A&B&C&D&L|-+->|  +-----+
       +---------+    |  |C&D&L|-+  +--+-----+
                      +--+-----+ |  |  | C&L | LN channel
                                 +->|  +-----+
                                 |  |  | D&L | LN channel
                                 |  +--+-----+
                                 |
                                 |  +--+---+
                                 |  |  | C |
                                 +->|TO+---+
                                    |  | D |
                                    +--+---+
                                   nLockTime
```

The `TO` is the timeout of the mechanism, encoded in the `nLockTime` of the transactions.
The amounts of the unilateral client outputs in the `nLockTime`d transactions would be equal to the total channel capacity of that client with the LSP.
The `nLockTime`d transactions are signed at the same setup stage that creates the tree nodes.

Assuming the LSP does not impose a minimum reserve, but the clients do impose a minimum reserve on the LSP, and the channels start with all funds owned by the LSP, then the LSP always has something to lose at the timeout.
This means that the LSP has to ensure proper exit by the time of the timeout.
In the worst case, the LSP has to publish the tree towards a client that has not cooperatively exited.

How about if the client *has* drained its channel, and owns 0 amount in its channel with the LSP?

1.  If the client disappears afterward, the LSP can always publish the timeout-tree path to that channel before the timeout.
    The LSP is thus obligated to publish *and pay for* the unilateral exit of the client.
2.  Otherwise, the client may cooperately exit with the LSP.
    We have two options:
    - The exiting client can provide a signature for transactions that puts the transaction outputs along the path to its channel to an output controlled solely by the LSP `L`, without any `nLockTime`.
      Once all clients in a particular node have done this cooperative exit, the LSP has a fully-signed transaction that replaces that node, and can use that transaction instead to recover the funds without requiring that an expensive unilateral exit be published.
      Once all clients of the timeout tree have cooperatively exited, the LSP can recover the funds directly from the confirmed funding outpoint.
    - The exiting client can simply hand over the private key it used for the tree.
      Once all clients in a particular node have done this cooperative exit and handed over the private key, the LSP can generate the signatures for an arbitrary spend of the funding outpoint or tree node outpoint.

Of the two cooperative exit options:

* Cooperative signing:
  - For MuSig2 Taproot multisignatures, the client has to persistently retain the scalars behind the `R` shares it provides for the transaction that transfers the funds to LSP `L`.
    There is a risk here that client bugs or attempts to recover from old backups (e.g. automated cloud backups of mobile phone data) may cause the client to reuse `R`, which can leak a private key that can be reversed to the root private key.
    - As the transaction is fixed, some form of hashing of the private key and the transaction data may allow the client to never have to persistently retain the scalar behind the `R`s, beyond the requirement of root private key protection.
  - Less blockspace efficient, as the LSP has to first send recovered funds to an output to `L` before sending it onwards to whatever liquidity-management operation it must perform.
  - However, the client can use standard simple hierarchical derivation, and can reuse keys across trees.
* Private key handover:
  - More blockspace efficient, as the LSP has full control of the keys to get the recovered funds and can change the transaction (including batching multiple recoveries).
  - However, the client must use hardened hierarchical derivation, and must select new keys at each timeout tree, as it loses unilateral control of the key once it exits from the timeout tree.

To provide atomicity, the LSP and client can use a PTLC to create a pay-for-signature (for the cooperative signing option) or pay-for-scalar (for the private key handover option) in the destination (onchain or new timeout tree), to drain the funds the client has in the current timeout tree.
However, even without PTLCs, the cooperative signing or private key handover can be done as a separate step after the client has drained its funds from the current timeout tree.

*With* PTLCs, the client can offer multiple signature shares (for cooperative signing option), one for each tree node output it holds keys for, by providing the delta between the different signature shares, and selecting just one signature share for the PTLC itself.

#### Inversion of Timeout Default For Hosted Poon-Dryja Channels

If a timeout tree hosts Poon-Dryja channels, and the timeout tree has a limited lifetime, then logically the hosted Poon-Dryja channels must also have limited lifetime.
(In theory, the timeout tree allows channels to be published as channels onchain, to survive the lifetime of the timeout tree, but this is blockspace-inefficient, and the preference is to swap out to a direct onchain fund or channel instead if the client desires this.)

If so, the same technique that we use on timeout tree node inputs can be applied to the channel funding outpoints:

```
               nLockTime
                +--+-----+
             +->|TO|  A  |
             |  +--+-----+
             |
             |  +-------------...
             +->|commitment tx...
  +--+-----+ |  +-------------...
  |  | A&L |-+
->|  +-----+
  |  | B&L | (and similar here)
  +--+-----+
```

As the technique forces the LSP to publish timeout tree nodes before the end-of-lifetime, reaching into the hosted channel also forces the LSP to publish its commitment transaction before the end-of-lifetime.
Standard Poon-Dryja mechanisms then ensure that the LSP publishes its ***latest*** commitment transaction.

## Pseudo-Spilman Channel Factories

In the LSP last-mile problem specifically, the LSP has a number of bitcoins it wishes to provide to clients, so that clients can receive and send over Lightning.
Our assumption is that the LSP always sells liquidity, and, once sold, does not ever take it back (since clients paid for the liquidity; it does not want to have to refund already-sold liquidity).

We can thus observe that the LSP is effectively always a sender, while the client is always a receiver, except that instead of sending actual coin ownership, it sends liquidity.
That is, the LSP last-mile problem implies a unidirectional scheme, with liquidity coming from the LSP and being sent to its clients.

If there is only one client, such a construction is pointless, as all the money the LSP has in the construction can only go to that client, thus effectively the entire liquidity of the LSP is liquidity towards just that one client.
Thus, we need a unidirectional scheme transferring liquidity from the LSP to more than one client (perhaps better to call it a liquidity broadcast scheme).

Spilman channels are a classic unidirectional channel construction, with one sender and one receiver.
Given a sender `S` and a receiver `R`, and a starting fund of 10 units, the funding outpoint would be:

```
+----------+
|   R&S    |->
|or(S&CLTV)|10
+----------+
```

The `CLTV` can be a `CSV` instead, and put as an output of another offchain transaction that serves similarly to the kickoff transaction of Decker-Wattenhofer; this variation is left as an exercise to the reader.

The first time the sender `S` sends 1 unit to the receiver `R`, it creates a transaction that splits the amount between the sender `S` and the receiver `R`, signs it, and provides the signature to the receiver.

```
                +--+---+
+----------+    |  | R | 1
|   R&S    |--->|  +---+
|or(S&CLTV)|10  |  | S | 9
+----------+    +--+---+
```

The next time the sender `S` sends 1 unit to the receiver `R`, it creates an alternate transaction that splits out more funds to `R`:

```
                +--+---+
+----------+    |  | R | 1
|   R&S    |--+>|  +---+
|or(S&CLTV)|10| |  | S | 9
+----------+  | +--+---+
              |
              | +--+---+
              | |  | R | 2
              +>|  +---+
                |  | S | 8
                +--+---+
```

Of note is that the receiver `R` does ***not*** provide the complete signature to *any* of the transactions to the sender `S`.
Thus, `S` cannot unilaterally close at all.
Instead, the `CLTV` timelock ensures that `R` has to unilaterally close before the timeout.

As `R` has the choice of which unilateral exit, it will obviously select the one that gives the most money to itself.
Thus, the mechanism is unidirectional.

For the pseudo-Spilman I present here, instead of *replacing* old state, the mechanism *chains* new state on top of old state.
This has the disadvantage that unilateral exit cost is proportional to the number of updates.
However, we expect the number of liquidity reallocations to be smaller than the number of actual payments by at least an order of magnitude, so the cost of unilateral exit should still be reasonable (or at least, does not significantly rise above that of a timeout tree for a several dozen clients).

As a concrete example, if the LSP `L` has clients `A` and `B`, and has sold liquidity, first to `B`, and then to `A`:

```
                +--+----------+
+----------+    |  |   B&L    | 1 +--+----------+
|   A&B&L  |--->|  +----------+   |  |    A&L   | 1
|or(L&CLTV)|10  |  |   A&B&L  |-->|  +----------+
+----------+    |  |or(L&CLTV)| 9 |  |   A&B&L  | 8
                +--+----------+   |  |or(L&CLTV)|
                                  +--+----------+
```

The reason for chaining is that, because there are multiple receivers, all receivers need to exchange signatures so that any of them can unilaterally exit.
Now, let us take the point-of-view of client `A`.
Client `A` knows that it, the entity `A`, exists ("I think therefore I am").
However, it cannot know if it is inside or outside The Matrix (i.e. virtualization argument).
So, it is possible that the entity that calls itself `L` and the entity that calls itself `B` are secretly agents of The Matrix and not separate entities.
Thus, if `A` shares its signature with the entity calling itself `B`, it is possible that the entity calling itself `L` will also learn this signature.
If so, then `L` can publish old state onchain if we were to naively use the original Spilman setup of replacing old state instead of chaining new state on top of old state.

So that publication of old state becomes safe, each new state must strictly depend on old state, so that publication of old state also implies the possibility of publishing new state (or in other words, new state requires old state anyway).
The transaction output can only be spent exactly once, and the clients, including client `A` that knows itself to be real and that the rest of the world may or may not be The Matrix, can ensure the correct operation of the mechanism by simply not co-signing a spend.

## Laddering

Many financial institutions offer a kind of financial contract wherein a depositor puts funds into a contract, and cannot withdraw, even partially, until some specific future date, at which point the depositor is given the original funds, plus an interest payment. The contract is also non-transferable. Such contracts are known by various names:

* Certificate of Deposit (United States)  
* Guaranteed Investment Certificates (Canada)  
* Term Deposit or Time Deposit (other countries)

Such contracts are inflexible; as noted, it is impossible to withdraw or transfer the contract until the end of its term. However, savvy investors instead split up their investable funds into multiple such contracts, set up so that their termination dates are staggered by one month or one year to each other. This technique is called "laddering".

For example, an investor might have three such contracts, terminating in December 2024, December 2025, and December 2026\. On December 2024, the first contract terminates, and the investor may decide to withdraw part of the funds and re-invest the remaining in a new contract that terminates on December 2027, or to add more funds to invest in the new contract, or to start closing the ladder by not starting a new contract.

Laddering provides investors the ability to change the amount of investment, and to add or reduce their investment into these contracts, once a month or once a year, depending on the laddering. Thus, even though the base contracts are inflexible, laddering allows investors to regain a little bit of flexibility, while retaining the advantages of long-term certificates of deposit.

From the point of view of the LSP, any construct it creates to provide paid service to clients is an investment (it reserves some funds, and expects to earn more money than the cost of maintaining the construct).
Thus, any construct that has fixed times where the LSP can practically move its funds, can be considered to be similar enough to a certificate of deposit that laddering is practical for the LSP.

# The SuperScalar Mechanism

Laddered timeout-tree-structured Decker-Wattenhofer channel factories with pseudo-Spilman leaves are simply the combination of the above four ingredients.

## Timeout-tree-structured Decker-Wattenhofer

First, let me demonstrate the combination of the first two ingredients; we shall add pseudo-Spilman and laddering in a separate subsection later.

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

The above diagram shows the "standard" timeout-sig-tree.
However, we should use the "inversion of timeout default" technique in actual implementations, i.e. timeouts are implemented as an `nLockTime`d transaction that distributes channels to sole control of clients, and LSP-unilateral `L` outputs are split among clients.
How this looks is left as an exercise to the reader.

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

### Combining Decker-Wattenhofer Old State Poisoning With Timeout Tree Inversion Of Timeout Default

As noted in the Decker-Wattenhofer Old State Poisoning section, it has the weakness that, while a client can force the LSP to publish the correct latest state by confirming the kickoff transaction, the client *has* to get the kickoff transaction confirmed onchain first.
This can be arranged by putting a fee for the kickoff transaction that is reasonable for most times, but during fee spikes, if the client cannot pay exogenous fees, it cannot initiate a unilateral exit.

Fortunately, the inversion of timeout default of timeout trees forces the LSP to publish tree nodes of the timeout tree before the end of its lifetime.
And with timeout-tree-structured Decker-Wattenhofer factories, Decker-Wattenhofer kickoff transactions are *also* tree nodes.
Thus, the combination of both schemes covers that weakness of old state poisoning.

## Pseudo-Spilman Factories At Leaves

I now add the third ingredient.

Instead of having leaf tree nodes that are Decker-Wattenhofer mechanisms, we instead have "wide" leaves (i.e. more than the arity of the tree), and have pseudo-Spilman factories in addition to client-LSP channels.

For example, if the intended arity would have been 2, our leaves would instead have 4 clients, but with 2 pseudo-Spilman channel factories, each with 2 non-overlapping subsets of the 4 clients on the leaves:

```
       nSequence
        +---+---------+
        |   |   A&L   | LN channel
        |   +---------+
        |   |   B&L   | LN channel
        |   +---------+
        |   |  A&B&L  | pseudo-Spilman factory
        |   |or(L&CSV)|
(tree)->|432+---------+
        |   |   C&L   | LN channel
        |   +---------+
        |   |   D&L   | LN channel
        |   +---------+
        |   |  C&D&L  | pseudo-Spilman factory
        |   |or(L&CSV)|
        +---+---------+
```

Similarly, if the arity is 4, leaves would be state transactions with 16 clients and 4 pseudo-Spilman channel factories with 4 clients each.
Generalizing, for arity of `k`, leaves would have `k^2` clients and `k` pseudo-Spilman channel factories with `k` clients each.

Then, when client `A` wants to buy inbound liquidity, and `B` is online and the sub-factory with `A` and `B` has available funds, the LSP can use the pseudo-Spilman factory to create a new channel with `A`:

```
       nSequence
        +---+---------+
        |   |   A&L   |
        |   +---------+
        |   |   B&L   |   +--+---------+
        |   +---------+   |  |   A&L   | LN channel
        |   |  A&B&L  |-->|  +---------+
        |   |or(L&CSV)|   |  |  A&B&L  | pseudo-Spilman factory
(tree)->|432+---------+   |  |or(L&CSV)|
        |   |   C&L   |   +--+---------+
        |   +---------+
        |   |   D&L   |
        |   +---------+
        |   |  C&D&L  |
        |   |or(L&CSV)|
        +---+---------+
```

Then, later, if the leaf node needs to be updated anyway (for example, if `C` and `D` buy enough inbound liquidity from the LSP that they need to get liquidity from the `A`, `B`, and `L` channel factory) the LSP can cut-through the state of the pseudo-Spilman factory:

```
       nSequence
        +---+---------+
        |   |   A&L   | LN channel
        |   +---------+
        |   |   A&L   | LN channel
        |   +---------+
        |   |   B&L   | LN channel
        |   +---------+
        |   |  A&B&L  | pseudo-Spilman factory
(tree)->|288|or(L&CSV)|
        |   +---------+
        |   |   C&L   | LN channel
        |   +---------+
        |   |   D&L   | LN channel
        |   +---------+
        |   |  C&D&L  | pseudo-Spilman factory
        |   |or(L&CSV)|
        +---+---------+
```

The advantage of using pseudo-Spilman at the leaves is that the pseudo-Spilman factory does not require an additional decrementing-`nSequence` layer.
This lets us reduce the additional delay added by the Decker-Wattenhofer layers.

In particular, HTLCs (and in the future, PTLCs) need to have, at minimum, a timelock that is the current blockheight plus the total Decker-Wattenhofer `nSequence` locktimes.
This has less impact for outgoing payments from clients; if the route already requires a timelock that is larger than that, then nothing needs to be changed.
However, for incoming payments *to* clients, the total Decker-Wattenhofer `nSequence` locktimes, plus margin, would have to be specified as the `final_cltv_delta`.

Keeping the total number of Decker-Wattenhofer layers is thus an important advantage, as a larger number of such layers impacts the public network, and reduces the ability of clients to receive from users that are distant (in terms of `cltv_delta`) from it.

The disadvantages are:

* The cost of unilateral exit increases with each state change, i.e. each time a client buys liquidity.
  - The LSP cannot afford to put an infinite amount of money into the factory anyway, so the cost is bounded anyway by how much money the LSP can put in the factory.
* Instead of effectively "splicing" in additional liquidity into an existing in-factory channel, this creates a novel channel with its own separate state.
  - In particular, the LSP and client now have to implement local multipath, to protect against edge cases where the client-LSP channels individually have insufficient capacity for a large incoming HTLC, but the total capacity can fit the large incoming HTLC.

Both disadvantages are less impactful compared to the reduction of `nSequence` delays.

### Inversion of Timeout Defaults and Old State Poisoning On Hosted Pseudo-Spilman

One can consider the pseudo-Spilman channel factory as a dynamically extendable part of the timeout tree.
Thus, when co-signing for the new transaction that spends from the channel factory, the clients and LSP also create an `nLockTime`d transaction that redistributes that output to the clients:

```
                          nLockTime
                           +--+-----+
                           |  |  A  |
       nSequence        +->|TO+-----+
        +---+---------+ |  |  |  B  |
        |   |   A&L   | |  +--+-----+
        |   +---------+ |
        |   |   B&L   | |  +--+---------+
        |   +---------+ |  |  |   A&L   | LN channel
        |   |  A&B&L  |-+->|  +---------+
        |   |or(L&CSV)|    |  |  A&B&L  | pseudo-Spilman factory
(tree)->|432+---------+    |  |or(L&CSV)|
        |   |   C&L   |    +--+---------+
        |   +---------+
        |   |   D&L   |
        |   +---------+
        |   |  C&D&L  |
        |   |or(L&CSV)|
        +---+---------+
```

Similarly, the new channel can also have the same treatment.

In addition, when the Decker-Wattenhofer state transaction is updated, the current old state tranaction will need to be poisoned, but in addition to the root pseudo-Spilman factory output being poisoned, the sub-factory and the new channel needs to be poisoned as well:

```
                          poisoning tx
                           +--+-----+
                           |  |  A  |
       nSequence        +->|  +-----+
        +---+---------+ |  |  |  B  |       poisoning tx
        |   |   A&L   | |  +--+-----+        +--+-----+
        |   +---------+ |                 +->|  |  A  |
        |   |   B&L   | |  +--+---------+ |  +--+-----+
        |   +---------+ |  |  |   A&L   |-+
        |   |  A&B&L  |-+->|  +---------+   poisoning tx
        |   |or(L&CSV)|    |  |  A&B&L  |-+  +--+-----+
(tree)->|432+---------+    |  |or(L&CSV)| |  |  |  A  |
        |   |   C&L   |    +--+---------+ +->|  +-----+
        |   +---------+                      |  |  B  |
        |   |   D&L   |                      +--+-----+
        |   +---------+
        |   |  C&D&L  |
        |   |or(L&CSV)|
        +---+---------+
```

## Laddering

I now add the fourth ingredient.

From the point of view of the LSP `L`, the mechanism created from the above three ingredients is an investment. The hope of the LSP `L` is that it can earn a return on this investment, from various means, including:

* Lightning Network routing fees.
* Selling of cheaper offchain liquidity.
* Fees for maintenance of the overall mechanism.

In addition, because of the timeout branches, the LSP cannot easily recover its funds until the end of the term.

Thus, a single timeout-tree-structured Decker-Wattenhofer mechanism with pseudo-Spilman leaves is very much like a term deposit, from the point of view of the LSP.

And as I pointed out earlier, savvy investors use laddering of multiple term deposit contracts in order to get a little more flexibility in how they allocate their funds.

The LSP itself can run multiple timeout-tree-structured Decker-Wattenhofer mechanisms with pseudo-Spilman leaves, with different sets of clients, and with overlapping terms, as in a ladder of term deposits in traditional finance. As the term of one mechanism ends, the LSP can start a new mechanism, inviting the clients in the terminating mechanism to transfer their funds to the new mechanism, including charges for the transfer. Then the LSP can recover the funds from the terminating mechanism via the timeout branch, onchain. The LSP gets earnings from routing fees, selling offchain liquidity, and the privelege of transferring to a new mechanism, and those earnings remain in Lightning-locked funds.

For example, the LSP can run 30 such mechanisms, all expiring on different days. When one of the mechanisms is about to expire, the LSP can invite the clients on that mechanism into a new mechanism. The LSP funds the new mechanism from the mechanism that expires today, creating a new 30-day mechanism. Then, the clients can move their funds from the dying mechanism to the new mechanism. Once all clients have exited the dying mechanism, on the completion of the term, the LSP can simply claim the entire UTXO, resulting in a single output being spent.

In the concrete example below, the LSP has a ladder of 9 timeout-tree-structured Decker-Wattenhofer factories with pseudo-Spilman leaves.  Each ladder has an "active period" of 7 days, and a "dying period" of 2 days.  During the dying period, clients can join one of the 2 factories built on the 2 days of the dying period, and transfer their funds to a new channel inside the new factory.  This allows clients a little leeway; if they miss the first day they can transfer, they have another chance on the succeeding day.  In actual deployments, I would mildly suggest an active period of 30 days and a dying period of 3 days; an LSP would then need to maintain 33 different factories at any time.  Ideally, there would be a single 1-input 1-output transaction per day, although if a client never comes online, the LSP would need to publish the path to its output, which increases the number of transactions needed onchain.

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

Now that I have hopefully given you a conceptual grasp of the laddered timeout-tree-structured Decker-Wattenhofer channel factory with pseudo-Spilman leaves construction, let us now turn to practical considerations for this mechanism.

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

The LSP can monitor the uptime of clients, and bin them according to what time of a 24-hour day they are most likely to be online. Then, when constructing trees for a new timeout-tree-structured Decker-Wattenhofer mechanism with pseudo-Spilman leaves, the LSP can group clients with similar "most active" times together in the same leaf nodes and in adjacent leaf nodes.

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
* Beyond a few layers away from the leaves, we could entirely remove state transactions (i.e. those with decrementing `nSequence`s). In effect, it would be a root timeout-sig-tree that backs multiple timeout-tree-structured Decker-Wattenhofer mechanisms with pseudo-Spilman leaves, which itself backs actual channels to clients.  
  - Again, if the LSP has to actually "go up a level" by more than one or two state tx layers from the leaves, then the group of clients that need to be awoken can be large enough that it is very unlikely all of them are online anyway, so you may as well reduce the delays of unilateral exit by not having more than a few layers of state transactions.  
  - Transactions that are not state transactions do not have a relative timelock, thus would not cause additional time delays in exit.  
* The LSP can group together clients with high uptime and put them into higher-arity nodes.  
  - Such clients would get better service (they would be grouped with other high-uptime clients which would be likely to be online as well when they need inbound liquidity) and cheaper and shorter unilateral exits (higher arity implies lower tree height implies less transactions on unilateral exit).  
  - Low-uptime and new clients would have to get more arity 2 and arity 1 nodes, increasing the cost of their unilateral exits, but this also gives better isolation against their sibling clients being offline.

# Necessary BOLT Extensions

I envision an extension to the BOLT specifications, where "base" node software would handle channel state updates, and a plugin (CLN, Eclair), client-provided trait structure (LDK), or side-daemon (LND) would handle channel factory protocol.

In effect, the "base" node software would provide an interface to its extension-mechanism that will expose events related to channel factories, and allow the plugin/trait/side-daemon to indicate how the channel works within the factory.

While this is necessary for laddered timeout-tree-structured Decker-Wattenhofer channel factories with pseudo-Spilman leaves, the same protocol extensions would be usable for ***any*** offchain channel-hosting mechanism.

The protocol extensions are:

* With the openv1 specification, add a `channel_factory_identifier` TLV to `open_channel`.
  - With 0-fee commitment transactions, *which* participant pretends to be the initiator and which one pretends to be the acceptor becomes irrelevant.
    The channel factory protocol can simply define which participants pretends to be the initiator, in some deterministic way.
    For example, in a truly peer-to-peer channel factory protocol, the participant with lexicographically earlier node ID can be the "initiator".
    For asymmetric LSP-client schemes, the LSP can be the initiator.
  - Even if the channel has to start with a different balance than "the initiator holds all the coins", the `push_msat` field can be used.
  - The `funding_created` message only specifies the transaction ID and output index that hosts the actual channel, without requiring multiple pretend `tx_add_input` and `tx_add_output` messages like in openv2 to synthesize the actual funding transaction outpoint.
  - The channel factory protocol would need to pre-sign any offchain transactions *before* the `open_channel` message is sent.
    Once the `funding_signed` message is exchanged, the participants can sign and publish the funding transaction of the actual channel factory.
  - The initiator would indicate `option_zeroconf`.
    - The funding transaction *for the channel* may never be published onchain, thus as far as the base node sofrtware is concerned, it is 0-conf.
    - Whether the *factory* itself is 0-conf should be handled with odd and/or >=32768 message ID messages negotiating the channel factory protocol.
  - The `channel_factory_identifier` would contain:
    - `protocol_id`: channel factory protocol identifier.
      i.e. this would differentiate between SuperScalar or some other factory protocol.
    - `factory_id`: the actual channel factory.
      i.e. for SuperScalar this could be a random identifier selected by the LSP.
    - `csv_delta`: how early to close the channel factory if an HTLC is about to time out.
      i.e. for SuperScalar, this would be the sum of all the Decker-Wattenhofer maximum `nSequence` from a root to a leaf node.
  - If the `channel_factory_identifier` TLV is specified (it would have to be even, as it cannot be safely ignored), the base node software would create an event on its extension mechanism, which must be handled by some extender, with either an "ok go accept it, expect the funding transaction to be `${FOO}`" or "not known, drop it".
    This event would have to include all the fields in the `channel_factory_identifier` TLV.
  - If the extension mechanism accepts the `channel_factory_identifer` TLV, the base node software would store the `channel_factory_identifier` fields in persistent storage, and also remember that the channel created is actually hosted inside *some* channel factory identified by the `channel_factory_identifier`.
* Additional messages that would indicate that the participants agree that the channel funding outpoint will be changed.
  - Whenever the channel factory changes state, it is very likely that the funding outpoint of the channels inside it will change.
  - We need a hand-over-hand strategy.
    That is:
    - The extension mechanism for the channel factory indicates to their base node software that a funding outpoint change will occur.
    - The extension mechanism can specify changes in the amounts owned by either side (which both sides would have to agree on).
      It can specify no change (for example, one of the *other* factory-hosted channels changed capacity, but not ours) or an increase or decrease in one side, or both sides.
    - The base node software of both endpoints agree on a provisional change in funding outpoint via some new messages.
      After this agreement, they will sign for the old and the new funding outpoints on each `commitment_signed`.
    - Then, the extension mechanism for the channel factory would arrange to sign the new channel factory state.
    - Finally, the extension mechanism indicates to the base node software that the change in channel funding is completed, after the new channel factory state is valid and the old state is invalid.
      (alternately, it could revert this, returning to the old funding outpoint, if the new channel factory state cannot be made valid; for example, the change started, but a required signer went offline before it could provide a signature).
    - The base node software of both endpoints then agree on the completion of the change (or the cancelling of the change).
      They exchange new messages regarding this event.
  - The above hand-over-hand strategy is ***exactly the same*** as in onchain splicing.
    We only need different sets of messages for them, as onchain splicing requires building transactions dynamically, whereas in this case, the channel factory protocol determines the new funding transaction outpoint.
    - The code to handle this part of channel factory extension would reuse the code for splicing.
    - Even if the channel factory protocol supports onchain splicing, it would require changes in the channel factory state anyway, so it might as well be funnelled to the same channel factory state.

-------------------------

ZmnSCPxj | 2024-11-25 20:18:07 UTC | #2

These are the SuperScalar UX notes that I wrote some time ago. I am now sharing it here.

***NOTE*** These should be considered as "very rough". ***Nobody*** has any experience with deploying a N>2 non-custodial offchain mechanism that integrates deeply with Lightning. It is entirely possible that my whole approach here is completely wrong-headed and delusional. This is the best I can do with what little knowledge I currently have as of October->November 2024. Hopefully, it can be improved as we go on to actually try implementing SuperScalar for larger groups of actual end-users.  Which is to say: if you follow these notes to the letter and end up with horrible UX that makes your mom hate you, that is your fault, not mine ^.^;;;;;;;;

--------------------------------------------------------

Subject: SuperScalar UX Notes

Preface: The Mobile Phone Environment
=====================================

These notes are primarily targeted for mobile phone uses.

I am not an expert in the mobile phone environment.
However, my initial study suggests the following:

* Messaging applications are common enough that significant
  parts of the mobile phone development environment are
  focused on timely delivery of messages.
  - This is not polled (or if it is polled, it is polled in
    aggregate amongst all messaging applications;
    not certain about exact implementation --- in Android,
    for example, timely delivery usually requires Google
    Services, and iOS seems to rely on similar services
    provided by Apple).
  - There are even cross-platform libraries/systems that
    make this easy to port across Android and iOS.
  - If a human user sends a message to another human user
    via a mobile-based messaging app, and the other user
    says "I did not receive it" the usual reaction is
    disbelief, since in practice message delivery is
    highly timely and reliable.
  - The example of Signal, which has end-to-end encryption,
    suggests that when a message is delivered, the app is
    given enough CPU power to at least perform decryption
    of the message, suggesting that at least some CPU power
    is available via this mechanism.
    - In SuperScalar, we require that clients also perform
      MuSig2 signing whenever the LSP signals a change in
      some subtree, which requires at least one and a half
      roundtrips (and probably two roundtrips to return the
      completed signature).
      I am uncertain at this point if it is possible to
      send small TCP bytestreams from the mobile app to
      the LSP during handling of an incoming message.
  - The example of Phoenix, which can receive funds even if
    the phone is locked and otherwise not actively used by
    the user, suggests that it is possible to compute an
    ECDSA signature and send it to the LSP at least using
    some mechanism to wake up the app (conjectured to be
    a messaging system similar to the above).
    Note that proper payment according to the BOLT spec
    requires a grand total of at least 3.5 round trips
    (2 roundtrips to irrevocably commit to the HTLC, then
    1.5 roundtrips to resolve the HTLC) and requires
    two ECDSA signing sessions each for client and LSP.

The above is from the point-of-view of the actual Bitcoin
Lightning wallet app.

From the point of view of the LSP, the LSP can simply expose
[LSPS5][].
Any Bitcoin Lightning wallet app which can afford to run a
webserver can register webhooks via LSPS5 that then cause
messages to be sent to the specific client mobile app, and
allow it to work with the SuperScalar provided by the LSP.

[LSPS5]: https://github.com/BitcoinAndLightningLayerSpecs/lsp/blob/52acff41b07f082d32512b0dd9a161d54ecaa2b9/LSPS5/README.md

The User Concepts
=================

A simple rule of thumb for UX design is: "give the user an
interface they are already used to".

Given that this is financial technology, it makes sense to
imitate existing banking apps and common banking
practices.

Thus:

* The app provides the equivalent of a "Bitcoin-denominated
  checking account".
* The account has an "account limit".
* The account has "deposit insurance", which can be
  either limited coverage (available for free) or unlimited coverage (requires one-time fee to be paid to miners and swap operators).

Expanding on the above:

* The app provides the equivalent of a "Bitcoin-denominated
  checking account".
  - Like any checking account, there is a monthly account
    maintenance fee.
    - Under the hood: This is the amortized cost of managing
      the UTXOs and liquidity needed in the SuperScalar
      and other protocols.
      The LSP can provide a fixed fee regime to the clients;
      the LSP can manage the risk of actual onchain fee
      variance by purchasing onchain feerate futures from
      miners, as described by Jack Mallers, and with a
      proposed mechanism [here][onchain-feerate-futures].
  - The fee may be waived if the account has a large enough
    amount consistently over the past month (as is typical
    for checking accounts provided by many banks).
    - Under the hood: Channel balances imply that the
      *more* funds the client owns in the channel, the
      *less* funds the LSP owns in the channel.
      Because of this, the LSP takes on less "cash drag"
      if the amount in the account is consistently high.
      (i.e. more of the LSP funds are in earning investments,
      such as published routing channels, and the earnings
      there can offset the cost of maintaining the
      unpublished channel with the client)
  - The advantage here, compared to a custodial scheme as in
    traditional banking, is that it is possible for a client
    to force account closure, even if the LSP has frozen the
    client account, and receive their funds onchain (minus
    fees taken by miners).
    This process takes some amount of time, possibly a month
    or two.
    - Under the hood: The client forcing account closure is
      simply triggering a unilateral exit.
      If we can use the SuperScalar addendum [Inversion of
      Timeout Default][], then the client app simply
      disconnects from the LSP and ignores requests from the
      LSP to wake up and connect, so that the LSP is forced
      to evict the unresponsive client by paying for the
      publication of the path to their unilateral exit.
* The account has an "account limit".
  - This is presented as the inverse of a credit card.
  - That is: the "account limit" of a ***credit card*** is
    how much you can ***spend*** until you have to receive
    funds into it (or increase your account limit).
  - In contrast, as this is a debit instrument, the
    "account limit" of the ***Bitcoin-denominated checking
    account*** is how much you can ***receive*** until you
    have to spend it (or increase your account limit).
    - Under the hood: the account limit is really just the
      total channel capacity of all channels between LSP
      and client, minus the channel reserve requirement
      imposed by the client on the LSP.
      The LSP does not impose channel reserves on the
      client (i.e. 0% reserve on the client side, but non-0
      on the LSP side) because the main purpose of the
      reserve is to prevent theft attempts from being
      free; the LSP is assumed to have very high uptime,
      and to have watchtowers with uptime uncorrelated with
      their actual LN node, so that theft attempts being
      free do not matter in practice (for the LSP; it matters for the client, hence the non-0 reserve it imposes on the LSP).
  - The UI is expected to show the current account balance
    and the current account limit in close proximity to each
    other, and to provide a simple way to show the
    difference between the current balance and the current
    limit.
    - Under the hood: the difference between the account
      balance and the account limit is the current inbound
      liquidity from the LSP to the client.
  - The UI allows the user to "upgrade account limit"
    for a cost (which may be waived by the LSP if the user
    has had a consistently high account balance over the
    past month).
    - Under the hood: This is really just purchasing
      inbound liquidity from the LSP.
      The cost of purchasing the inbound liquidity can be
      made fixed via the use of onchain feerate futures as
      described earlier, and if the client has a
      consistently high account balance in their account,
      as noted as well more of the LSP funds can be put to
      earning investments such as published channels,
      which also offset the cost of upgrading the account
      limit.
    - Under the hood: naively, we might think that the
      LSP can control exactly the liquidity that is
      devoted to each user, and thus to hide the
      details of liquidity management.
      Unfortunately, this is not the case, as the LSP
      needs to provide, upfront, some funds of its own,
      before it can cater to the needs of large users.
      Suppose some individual user has some large
      amount of funds inside a SuperScalar.
      When transitioning from one SuperScalar to the
      next, the LSP has to put up funds that is at
      least the account balance of the individual
      large user, ***plus*** some additional margin
      for future receives of that large user.
      While this lockup is transient, there is always
      some risk that the client will force a unilateral
      exit *before* moving from the old SuperScalar to
      the next, thus forcing the LSP to lock up those
      funds to that client for a month (as changing
      in-SuperScalar states --- i.e. moving liquidity
      --- requires the client cooperation, and the
      client can refuse to cooperate, the LSP
      practically risks having the large fund be
      locked up in a form that cannot be used without
      a lot of blockspace usage and time for
      unilateral exit).
      The risk that the LSP takes on here, is the
      charge it would have to impose on the client for
      the larger amount the client can keep in the
      SuperScalar.
      This holds true even outside of SuperScalar
      (though blockspace use in unilateral exit is
      better outside of SuperScalar), and is the
      entire reason why we have *any* kind of
      pay-for-inbound-liquidity scheme in Lightning.
      Thus, it is infeasible to ***not*** expose
      the account limit concept to end-users; a
      higher account balance is a larger risk on
      the LSP and must be paid for, thus end-users
      need to be aware that increasing their account
      limit will be costly, and they should be
      informed of this *before* they put any amount
      in the account.
    - Under the hood: Under the [Inversion of Timeout Default][] addendum, part of the security of
      clients lies in the fact that the LSP has to
      lock in funds equal to *or greater than* the
      account limit of the client.
      Provided the client does not ever upgrade its
      account limit (expected for most clients)
      then it does **not** need an external UTXO to
      pay for onchain fees on unilateral close
      (i.e. it does not require exogenous fees, as
      the LSP is the one at risk of loss if it
      does not unilaterally exit before timeout).
      However, it does imply that the LSP cannot
      have "fractional reserve" of the account
      limit.
      Suppose we have clients `A` and `B`, promised
      account limits of 10 units each.
      Suppose the LSP tries a fraction reserve of
      account limit, giving each one 7 units of
      capacity, on the assumption that the total
      capacity needed by both clients will take
      no more than 14 in total, in practice.
      Suppose `A` ends up having a balance of 2
      units, but `B` maxes out its promised
      account limit of 10 units.
      The total is 12 units, which appears fine
      (as the total capacity allocated to `A` and
      `B` is 14 units), but the Inversion
      protection that `B` has only covers up to 7
      units in case of LSP misbehavior.
      The Inversion protection cannot be increased
      without adding further Decker-Wattenhofer
      layers in the Inversion protection transaction,
      which weakens the protection --- the protection
      itself would require further protection of
      its own mutable state.
      The only way for `B` to protect itself fully
      would be to move some funds onchain to get
      an onchain UTXO it can use to pay for P2A
      exogenous fees.
      Thus, if the LSP promises an account limit of
      10 units, then the LSP definitely needs to
      provide 10 units, plus a little extra, to
      each of its channels with `A` and `B`.
* The account has deposit insurance, which can be
  either limited coverage (for free, and the default) or unlimited coverage.
  - In case the LSP refuses to give proper service to
    the client, the client can force account closure
    regardless of what the LSP does.
    - For example, if the LSP shuts down without
      properly allowing clients to withdraw funds,
      or if the LSP is bought out by an entity that
      decides to freeze the accounts of some clients.
    - The client can always force account closure,
      though the UI attempts to funnel them through
      cooperative account closure before letting
      them force account closure.
  - The forced account closure will be covered by the
    deposit insurance; i.e. the deposit insurance will
    put the account balance onchain, minus mining
    fees, if the client forces account closure.
    The deposit insurance is no-fault; the client
    does not need to justify why it forced the
    account closure.
  - At the start of the service month, the coverage
    of the limited coverage scheme is equal to the
    client account limit at the time, plus some extra
    amount as a buffer.
  - If the client has limited coverage (the default),
    then the maximum amount the client can safely get
    from the client-initiated forced account closure
    is limited by the deposit insurance, minus onchain
    mining fees.
    Forced account closure can take a month or more
    to resolve.
  - The limit of the limited coverage is higher than
    the account limit at the start of the service
    month, so for most users, limited deposit
    insurance coverage is sufficient.
    However, if the client upgrades their account
    limit and receives more funds into their
    account, the client may have a total account
    balance that is greater than the existing
    coverage.
  - The client can get unlimited coverage by moving
    some funds into a deposit coverage reserve,
    plus paying onchain mining fees and swap fees.
    The client can get back those funds into their
    account (after paying onchain mining fees and
    swap fees, and only if they have available
    space under the account limit), but loses
    unlimited coverage and returns to limited
    coverage.
  - If the client has unlimited coverage, then the
    client can always get the full amount of the
    account, minus onchain mining fees.
    Onchain mining fees for forced account closure
    with unlimited coverage will be larger than with
    limited coverage, but forced account closure can
    take a few days or weeks to resolve instead of
    the 2 months that the limited coverage resolves.
  - Under the hood: the limited coverage comes from
    the "Inversion of Timeout Defaults" addendum.
    Under "Inversion of Timeout Defaults" scheme,
    the LSP potentially loses an amount equal to
    the account limit, plus some additional amount,
    if it lets the timeout pass without paying for
    a unilateral exit or supporting an assisted exit.
    This total amount (which is the account limit
    plus a share of liquidity-in-stock) is always
    greater than the account limit at the start of
    a new SuperScalar, and ***is*** the deposit
    insurance coverage.
    However, if the current balance becomes greater
    than the coverage (because the client bought
    additional inbound liquidity and received
    funds), the LSP has an incentive to not
    pay for unilateral exit of the client, as its
    liability to that client in the timeout default
    case is smaller than the amount owned by the
    client in the account.
    However however, the client can move some funds
    onchain (the deposit coverage reserve), which
    the client can use to pay for unilateral exit.
    In that case, the client can always enforce its
    account balance by being able to pay for its
    own unilateral exit, regardless of the timeout
    default amount they would get.
    The onchain funds can be returned to the
    SuperScalar ownership by a simple swap, but
    returns the client to having limited coverage.
  - The deposit insurance is ***NOT*** provided by
    some third-party entity.
    It is enforced by the client app directly, using
    the funds shared between LSP and client.
  - The existence of an unlimited deposit coverage
    is a security detail that impacts larger users.
    In principle, it can be hidden from the user,
    but as the deposit coverage reserve is an onchain
    fund, it ***must*** be managed with onchain fees
    in mind, which vary and are always costly, and
    cannot be easily moved to offchain.
    In particular, ***it is protection against the
    LSP misbehaving***, and thus should really not be
    handled by the LSP (and thus CANNOT be hidden by
    the LSP in its monthly maintenance fee --- we
    cannot trust the LSP to implement it correctly!
    By the same token, we cannot use swap-in-potentiam
    to quickly move the reserve fund from onchain to
    offchain --- swap-in-potentiam assumes the LSP
    will willingly cooperate in order to immediately
    be able to spend using the swap-in-potentiam UTXO,
    and in a scenario where the client has lost trust
    in the LSP, it is implausible to use that fund to
    pay for unilateral closure to protect against
    LSP misbehavior).
    It has to be implemented completely by the client
    software, preferably in open-source code with
    deterministic compilation.
    FWIW, this only affects larger users; smaller
    users can live with the default limited deposit
    insurance coverage; even if the smaller user
    upgrades their account limit, if the actual
    balance does not go above the coverage (which
    is above the limit at the start of the service
    month) then the user does not need to switch
    to unlimited insurance coverage, as on the
    *next* service month, the limit of the limited
    insurance coverage will rise above the new
    account limit.

[onchain-feerate-futures]: https://delvingbitcoin.org/t/an-onchain-implementation-of-mining-feerate-futures/547
[Inversion of Timeout Default]: https://delvingbitcoin.org/t/superscalar-laddered-timeout-tree-structured-decker-wattenhofer-factories/1143/26

Onboarding Flow
===============

When initially installed, the app simply presents an
interface to open an account to the LSP.
The app may present an interface to select one among
multiple LSPs that provide the needed protocols.

The app provides the options below:

> - Standard account open
>   - The account will be opened within 48 hours.
> - Express account open
>   - The account will open immediately, but with a
>     higher fee, and there is a minimum deposit
>     amount.
> - Onchain account open
>   - The account will open as soon as an onchain
>     address deposit gets N confirmations.

For all types of account opening, the app presents
the terms and conditions below:

> - Monthly maintenance fee: NN satoshis
>   - The fee may be waived if you maintain an
>     account balance that is at least NN% of the
>     account limit for the entire month.
> - Starting account limit: NNNN satoshis
> - Account limit upgrade fee: pay NN satoshis
>   to increase your account limit by NNNN
>   satoshis, at any time.
>   - The fee may be waived if you maintain an
>     account balance that is at least NN% of the
>     account limit for the entire month.
> - Pay next maintenance fee on YYYY-MM-DD HH:MM
>   (TZ).
>   - Your device must have Internet connectivity
>     and this app must have notification
>     permission at this time.
>     The maintenance fee will be deducted from
>     your account balance.
>   - If you miss the above payment, you can pay
>     on YYYY-MM-DD HH:MM (TZ), YYYY-MM-DD HH:MM
>     (TZ)....
>   - If you miss all the above payment windows,
>     your account will be closed and funds will
>     be moved onchain, as protected by your
>     deposit insurance.

- Under the hood: paying the monthly maintenance
  fee is actually moving from an old, dying
  SuperScalar in the ladder, to the latest
  SuperScalar.
  As SuperScalar constructions are made once a day,
  those are the times at which you can move from
  the old SuperScalar to the newest SuperScalar.
  Thus, the device must be online and capable of
  receiving messages from the LSP, so that the
  LSP can invite the client to join the newest
  SuperScalar, and charge the maintenance fee at
  that point.
  Joining the newest SuperScalar requires multiple
  signing sessions with the client, and thus
  Internet connectivity and enough CPU to sign
  everything.
  If the client misses the monthly payment, the
  LSP gives them additional opportunities to join
  the next SuperScalar on succeeding days, up to
  some limit.
- Under the hood: if the user completely misses
  the given times, the LSP has to evict them via
  the unilateral close mechanism, which, if we
  use the [Inversion of Timeout Default][]
  addendum, is paid for by the LSP.

Standard Account Open
---------------------

This is the simplest case: the client requests to
be scheduled to open an account on the next
available slot the LSP has.

The terms and conditions presented are:

> - Your account will open on YYYY-MM-DD HH:MM
>   (TZ).
>   - Your device must have Internet connectivity
>     and this app must have notification
>     permission at this time.
> - If you miss the above opening time, you can
>   open at YYYY-MM-DD HH:MM (TZ).
> - You cannot receive or send funds until the
>   account opening time that your device has
>   been online for.

- Under the hood: new SuperScalar mechanisms are
  constructed daily, and the LSP simply schedules
  the new client to join the next SuperScalar
  mechanism, and wakes them up at the appropriate
  time to complete the signing session.
  If the client does not come online on the first
  scheduled day, the LSP allows the client to join
  on the next scheduled day.

The LSP may also require initial deposit, depending
on the risks the LSP is willing to take on in its
environment.
For instance:

- The LSP may require a minimum initial deposit,
  without deducting any amount from it, to prevent
  trivial DoS attacks where a botnet just spams the
  LSP.
  - The initial deposit may be done in multiple
    receives, with sending disabled until the
    minimum initial deposit is reached.
- The LSP may deduct an initial fee from the
  initial deposit, again to deter trivial DoS attacks
  from funded attackers.
- The LSP may waive the above requirements if it can
  ban an identifiable individual that attempts to DoS
  the LSP.

How the above are communicated between the app and the
LSP are left as an exercise to the reader.

Express Account Open
--------------------

In this flow, the LSP charges a higher fee for the
account opening, but the account is opened
immediately.

The terms and conditions for this opening are:

> - You must pay a Lightning invoice with at least
>   NNN satoshis, as initial deposit.
> - A fee will be deducted from the initial deposit,
>   of NNN satoshis, or N.N% of the amount deposited,
>   whichever is larger.
> - Once paid, the account is opened for sending and
>   receiving.
>   Upgrading the account limit will not be immediately
>   available.
> - On YYYY-MM-DD HH:MM (TZ) the account limit can be
>   upgraded at your control.
>   - Your device must have Internet connectivity
>     and this app must have notification
>     permission at this time.
>   - If you miss the above time to enable upgrades
>     to your account limit, you can enable account
>     limit upgrades on YYYY-MM-DD HH:MM (TZ).

- Under the hood: the client uses a JIT channel flow,
  such as [LSPS2][].
  The minimum size is `min_payment_size_msat` of
  LSPS2, and the fee scheme is `min_fee_msat` and
  `proportional` of LSPS2.
  Once the LSPS2 invoice is paid, a non-SuperScalar
  onchain channel is opened.
  As the cost of adding new inbound liquidity in an
  onchain channel is higher, the LSP also invites
  the client to join the next daily SuperScalar
  construction, and the app presents this as
  being unable to increase the account limit until
  after the app moves the user funds from the
  onchain JIT channel to a SuperScalar.
  Once the app has moved user funds into SuperScalar,
  it enables upgrading of the account limit, i.e.
  purchasing inbound liquidity within SuperScalar.

[LSPS2]: https://github.com/BitcoinAndLightningLayerSpecs/lsp/blob/52acff41b07f082d32512b0dd9a161d54ecaa2b9/LSPS2/README.md

Onchain Account Open
--------------------

In this flow, the app provides an onchain address
to be funded by the user.
Once the onchain address has a UTXO of at least a
minimum size, and the transaction with that output
has confirmed with sufficient depth, the channel is
opened.

As a bonus, under this flow, if the LSP refuses
service and blocks account opening, the deposited
funds can be refunded, minus onchain mining fees.

The terms and conditions for this flow are:

> - You must make at least one deposit to a given
>   `bc1p` (P2TR, "Taproot", "bech32m") onchain
>   address, of at least NNNNNN satoshis.
> - The account is opened for sending and receiving
>   as soon as the onchain send to the given address
>   has N confirmations.
> - A mining fee of NNN satoshis will be deducted
>   from the initial deposit.
> - The initial account limit is equal to the
>   initial amount deposited, minus the mining fee.
> - Upgrading the account limit will not be immediately
>   available.
> - On YYYY-MM-DD HH:MM (TZ) the account limit can be
>   upgraded at your control.
>   - Your device must have Internet connectivity
>     and this app must have notification
>     permission at this time.
>   - If you miss the above time to enable upgrades
>     to your account limit, you can enable account
>     limit upgrades on YYYY-MM-DD HH:MM (TZ).
> - If the LSP refuses to complete account opening,
>   the initial deposit can be refunded to an
>   onchain address you control, minus mining
>   fees, NNN blocks after the deposit confirms.

- Under the hood: This flow uses [swap-in-potentiam][].
  Once the swap-in-potentiam address is funded, with
  the minimum confirmation depth imposed by the LSP,
  the app can then open a channel using those funds.
  Providing a plain address maximizes the number of
  wallets that can deposit to the address, while
  allowing "instant" channel opening (i.e. no
  additional confirmations are needed beyond
  confirming the deposit to the plain
  swap-in-potentiam address).
  Once the swap-in-potentiam address has been
  funded, the client can present it to the LSP and
  construct an onchain channel with no additional
  trust.
  The ability to refund the onchain amount, in case
  the LSP refuses to accept the channel, is built
  into the swap-in-potentiam protocol.
  As the cost of adding new inbound liquidity in an
  onchain channel is higher, the LSP also invites
  the client to join the next daily SuperScalar
  construction, and the app presents this as
  being unable to increase the account limit until
  after the app moves the user funds from the
  onchain swap-in-potentiam channel to a SuperScalar.
  Once the app has moved user funds into SuperScalar,
  it enables upgrading of the account limit, i.e.
  purchasing inbound liquidity within SuperScalar.

[swap-in-potentiam]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2023-January/003810.html

Deposit Insurance Coverage Management
=====================================

By default, the user gets a limited deposit insurance
coverage, that is equal to the account limit at the
start of a service month, plus some extra margin.
For most users, this is sufficient, as they would not
be able to receive more than their current account
limit anyway.

However, if the user then upgrades their account limit,
and receives funds, then the limited deposit insurance
coverage may become lower than their actual account
balance.

When the user account balance reaches 90% of the
limited deposit insurance coverage, the app can remind
the user to switch to unlimited deposit insurance
coverage.

To enable unlimited deposit insurance coverage, the
app has to move some funds to a locked reserve fund
that is separate from the account.
Mining and swap fees are also charged to move to this
locked reserve fund.

The user may withdraw the locked reserve fund at any
time, which returns them to limited deposit insurance
coverage.

> - If the LSP refuses to provide service, or otherwise
>   attempts to freeze or impede sending, receiving,
>   upgrading account limit, or closing your account,
>   you can, at any time, force the closure of your
>   account despite LSP interference and receive your
>   funds in an onchain wallet.
> - Fees involved in onchain operations to enforce
>   the closure of your account are paid by you and
>   deducted from your recovered account.
> - Onchain operations to enforce closure require a
>   minimum amount.
>   If your account balance is below the minimum
>   amount after fees, force closure will not recover
>   any of your funds, but neither will the LSP be
>   able to retain your funds.
> - If possible, make a good-faith effort to contact
>   the LSP at `<whatever@wherever.com>` with your
>   concerns before initiating force closure.
> - Under limited deposit insurance coverage:
>   - Force closure can take 1-2 months before you
>     can move your funds to an onchain wallet.
>   - Enforcement fees paid to miners are lower.
>   - The coverage is limited; the limit of the
>     coverage is set at the start of a service
>     month, and set to be higher than your account
>     limit at that time, but if you upgrade your
>     account limit to be higher and then receive
>     funds, your account balance can end up higher
>     than the current coverage.
>     In that case, you may recover less than your
>     account balance, and only up to the limited
>     coverage.
> - Under unlimited deposit insurance coverage:
>   - You need to reserve a fund of NNNNN satoshis,
>     separate from (and funded by) your account
>     balance.
>     Creating and withdrawing this reserve fund
>     requires paying mining and swap fees.
>   - Force closure is expedited and can take 1-2
>     weeks instead of months.
>   - Enforcement fees paid to miners are higher
>     (for the expedited closure and to ensure
>     unlimited coverage).
>   - Your entire account balance is protected,
>     and your reserve fund, minus enforcement
>     fees, will be included in the onchain
>     withdrawal.

- Under the hood: the [Inversion of Timeout Default][]
  addendum provides the mechanism for the deposit
  insurance.
  The addendum has the deposit insurance coverage ---
  actually the branch where the LSP is punished if it
  does not provide an exit (unilateral or assisted)
  to clients before the timeout of the tree --- of
  fixed amount at the start of a SuperScalar
  mechanism.
  The assumption here is that the client may not have
  an external UTXO with which to pay exogenous fees
  for unilateral exit, and must rely on the punishment
  branch to force the LSP to give it a unilateral
  exit.
  By reserving some funds into an onchain address
  controlled solely by the client, the client can
  enforce the unilateral exit directly (due to having
  an external UTXO to pay exogenous fees with), at
  the cost of having to pay the unilateral exit for
  itself instead of the LSP paying for it.

Send And Receive Flow
=====================

While *most* Lightning payments resolve quickly,
there is a small chance that they take a long time
before they can resolve.
Thus, like typical Lightning wallets, a
SuperScalar-based app has to show outgoing payments
as "pending" with the corresponding funds locked
until the payment completes.

Upgrade Account Limit Flow
==========================

The app may provide an option that authorizes it to
automatically upgrade account limit when the account
limit is being approached by the account balance.

When an account limit upgrade is requested
by the user, LSP quotes some number of satoshis
of cost, in order to increase the account limit by
some number of satoshis.
The user may request an account limit upgrade at any
time.
The account limit upgrade may take several minutes
to an hour to finalize; the user can send and
receive normally even while the account limit
upgrade is ongoing.

- Under the hood: As noted, this is actually a
  purchase of inbound liquidity from the LSP.
- Under the hood: The optimistic case is that the LSP
  is able to wake up other client apps in order to
  authorize a change in state of a sub-tree of the
  SuperScalar construction.
  However, if interaction with one of the other
  clients times out, the LSP can instead fund an
  onchain channel with the client, with a timeout
  equal to the timeout of the current SuperScalar the
  client is on.
  (the timeout on the onchain channel can be
  implemented as an `nLockTime`d transaction that
  can spend the funding transaction output)
  The LSP takes on this risk (i.e. the price it
  quoted remains the same, thus, the LSP has to
  factor in the risk that other client apps do not
  come online fast enough).
  If the LSP has already opened an onchain channel,
  and the other clients do not respond fast enough to
  some request to increase in-SuperScalar capacity,
  then the LSP can splice into the existing channel.
  Basically, onchain is used as a fallback in case
  of coordination problems (i.e. some clients are
  offline).
- Under the hood: While an account upgrade is ongoing,
  and the in-SuperScalar channel capacity increase
  is being negotiated by the LSP with other clients,
  the client and LSP have to maintain two sets of
  state in the channel: one for the
  pre-capacity-increase channel, and one for the
  post-capacity-increase channel.
  Other clients that are needed to sign off on the
  capacity change must also maintain two sets of
  state with their in-SuperScalar channel.
  This code would be very similar to handling of
  multiple states in onchain splicing, and is likely
  to share code with splicing handling.
  This allows all involved clients to continue
  sending and receiving as normal while the account
  limit upgrade is ongoing.

Monthly Maintenance Fee Flow
============================

The monthly maintenance fee is paid by the user
once a month.
The app should default to being authorized to
automatically pay the monthly maintenance, as per
the schedule set by the LSP.
This implies that the app controls the keys involved
in the Lightning channel.

The LSP sets particular times at which the monthly
maintenance fee may be paid.
The device must have Internet connectivity at those
times, and the app must have notification permission
at those times.
The LSP will automatically awaken the app at those
times, at which point the app authorizes the payment
of the monthly maintenance fee.

If the user misses the given time, the LSP also
gives the user a few more given times, on succeeding
days, at which they can pay the monthly maintenance
fee.

The app can display the time at which it will be
asked to pay the monthly maintenance fee, as well
as the grace period the LSP allows.

The process of monthly maintenance is:

* The client pays the monthly maintenance fee to the
  LSP, who queues them up for completion of monthly
  maintenance.
* Once the client has paid the fee, the monthly
  maintenance proceeds in two phases:
  * Common maintenance phase: For several seconds or
    minutes, up to an hour or two, the LSP repeatedly
    attempts to create a set of online clients that
    are able to cooperatively perform maintenance
    tasks simultaneously while online.
    During this time, all the online clients can
    continue to send and receive funds normally,
    but cannot negotiate an account limit upgrade
    while this is ongoing.
  * Per-client maintenance phase: The client app
    arranges to complete the monthly maintenance,
    while still able to send and receive funds
    normally, as well as negotiate an account
    limit upgrade.

If a client has paid the monthly maintenance fee,
but is not part of the common maintenance phase
(e.g. its Internet connectivity is interrupted between
paying the fee and completing the common maintenance
phase), the LSP will invite the client to complete the
common maintenance phase on the succeeding day(s)
until the client can complete the common maintenance
phase, or until some timeout, at which point the LSP
will force closure of the client account and refund
the monthly maintenance fee.
The client may also force closure of the client
account if the LSP refuses to allow the client to
complete the common maintenance phase.

The per-client maintenance phase can be done as long
as te client is online and the LSP is cooperating.
If the LSP or client refuse to complete the per-client
maintenance phase, the other side can force closure
of the account to recover funds.

- Common maintenance:
  - Under the hood: This is the part where the clients
    and LSP actually construct a new SuperScalar
    construction, with all funds initially owned by
    the LSP, but with in-SuperScalar channels having
    capacity equal to the current account limits of
    the clients.
    This requires multiple MuSig2 signing sessions;
    if some client does not complete the MuSig2
    signing sessions quickly enough, the LSP will
    restart the opening without that client, until
    it can get a completed SuperScalar opening
    with the remaining clients all able to complete
    necessary signatures.
- Per-client maintenance:
  - Under the hood: This is just a swap from the
    current SuperScalar to the newly-constructed
    SuperScalar tree.
    If both client and LSP are online and
    cooperative, this should be fast and resolve in
    seconds.
    - Under the hood: After completion of the
      common maintenance phase, the LSP will forward
      payments to the client on the new SuperScalar
      instead of the old SuperScalar.
    - Under the hood: Ideally, the client simply
      sends the entire account balance from the
      old SuperScalar to the new SuperScalar.
      However, if the client has pending sends,
      the client has to maintain the per-client
      maintenance state until the LSP has either
      fulfilled or failed the HTLC (i.e. "resolve"
      the HTLC, meaning it either fulfilled or
      failed).
      If the LSP does not resolve the HTLC of the
      pending send before the old SuperScalar
      dies, the LSP is forced to unilaterally close
      the old SuperScalar under the [Inversion of
      Timeout Defaults][] addendum, and the client
      can force close of the new SuperScalar as
      well (by refusing to cooperate with the LSP
      until the LSP is forced to time out, as per
      Inversion).
      If the LSP fulfills all pending HTLCs, then
      the old SuperScalar will not have any client
      funds and the client can hand over the private
      key to the old SuperScalar construction (to
      allow the LSP to recover funds without
      publishing the complete unilateral exit).
      If the LSP fails any pending HTLCs, then the
      amount is returned to the old SuperScalar,
      but the client app can immediately move the
      funds to the new SuperScalar.
    - Under the hood: The client temporarily has
      double its account limit while the old and
      new SuperScalars are still active, but since
      the old SuperScalar is dying soon, the client
      needs to strictly enforce its own account
      limit while doing per-client maintenance.
      This occurs if the client has any pending
      HTLCs being sent; the HTLC amount could still
      be refunded if the HTLC is failed.
      As a concrete example: suppose the account
      limit is 10 units, the client has 9 units,
      and has a pending send of 1 unit.
      On transferring to the new SuperScalar, the
      client moves the 9 units to the new
      SuperScalar, leaving inbound capacity on the
      new SuperScalar at 1 unit.
      If the client receives an HTLC on the new
      SuperScalar of 1 unit, it ***MUST*** reject
      it until the pending send of 1 unit (on the
      old SuperScalar) has resolved; otherwise,
      if it fulfills the incoming HTLC *and*
      the pending send ultimately fails and is
      refunded, the client has to fit 11 units of
      balance in 10 units of limit.

Account Closure
===============

Account closure can be assisted by the LSP, or be
forced by the client.

Once the client starts either account closure flow,
it can no longer send, receive, or upgrade account
limit.

The client can only initiate an assisted account
closure if there are no pending sends or receives.
However, the client (and LSP) can initiate a forced
closure at any time, even if there are pending sends
or receives.
Pending sends and receives during a forced closure
can get resolved offchain, but at the worst case
will be resolved on the blockchain.

As account closure involves onchain activity, there
are practical minimum amounts that can be used
post-closure.
If the account balance is below the practical onchain
minimum amount, then an assisted account closure is
not feasible (the client will end up paying their
entire balance as onchain fees).

After completion of account closure, the client is
in possession of onchain funds.
The client can then sweep those funds onto some
onchain address or silent pay address, or can
use those funds to reopen the account using the
"onchain account open" flow.

Assisted Account Closure
------------------------

In this flow, the client requests the LSP to close
the account, and the LSP must accept this request.
In case the LSP declines this request or otherwise
does not respond to this request, the client app
provides an option to force account closure instead.

In the case that the client has absolutely 0 account
balance, the account can be trivially closed.

- Under the hood: if the client has 0 account balance,
  it can trivially hand over the private key used in
  the current SuperScalar.

The closure requires two phases, both requiring
onchain confirmation.
The client sets the number of confirmations that it
would consider acceptable, though no more than say
12 confirmations.

- LSP cooperation phase.
- Client claim phase.

The details are:

- LSP cooperation phase.
  - Before starting the account closure, the client
    app provides estimates on how much cost the
    user will pay based on current onchain feerates,
    with the warning that onchain feerates can change
    drastically and that the amount paid may become
    larger later.
  - Under the hood: the client offers HTLC(s) of
    all funds in its account (i.e. inside
    SuperScalar) to the LSP.
    The LSP then takes some onchain fund of
    equivalent value into an onchain HTLC.
  - Under the hood: the client indicates its
    desired onchain feerate, and is responsible
    for paying some of the costs involved.
    The LSP agrees if it considers the onchain
    feerate reasonable; if it cannot get agreement,
    the client can always fall back to forced
    account closure.
    The client pre-pays for the onchain fees,
    deducting from its account balance as soon
    as the LSP indicates acceptance.
    The client is responsible for paying:
    - Transaction cost of the LSP cooperation
      phase.
      - 1 transaction input (LSP funds).
      - 1 transaction output (LSP change).
      - 1 transaction output (onchain HTLC).
    - Transaction cost of the LSP timeout
      recovery.
      - 1 transaction input (onchain HTLC, timeout
        branch).
      - 1 transaction output (LSP funds).
  - Under the hood: this phase completes once the
    onchain HTLC address has confirmed to sufficient
    depth that the client considers safe.
- Client claim phase.
  - Under the hood: the client claims the onchain
    HTLC, revealing the preimage.
    This is just a 1-input 1-output transaction.
  - Under the hood: the client SHOULD use a "burn"
    strategy with an RBF-able transaction: it can
    start at what it considers a reasonable fee,
    but as the onchain HTLC timeout approaches,
    the client increases the onchain transaction
    fee, until at the block just before the the
    timeout it offers the entire amount as onchain
    fee to miners.
  - Under the hood: this phase completes once the
    1-input-1-output claim transaction has confirmed
    to sufficient depth that the client considers
    safe.
    On completion of this phase, the client hands
    over the private key to the SuperScalar it just
    exited from.

Forced Account Closure
----------------------

The client or LSP can initiate an account closure at
any time.

There are two forms of forced account closure:

- Forced account closure by timeout.
- Forced account closure by publication.

Which kind of force account closure the client
*actually* uses depends on its current mode of
deposit insurance coverage:

- If the client currently has limited deposit
  insurance (the default), the client forces account
  closure by timeout.
  - This takes longer, but the cost on the client is
    lower.
- Otherwise, the client has unlimited deposit
  insurance, and the client forces account closure by
  publication.
  - The cost on the client is higher, but because it
    does not wait for a timeout, the client can get
    its funds earlier.
  - Under the hood: in this mode, the client has some
    funds in an onchain address, which can be used
    to pay for exogenous fees via P2A outputs on the
    timeout tree nodes.

As account closure requires moving funds onchain,
there is a practical minimum account balance before
users can actually recover funds onchain.
However, even if the user account balance is below
this practical minimum, the LSP cannot get that
balance in the forced account closure flow, and the
balance will instead go to miners.

For details:

- Under the hood: Ultimately both kinds of forced
  account closure boil down to forced closure by
  publication.
  Under [Inversion of Timeout Default][] addendum,
  forced account closure by timeout simply has the
  client wait until the impending death of the
  SuperScalar it is currently in, and then the LSP
  is forced to publish the unilateral exit (the
  "forced closure by publication") onchain, as
  otherwise the LSP can lose funds that were locked
  in the SuperScalar.
- Under the hood: Multiple timeouts are involved in
  any forced account closure; these timeouts are
  enforced to allow participants to ensure that the
  latest state is what is resolved onchain.
  Forced account closure by timeout can take 1-2
  months, while forced account closure by client
  publication will take a few weeks to a month.
  Note that the forced account closure by timeout
  is really waiting for the LSP to perform forced
  account closure by publication, thus the additional
  worst-case month is due to having to wait up to a
  month for the current SuperScalar to time out.
  If the client has unlimited deposit insurance
  coverage (i.e. has an exogenous UTXO they can use
  to pay for exogenous fees) then the client can
  use forced account closure by publication itself,
  which shortens the time, but also increases the
  cost of forced account closure.

Post-closure
------------

Once the client has completed a closure (whether
assisted or forced), the client app will have some
onchain funds that can be swept.

-------------------------

