# Path Queries: Addressing Payment Reliability and Routing limitations

brh28 | 2025-05-07 22:19:36 UTC | #1

## Introduction

To route a payment on the Lightning Network, a sender must choose a path to the destination using channels which contain sufficient liquidity and meet certain forwarding rules (e.g routing fees). As neither the liquidity nor the balance in payment channels are shared with the network, a sender does not know if there is sufficient liquidity in each channel when constructing a path. This is referred to as *liquidity uncertainty.* With no other mechanism to resolve this uncertainty, nodes are left to a trial-and-error strategy. That is, attempt the payment anyway and hope it works. If it doesn't, update the local view of the graph and try again. This approach to routing has a host of issues:

#### 1\. Sub-optimal routing 

Without prior knowledge, a channels probability of error due to insufficient liquidity approximates the ratio of the payment amount to the channel capacity, which means larger payments through relatively low capacity channels are prone to error. Furthermore, this risk is compounded at every channel, which means longer paths become exponentially difficult. See appendix A for details.

Routing calculations must account for this uncertainty by favoring shorter paths and higher capacity channels. Not only does this effect the payment sender who's more likely to pay extra in fees for reliable liquidity, it's also a centralizing force on the network. See appendix B for details   


Low payment success rates implies a large set of potential routes. When the final route is unknown, routing fees are more difficult to predict. 

Failed payments are a burden to routing nodes in the failing sub-path in the form of locked liquidity, HTLC slots, and wasted computational resources.

#### 2. Slow Discovery Process

By using payments to discover feasible paths, HTLCs need to be set up and torn down at each hop, which requires multiple rounds of communication between peers. This is a lot of wasted time whenever a payment fails. Furthermore, this process must be executed serially to avoid the delivery of multiple successful payments, which fundamentally limits the rate at which path discovery can occur.

#### 3. Routing limitations

To route a payment, the sender needs the latest updates to the graph. This requirement is a burden to the sender, who needs to constantly sync the channel graph, and to routers, who must limit their channel updates.

* * *

## Proposal

Allow nodes to share routing information with each other in the form of path queries. The proposal includes the following optional messages, which allow nodes to cooperatively construct a path:

1.  `path_query`

- source_node_id
    
- destination_node_id
    
- amount
    
- expiry
    

2.  `path_reply`

- path
    
    - amount
        
    - fees
        
    - cltv_delta
        

3.  `reject_path_query`

- reason

Upon receiving a `path_query`, a node can choose how it wants to respond, including rejecting or ignoring it. The `path_reply` message helps the requester deliver a potential payment because it leverages routing information at the queried node. Notably, this solves the liquidity uncertainty problem for the queried node's channels because a node knows it's own balances and can respond accordingly. In contrast to the single execution onion, queries are lightweight and can be made concurrently. A parallel discovery process reduces liquidity uncertainty at a significantly faster rate than onions. Finally, it's worth noting that a router can respond with any policy (e.g fees) it desires, unconstrained by existing rate limits.

## Putting into practice

The proposal outlines a basic set of messages and it is for the node to choose their own request & response strategies, including *who* they want to talk to (any subset of nodes), *what* they want to respond to (e.g minimum amounts) and any rate limits (number of requests and replies/paths). While there are innumerable strategies that may evolve, let's walk-through a simple example where all nodes adopt a PEER_ONLY strategy (i.e nodes interact only with their channel peers):

```
        +--- A ------- B ---+          
        |              |    |
        |              |    |
L -----	S    +---------+    R
        |    |              |
        |    |              |
        +--- C ------- D ---+           
```


Before attempting the payment, the sender (S) may choose to query any subset of it's channel peers. The sender already knows that (L) is a leaf so does not query it. Alice (A) advertises the lowest routing fees but, since this is a larger payment, the sender decides to make concurrent queries to both Alice and Carol:

- Alice receives a `path_query` message requesting a path from herself (A) to the receiver (R). She sees she does not have the outbound liquidity to Bob (B) to complete the payment, so can either immediately respond with a `reject_path_query` with a reason indicating a temporary failure to find a route, or wait until liquidity becomes available to send a `path_reply`.
- Carol (C) receives a `path_query` requesting a path from herself to the receiver (R). She has sufficient outbound liquidity through Dave (D), but before responding to the sender, she decides to query Dave:
    - Dave receives a query from Carol for a path from himself (D) to the receiver. Similar to Alice, Dave responds that he has no route available 
