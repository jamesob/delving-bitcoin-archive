# Erlay: Overview and current approach

sr-gi | 2025-01-31 20:23:48 UTC | #1

Earlier last year I started working on a second attempt to implement Erlay into Bitcoin Core, a process that started from convincing myself that the theoretical improvements were achievable in practice, followed by a research-oriented approach where multiple protocol choices need to be simulated and that will hopefully conclude with a well though implementation that satisfies the original goals.

After several months of working alongside other Bitcoin Core developers (teaming up to create an Erlay working group), endless conversations, and hours of simulations (after implementing a [discrete time simulator for the matter](https://delvingbitcoin.org/t/hyperion-a-discrete-time-network-event-simulator-for-bitcoin-core)). I'm happy to start reporting the current state of the project, especially for those of you who may be curious by not directly involved in it.

This initial post will cover an overview of Erlay, alongside the current implementation approach, thought process, and some open questions.

Future posts will contain some of the experiments we have been performing to answer those questions. Bear in mind Erlay is still in active development so some of the questions may still have no satisfactory answer.

Any errors found within this write-up, or any of its follow-ups, are my own.

# Erlay

## Overview 

Erlay is an alternative method of announcing transactions between peers in the Bitcoin P2P network. Erlay's main goal is to reduce bandwidth utilization when propagating transaction data.

The protocol builds on the assumption that a significant percentage of the data exchanged between peers when broadcasting transactions belongs to **inventory** (or `INV`) **messages** [^1], which are used to announce the transaction hashes between peers. On receiving an `INV` message containing a collection of transactions, a well-behaved peer will respond with a message requesting all its missing transactions, this is known as a **getdata message** (or `GETDATA`).

Erlay tries to solve this issue by using set reconciliation to work out the transaction differences between two nodes connected to each other instead of just announcing all transactions through all links. To do so, nodes keep a set of transactions to be reconciled (or **reconciliation set**) between each of their Erlay-enabled peers, and reconciliation is performed at regular intervals. Once it is time to reconcile, peers exchange **sketches** [^2] of their reconciliation sets, which can be used to optimally compute the symmetrical difference between their **reconciliation sets** and, therefore, identify which transactions need to be sent by each end of the connection.

Therefore, Erlay aims to reduce traditional transaction relay (called **fanout** from now on) as much as possible but, maybe counterintuitively, not to completely replace it. For Erlay to be optimal, a small amount of **fanout** is still needed, given set reconciliation works best (sketches are smaller and sketch differences are cheaper to compute) when the differences between sets are small. This means that, even if all peers of a given node are Erlay enabled, some transactions will still be exchanged using **fanout**[^3]. Moreover, fanout is more efficient, and considerably faster, than set reconciliation provided the receiving node **does not know about** the transaction being announced.

## Approach

One of the first details to decide on, when implementing Erlay, is how small we want the network transaction **fanout** to be. That is, for each Erlay node, how many peers should it select to exchange transactions using fanout. This is a tradeoff between **bandwidth efficiency** and transaction **propagation latency**: the bigger the fanout, the more initial fast coverage of the network, but the less bandwidth savings. If the fanout rate is too low, we could incur drastically slower transaction propagation times, if it is too big, the bandwidth saving would be too small (maybe even not worth the additional code complexity of implementing Erlay). Furthermore, what kind of nodes are selected also matters: in Bitcoin Core, inbound connections are not trusted [^4], and doing fanout only via inbounds could lead to the transaction propagation being controlled by adversarial peers. On top of that, most nodes do not even accept incoming connections due to their hosting settings. However, completely ignoring inbound connections for fanout would also be a mistake.

This raises the question: **How many peers do we fanout to, and how are they picked?**

Our current approach is to pick a mix of both **inbounds** and **outbound** peers, as long as they are available (i.e. an unreachable node would only pick outbound). [The current version of the PR](https://github.com/bitcoin/bitcoin/pull/30116) uses **1 outbound and 10% of inbounds,** based on [Gleb’s simulations](https://github.com/naumenkogs/txrelaysim/issues/7#issuecomment-901869563).

> :construction: We are currently working on extending this, given **using a percentage of connections** for this has the downside that it makes Erlay bandwidth usage still proportional to the number of connections a node maintains. A goal of Erlay is to allow for a higher number of connections per node without increasing the node’s bandwidth requirements. Sort of a **scale-free** solution.

> :test_tube: The goal would be finding values that do not necessarily scale with the number of connections. This needs to be simulated

Another question that also relates to peer selection is: **How do we decide what peers to fanout to / reconcile with?**

This question has many parts, depending on the adopted solution. The first of them is: are peers selected for fanout/reconciliation on a connection level, or is the decision-making done at the transaction level (i.e. for every transaction, a subset of peers is selected for fanout/reconciliation)?

For this first part of the question, we have decided to go with the latter approach: 

> :exclamation: **Choose peers to fanout to on a transaction level**

The rationale for this is to make the process more fair, otherwise, once a peer is selected for fanout, all transaction exchange would be done using fanout, even if the peer is Erlay-enabled. Also, choosing this on a connection level makes the process way more gameable: if a peer is not happy with our selection they can try to find ways to be “promoted” to reconciliation, depending on how the decision is made (e.g. imagine a naive way of choosing such as flipping a coin per connection, a peer could try to reconnect until they are selected for reconciliation) or even find another peer that picks them for reconciliation.

Given peers are chosen for fanout on a transaction level, the next natural question is: **When do we pick peers for fanout?**

We have two options here, either on the transaction relay schedule (when **transactions are queued** to be sent to peers over the next **trickle** [^5] or on the transaction relay itself (when the `INV` messages are being constructed and transactions are about to be sent out).

### Choosing fanout peers at relay scheduling time

The main motivation for choosing peers at scheduling time is being able to more easily reason about transaction dependencies, that is, how to exchange a given transaction if some of their ancestors are already being scheduled for exchange. Being inconsistent on how dependent transactions are exchanged between peers can hurt **orphan transaction** rates. 

> :test_tube: **This is the approach we are currently following**. Simulation results will be posted on a separate post and linked here after.

### Choosing fanout peers at transaction relay time

Choosing peers at relay time has a better effect on being effective when selecting peers for fanout. The main downside of **deciding at scheduling time** is that it **is not reactive to what may happen between the selection and the actual relay**, that is, a peer that is selected for fanout for a given transaction may announce it to us before our announcement timer goes off, meaning that we will skip that announcement to them when the time comes, effectively reducing the amount of peers we fanout that transaction to. However, **deciding at relay time has the downside of not being reactive enough to things that happen during the fanout selection process**, such as the previous ancestors example, but also if we have already selected enough peers to fanout to, but suddenly we cannot reconcile the given transaction with the remaining of our peers (e.g. the transaction has a collision with an existing one in their set [^6], their reconciliation sets are full, …) we would end up having a **higher fanout rate** than intended, and we would not be able to compensate with some of the peers that we have already decided for, given those messages would have already been sent.

> :test_tube: **This is a valid alternative approach**. Simulation results will be posted on a separate post and linked here after.

### Making transactions available for fanout/reconciliation

Independently of the method chosen to pick fanout peers, **transactions must only be available after a given delay** this helps prevent adversarial peers from knowing when a transaction enters our mempool, which can be used to learn who the transaction originator is. This behavior is not new, and it has been used for years. This process is known as **trickle** [^5]) and it is applied to every connection depending on their type.

For Erlay connections, transactions **are made available for reconciliation when the timer for the given peer trickles**. Transactions flagged for reconciliation between intervals are delayed and won’t be part of a sketch should a reconciliation be scheduled. 

The way we currently deal with this is by having two internal collections behind the reconciliation set interface, the **ready set**, and the **delayed set**. Transactions are added to the delayed set and readied on trickle (moved from one to the other). The set is seen as a single entity though, therefore the limits, collisions, deletions, … are performed by checking both internal collections. **For reconciliation purposes, only transactions in the ready set are used.**

An interesting implication of this is that transactions added to the reconciliation set at `t-1` are made available at time `t` **strictly after fanout.** The process of making the timer advance belongs to the networking [thread that builds and sends out network messages](https://github.com/bitcoin/bitcoin/blob/35000e34cf339e46d62b757c3723057724d23637/src/net_processing.cpp#L5676) (more concretely, during the `INV` message building). Being this a sequential process, **fanout data from `t-1` is always processed before reconciliation data**, even if a reconciliation request happens right after trickling. This is especially interesting in situations [where ancestors need to be considered](https://www.notion.so/Erlay-1537b3fef97780038aa6fa2ea5aef421?pvs=21), since we should not end up in a situation where we produce orphans by splitting ancestors between fanout and reconciliation.

Let’s consider the following example:

```cmd
                                    P1                   P2             
                                                                        
 ┌──────┐                           Fanout: txA, txB     Fanout: txA    
 │ txA  ├────────┐                  Recon: ∅             Recon: txB     
 └──────┘        │     ┌──────┐     Processed: ∅         Processed: ∅   
                 ├─────┤ txC  │                                         
 ┌──────┐        │     └──────┘     P3                   P4             
 │ txA  ├────────┘                                                      
 └──────┘                           Fanout: ∅            Fanout: txA    
                                    Recon: txB           Recon: ∅       
                                    Processed: txA       Processed: txB 
                                                                        

txC is added at time t-1,and made available at time t
txA and txB were added at time t-n, for n < 1
```

Given the outlaid setting where `txA` and `txB` have already been added to their respective sets at `t-n` (for `n<1`), and `txC` is added at time `t-1`. How do we handle the following situations:

- `P1`: if `txC` is picked for reconciliation, we can be sure that all ancestors will be announced before it. If it is picked for fanout, all the ancestor set will be announced at the same time.
- `P2`: if `txC` is picked for reconciliation, we know that `txA` will be announced first, leaving `txB` and `txC` to be reconciled at a later time. If it is selected for fanout, `txB` **would also need to be moved from reconciliation to fanout, otherwise** `txC` **could become an orphan.** This can happen if fanout for `t` happens before reconciliation for `t-n`, since reconciliation needs to be actively requested.
- `P3`: if `txC` is picked for reconciliation, `txB` and `txC` will be exchanged using the same method, so all good. If it is picked for fanout, `txB` would also need to be moved from reconciliation to fanout (for the same reasons pointed out in `P2`).
- `P4`: if `txC` is picked for reconciliation, we are fine, given fanout for `txA` must happen before reconciliation for `txC`. If it is picked for fanout, it will be exchanged alongside `txA`, so all good.

Notice there could be two additional cases, one analogous to `P2` with `txA` and `txB` swapped, and one with all dependencies in the reconciliation set. Both cases have the same implications as `P2`. In the first case the order doesn’t really matter, whereas in the second, if `txC` is picked for fanout, all others would need to be moved, and if not, all the ancestor set can be easily reconciled. Cases where the two sets are empty (ancestors have already been processed) are uninteresting for this matter.

Therefore, we only need to be careful about flagging dependent transactions for fanout if they have ancestors in the reconciliation set: 

> :exclamation: If a transaction that has ancestors in the reconciliation set of a given peer is flagged for fanout, it’ll trigger moving all the ancestors from reconciliation to fanout. **This is implemented in the current approach** 

# Simulations

Simulations will be presented in independent posts, but I'll be linking them here for reference and easy access. 

# Acknowledgements

Many of the results and ideas presented here originate from discussions with several people, both inside and outside the working group including, Gleb Naumenko (@naumenkogs), Pieter Wuille (@sipa), Greg Maxwell, Mark Erhardt (@murch), Marco De Leon (@marcofleon) and others.

 [^1]: As presented in the [Erlay paper](https://arxiv.org/abs/1905.10518), most of these transaction announcements are redundant, given the transaction has already been announced to (or even received by) a given node. This is the main assumption Erlay builds on top of
[^2]: What sketches are, and how are they computed is outside the scope of this document. They can be seen as black boxes with two properties: 1. Given two sketches, the symmetrical difference of the elements in the reconciliation sets can be computed. 2. Sketch size grows proportionally to the expected difference, instead of to the number of elements in it
[^3]: Under what conditions these peers are picked is really implementation dependent, and one of the topics that will be discussed later in this document
[^4]: No connection is really “trusted”, but inbounds are seen as likely adversarial, given they are not selected by us in a hard-to-bias manner. A whole section could be written on this, but to keep it simple, **inbounds = bad**
[^5]: Trickling happens on a timer that mimics a Poisson process. Inbound peers are on a shared timer with an expected value of `5s`, whereas outbound peers have their own timer with an expected value of `2s`. That means, on average, transactions are sent out to inbound/outbound peers every 5s/2s
[^6]: Transactions added to a peer’s **reconciliation set** are identified using **a shorter id** than transaction ids, meaning that the changes of an id collision are higher

-------------------------

