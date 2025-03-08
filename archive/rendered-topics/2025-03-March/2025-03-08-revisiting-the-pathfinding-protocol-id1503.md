# Revisiting the pathfinding protocol

brh28 | 2025-03-08 20:26:32 UTC | #1

Hey all,

In my view, pathfinding still seems to be one of the biggest pain-points of lightning today, but I haven't seen much discussion about it at the protocol level. I'm not a protocol dev so I'm hoping someone simply point me right in the direction. Otherwise, I'm hoping this can be a starting point.  

#### Background

To route a payment on the Lightning Network, a sender must have a path to the destination using channels which contain sufficient liquidity and forwarding fees. At present, only channel topology and routing fees are shared to nodes on the network via the gossip protocol. Channel balance, however, is not shared with the network. Therefore, when constructing a route, a sender has an incomplete set of data required to construct a feasible path, leaving them instead to probe the network (i.e guess-and-check) for a functional path. Additionally, fee updates are rate-limited, which is a constraint that reduces a routing nodes ability to influence the flow of liquidity through their channels. Finally, nodes must constantly sync the current state of the network, and even then, may not have the benefit of historical data to inform them of channel reliability.

At it's root, these issues stem from the source node's dependence on received gossip data. The gossip protocol works by flooding the network to ensure everyone receives each message. Due to the global nature of this protocol, messages are public and must be limited in count. 

In short, pathfinding with gossip data is unreliable, especially as the network grows. When channels are unreliable, network participants are incentivized to outsource node operations to larger (likely custodial) nodes who have more liquidity and more transaction volume to probe the network.

#### Solution - Nodes ask channel peers for a path to the destination. 

As far as I know, this isn't entirely a novel idea and it's actually what Phoenix does today with their Acinq node, but outside of the lightning protocol. Yes, a node reveals their destination to their channel peer, but they get a much better payment reliability while preserving custody of their funds.

Now, let's say we extend these query messages to all nodes on the network and that, as an example, Alice is completing an invoice to Dave. Rather than requiring Alice to construct a path using her own map of the network, Alice can ask her channel peer (Bob) for a path to Dave. Bob knows he has liquidity through Carol, so he asks Carol for a path. This process repeats until we reach Dave. With zero knowledge of the graph, the sender (Alice) can find a feasible path to Dave without requiring nodes to gossip their channel balances. It also (potentially) allows nodes to set their routing fees on a per transaction basis, or at least at a more granular level.   

It should also be noted that this approach scales much better as nodes do not require full knowledge of the graph to send payments.

Of course, the primary cost of such a scheme is privacy as a sender reveals their destination to their channel peer, but I think:
1. the current evolution of the network ("Lightning-as-a-Service") shows many/most participants prefer reliability over privacy.  
2. Those that are privacy focused could still rely exclusively on gossip data for path-finding. 
3. Emergence of trampoline & blinded paths suggest destination could be concealed with higher-level routing. This approach seems to resemble other networks with reliability at the base and privacy built on top.

Would love to hear some thoughts.

-------------------------