- Upon discovering insufficient liquidity from D -> R, Carol splits the sender amount and concurrently queries Bob (B) and Dave (D) with their respective splits. 
    - Dave receives a new query requesting a path from himself (D) to the receiver, but of a lesser amount. This he does have the liquidity for! Since Dave knows he can route the requested payment, he responds to Carol with the given path and routing details. 
    - Bob (B) receives a new query requesting a path from himself (B) to the receiver for his split amount. Similar to Dave, he knows he can route the payment, so resonds to Carol with his routing details. 
- Upon receiving the path details from Bob and Dave, Carol can now confidently assemble a MPP from herself to the receiver. She constructs the MPP, adds her own routing details and sends a `path_reply` to the sender. 
- Upon receiving the `path_reply` from Carol, Alice attempts the payment and on her first attempt, the payment succeeds.
  

As you can see, path queries enable nodes to use queries to concurrently probe the network for feasible paths. When chained together, these queries swarm to the destination. Each hop knows it's channel balance and can therefore reduce the liquidity uncertainty for their channel(s) in the path. After receiving a `path_reply` a node can prepend itself to the path and back-propogate it to the source. 

While the small example above illustrates the process, it is important to remember that liquidity uncertainty is particularly detrimental for longer paths on larger networks; trial-and-error may work for a small network like this, but does not scale to a growing number of nodes.

#### Benefits

As a simple set of flexible messages, it's impossible to define all potential use cases. However, benefits can be broadly outlined as follows:

1.  Improves routes for larger payments, including lower fees, lower expected payment delivery times, and better fee estimation.
2.  Eliminates reliance of a fully synced channel graph. This is especially useful for mobile nodes who are not always available to receive updates and may prefer a relationship with an LSP.
3.  Enables more dynamic routing policy, whereby nodes can share updates without globally enforced rate limits.
4.  Distributes routing more evenly across the network, making smaller routing nodes more competitive
5. Allow for network growth

#### Addressing potential concerns

1.  Privacy.

Naturally, any time information is shared, there is a privacy implication. However, this proposal is entirely optional; a node may choose to reveal as much or as little information to whomever they choose. For example, rather than revealing the true payment destination, a node can query a sub-path, which nicely compliments blinded paths. 

Additionally, the use of onion messages could significantly improve a node's anonymity set, especially when querying node's multiple hops away. 

2.  Denial-of-service 

Nodes may choose their own response strategies, including filtering requests (e.g minimum amount) and setting rate limits.

## Expanding the messages 

The messages defined in this proposal are pretty bare. New fields can be added to these messages to enhance a node's capabilites:

- Filters to the `path_query` to reduce the number of reply messages, such as a `maximum_fee`
- Confidence scores that tell the requester the expected likelihood of payment delivery; the higher the routing node's confidence, the more a path suggestion behaves like a *quote* for delivery.
- Time limits. A node can define a window of time their interested in a given path (e.g 1min, 1hr, 1day, always) and get notified with updates.

## Comparisons to Trampoline

As far as I can tell, path queries can be used to accomplish everything trampoline routing proposes. The [trampoline proposal](https://github.com/lightning/bolts/blob/32fd16e243d7ea4438c6646f35e731a73d691ca7/proposals/trampoline.md#introduction) states, "The main goal of trampoline routing is to reduce the amount of gossip that constrained nodes need to sync". Furthermore, it states "constrained devices should only keep a small part of the network and leverage trampoline nodes to route payments." With path queries, a node does not need any knowledge of the graph, but instead only a connection to a supporting peer.

Trampoline pursues this goal while also preserving anonymity for the sender and receiver. This is done by including multiple hops in the trampoline route. Using path queries in the form of onion messages, a source node can attain a comparable anonymity set with only a single 'trampoline hop' using the following process: 

1. Select a node with path query support as a 'trampoline' hop.  
2. Query channel peer for a path to trampoline. 
3. Query trampoline for a path from trampoline to final destination.
4. Send payment using aggregate route 

Note that while the pathfinding process is similar to trampoline in that it leverages routing information of other nodes, the final route does not use a special "trampoline route", but rather a regular onion route. This means the trampoline hop has no information about the source other than the in-bound channel.   

Also, by requiring only a single trampoline hop, the full route is likely to be shorter. And as discussed above, paths derived from queries are likely to have higher success rates and lower fees while moving across the network.  

## Moving Forward

My primary goal is to get feedback, iterate on the proposal and open a draft PR into the BOLTs. Moving forward, I would be happy to work with any implementations interested in this feature. 

* * *

## Appendix A - Payment Failure Probabilities 

Referencing [Mastering the Lightning Network](https://github.com/lnbook/lnbook/blob/54453c7b1cf82186614ab929b80876ba18bdc65d/12_path_finding.asciidoc#liquidity-uncertainty-and-probability)

#### For a given channel 

The success probability function (only considering liquidity) is: 

P(a) = (c + 1 - a)/(c + 1),   where a = payment amount, c = channel capacity

Using limits for approximation, we can eliminate the `+1`s and reduce the function to:

P(a) = (c - a)/c = 1 - (a / c)

As an additive inverse, we can deduce our probability of failure due to liquidity as (a / c) - the ratio of payment amount to the channel capacity. 

#### For a payment path

The success probability of a payment path is the product of probabilities for each channel in the path(s):

\[$P_{payment} = \prod_{i=1}^n P_i$\]

Likewise, probability of failure is a product of probabilities.

## Appendix B - Network Centralization

As this [2021 study](https://arxiv.org/pdf/2102.09256) shows, channel attachment strategies that connect to central nodes (e.g High degree, Betweeness) have significantally better payment success rates and better overall performance than more distributed strategies (e.g k-median), even when the distributed strategies offer lower fees. This is evidence that short paths through high capacity channels give larger routing nodes (high channel count, high capacity) a competitive advantage for larger payments, allowing them to capture more in routing fees.

Additionally, payment success probabilities are higher for nodes with more volume ([see here](https://arxiv.org/pdf/2006.14358)) because their previous payment attempts reduce their liquidity uncertainty. Lower uncertainty ultimately leads to better payment performance, meaning small nodes are at a disadvantage as a payment originator as well as routing.

-------------------------

brh28 | 2025-05-09 17:39:59 UTC | #2

Expanding on privacy implications...

Naturally, any time information is shared, there is a privacy implication. A `path_query` reveals a downstream node - either a hop or the destination - to the prospective routing node. When iterated upon, each node in the path becomes aware of the *queried* destination. Meanwhile, a `path_reply` implicitly reveals information about channel balances. As so, let's consider sender/receiver anonymity and channel balance privacy:

*Sender Anonymity*

While a single query does not tell a routing node about the source of a payment, the number of queries a routing node receives and whom they come from may reduce the anonymity set of the *query origin*. Depending on the nature of the payment, the sender may consider anonymity in it's path construction process, including adding trampoline hops or opting out of queries altogether.

*Receiver Anonymity*

While the receiver does not have a choice in how a payment gets routed to them, they do get to choose the entry point of the payment via blinded routes. Using path queries, a receiver can construct more reliable paths to itself; the longer the path, the more anonymity from the sender and it's gang of routing nodes. The receiver may also choose to construct their sub-path using trampolines to prevent routing nodes from discovering full paths.

*Privacy of channel balances*

First, it's important to note  the following:

1. In source-based routing, payment reliability and channel balance privacy are fundamentally at odds with one another. If a path is constructed with zero knowledge of channel balances, payment success probabilities are low, and vice-versa, perfect knowledge leads to optimal routing. 
2. Channel balance is shared between channel peers, which means nodes already have a trust relationship with their peers regarding privacy.
3. Channel balance information can already be obtained by other nodes via probing. 

With that in mind, path queries differ from trial-and-error (including probing) in the manner that liquidity uncertainty is reduced. Trial-and-error informs the *sender* about liquidity on the path, while path queries informs the *requester* about liquidity on the path. In our PEER_ONLY strategy described above, the sender (S) gained no information about liquidity on the network other than what was used for the final path. While probing remains an unsolved problem, path queries enable better information control as nodes can choose *who* they want to reveal liquidity information with.

-------------------------

renepickhardt | 2025-05-14 09:36:17 UTC | #3

Thank you for your proposal and thoughts. I think most people will agree with the problem setting that you are describing. I also agree that there is a relation to trampoline routing. I was missing a bit the comparison to the proposal to [share liquidity information within the local neighborhood as proposed here](https://github.com/lightning/bolts/pull/780). I wonder about the trade offs between those two ideas? For that keep in mind that the recipient of a payment could potentially share its local view of the network in the invoice. 

While I agree that the ability for nodes to cascade path queries seems interesting and useful I was wondering weather it could lead to eventually probe the entire network and thus quite some communication overhead - which would have to be redone all the time as the liquidity state of the network should be rather dynamic. 


Please allow me my concerns that the benefits of your proposal may not be as high as one could think:

Generally [we have an issue with payments being infeasible](https://delvingbitcoin.org/t/estimating-likelihood-for-lightning-payments-to-be-in-feasible/973) - even if full information about liquidity was known. The uncertainty about liquidity can be [addressed  by implementing probabilistic models](https://arxiv.org/abs/2103.08576) (which all implementations do). Of course removing the uncertainty is better than handling the uncertainty. 

[quote="brh28, post:1, topic:1672"]
#### Benefits

As a simple set of flexible messages, it’s impossible to define all potential use cases. However, benefits can be broadly outlined as follows:

1. Improves routes for larger payments, including lower fees, lower expected payment delivery times, and better fee estimation.
[/quote]

The supported size of the payment depends on the liquidity state and at most to a small degree on the uncertainty. In particular as mentioned before the fact that payments of certain sizes are expected to be infeasible has nothing to do with the uncertainty about the liquidity. In particular [I encourage you to look at this study](https://github.com/renepickhardt/Lightning-Network-Limitations/blob/eec0945424289915d02868f2a691f160df5fead1/likelihood-of-payment-possability/An%20upper%20Bound%20for%20the%20Probability%20to%20be%20able%20to%20successfully%20conduct%20a%20Payment%20on%20the%20Lightning%20Network.ipynb) which [has been independently verified](https://stacker.news/items/ir412708). From that research you can find this diagram: 

![image|562x391](upload://zxUIZz0LhEj3tFarfCI40ReZbEG.png)

It indicates that we expect that 2.5% of all payments cannot exceed the size of 766 sats. Yes the uncertainty makes it harder to find those channels that allow to transport those 766 sats but removing the uncertainty does not increase that size at all. in particular [it was shown that the bottlekneck is in 95% of all cases within the outbound liquidity of the sender and the inbound liquidity of the recipient.](https://arxiv.org/abs/2107.05322) 

Similarly I doubt that it is obvious that the removal of the uncertainty will yield lower fees. A counterargument may be: Routing nodes may - given more knowledge about the liquidity - increase the fees of their channels if they see that their liquidity is desirable. 

[quote="brh28, post:1, topic:1672"]
2. Eliminates reliance of a fully synced channel graph. This is especially useful for mobile nodes who are not always available to receive updates and may prefer a relationship with an LSP.
[/quote]

As with trampoline routing which we already have.

[quote="brh28, post:1, topic:1672"]
3. Enables more dynamic routing policy, whereby nodes can share updates without globally enforced rate limits.
[/quote]

I expect this will more quickly lead to deplete channels. Also it is not clear how many nodes will be queried and asked for one routing / payment request. In your proposal you already suggested that a sender can send many of those queries in parallel.

[quote="brh28, post:1, topic:1672"]
4. Distributes routing more evenly across the network, making smaller routing nodes more competitive
[/quote]

see above. Cheap channels will drain more quickly. If at all the fact that nodes have knowledge will decrease loead balancing and not increase it.

[quote="brh28, post:1, topic:1672"]
5. Allow for network growth
[/quote]

How so? The fact that payment amounts become infeasible is positively corelated with the number of participants (and negatively corelated with the amount of coins on the network). In particular network growth is limited by onchain constraints anyway. So I doubt that this is a benefit of your proposal.

## Summary 

While I attacked your benefits I like the proposal as I think it is in some sense more elegant and cleaner than asking within the friend of a friend network. However I am not convinced that both of those ideas will really improve the reliability situation drastically - though I agree it would be nice to have some uncertainty removal. 

What for example about a simple flag in channel updates that indicates on which side the liquidity is currently located? Given that channels are expected to deplete this information would be most useful anyway. Nodes could quickly share it - potentially even network wide - with a single message for all their channels. 

Thus I think if the community is willing to think about protocols to reduce uncertainty of liquidity we should think about which of the ideas is the best way forward by comparing those ideas to each other

-------------------------

brh28 | 2025-05-14 17:25:28 UTC | #4

Hey Rene, I really appreciate your feedback!  

[quote="renepickhardt, post:3, topic:1672"]
I was wondering weather it could lead to eventually probe the entire network and thus quite some communication overhead - which would have to be redone all the time as the liquidity state of the network should be rather dynamic.
[/quote]

Quite the opposite, the purpose of the proposal is to allow nodes to find a reliable path *without* prior knowledge of the graph, meaning that continuously monitoring the network via probes is not necessary for successful payments. At the peer level, a `path_query` & `path_reply` only requires one round-trip vs the 3 round trips required to set up and tear down an HTLC (1.5 roundtrips per commit x 2 commits).

[quote="renepickhardt, post:3, topic:1672"]
Generally [we have an issue with payments being infeasible](https://delvingbitcoin.org/t/estimating-likelihood-for-lightning-payments-to-be-in-feasible/973) - even if full information about liquidity was known
[/quote]

Agree, the proposal does not intend to create feasible paths, but rather, to quickly find them.  

[quote="renepickhardt, post:3, topic:1672"]
A counterargument may be: Routing nodes may - given more knowledge about the liquidity - increase the fees of their channels if they see that their liquidity is desirable.
[/quote]

Alluding to *Privacy of channel balances* above, queries give routing nodes better control of their routing information, including liquidity; they can choose *who* to reveal information to and *how much* information they want to provide. Generally speaking, the more channels a node has, the harder it is to make inferences about their liquidity state from an offered path. 

[quote="renepickhardt, post:3, topic:1672"]
As with trampoline routing which we already have.
[/quote]

Trampoline adds much more complexity at the protocol level. Additionally, it does not completely eliminate syncing as the node still needs a feasible path to the first trampoline hop. 
[quote="renepickhardt, post:3, topic:1672"]
I expect this will more quickly lead to deplete channels.
[/quote]

I disagree, dynamic policy can actually be used to help balance channels, as nodes are able to change their routing fee in each direction based on their current liquidity state. This is conceptually similar to rate cards, but offers routing nodes more flexibility and payment senders do not need to guess the rate.    

[quote="renepickhardt, post:3, topic:1672"]
Also it is not clear how many nodes will be queried and asked for one routing / payment request.
[/quote]

Could you elaborate on your concern here? Request and response strategies are freely determined by the node.


[quote="renepickhardt, post:3, topic:1672"]
How so? The fact that payment amounts become infeasible is positively corelated with the number of participants (and negatively corelated with the amount of coins on the network). In particular network growth is limited by onchain constraints anyway. So I doubt that this is a benefit of your proposal.
[/quote]

Apologies, I should have been more specific here; I'm referring to growth of the network *diameter* here. As the network grows outwards, there must be a way to reliably find longer paths (when feasible). Conversely, if the network converged on a hub-and-spoke model, that would improve payment reliability, but it's not a desired outcome. 

As you mention, onchain constraints (block size) is a limiting factor to the *growth rate* of the network - not the maximum size. Over time, the network should still support an indefinite number of nodes and channels. Furthermore, payment channels could theoretically be constructed on different settlement layers (or 'realms'), so I don't think we should assume that growth is bounded by block size.  

***

I believe this response is already long enough, so I'll leave it here. Let me know if you want me to go more into anything I missed, including comparisons to other approaches such as FofF or liquidity indications.

-------------------------

