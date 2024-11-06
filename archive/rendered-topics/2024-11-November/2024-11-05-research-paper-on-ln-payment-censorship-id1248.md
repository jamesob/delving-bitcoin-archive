# Research Paper on LN Payment Censorship

cndolo | 2024-11-05 11:57:11 UTC | #1

Dear all,

In a recent paper [1], we explored the threat of censorship by a network-level
adversary, such as an autonomous system (AS), in the Lightning network.
I would like to summarise the contributions in this post, while inviting
comments and discussions on the subject from the broader LN community.

## Background

A relatively recent paper [2] presented a comprehensive attack on privacy in the
LN that exploits the fact that P2P messages can be identified using
network-level information, i.e., TCP headers, despite encryption.
When combined with the sequence of messages, it is possible for an adversary
observing network traffic to determine what application messages are
being exchanged by nodes.
Based on collected packet traces, the authors showed that they can trivially
classify application messages which serves as the foundation for the further
attack.
The fact that the network layer is quite centralised makes the potential impact
even greater, as most channels are hosted at just a handful of ASs.

## Censorship

Motivated by the network-level centralisation, we studied the LN's vulnerability
to censorship.
We maintain the same attacker model as [2], which is a powerful but malicious
network-level attacker such as an AS.
However, our attacker is keen on imposing censorship on **only** "their" nodes
while going undetected so as to maintain a certain degree of plausible
deniability.
To this end, the adversary only imposes censorship on payments in the LN and
does not interfere with any other LN operations or the greater network.

Assuming the adversary is monitoring traffic exchanged between to/from one of
their nodes, the basic steps they may follow can be summarised as follows:

  1. Identify TCP segments related to LN payments. Anything else besides such
     segments is not of interest/relevance to the adversary.
     This is typically accomplished by identifying an `update_add_htlc` message.
  2. Interfere with the payment by dropping the first outgoing `revoke_and_ack`
     message that is sent by the initiator. The specific message is not a
     requirement but was selected due its terminal position as it affords the
     attacker time to confirm that a payment is underway.
  3. Drop every subsequent `revoke_and_ack` message that will be transmitted by
     the initiator until the timeout eventually expires.

In doing so, the affected nodes are still able to open channels, send gossip
etc. which supports the adversary's goal of remaining undetected.
Clearly, a single node operator is technically able to execute these same steps,
however, the impact is related to the number of affected channels.
In the paper, we discuss how and show that the adversary can refine their attack
by determining the node's role in the payment path.
Similarly, this is achieved using only the information available at the network
layer.

In summary, these are some the key points from our analysis:

  1. We implemented classification rules (using message size, direction and
     sequence) in XDP and netfilter programs which we deployed to test
     networks.
     Complementary to [2], we showed that such classification is trivially
     and efficiently possible on the fly.
  2. As a result, it is feasible to execute the described censorship attack,
     and will remain to be so unless the currently-estimated throughput
     increases significantly.
  3. Perhaps unsurprisingly, the network-level centralisation increases the
   potential magnitude of the attack.
   Simulations, however, showed this varies greatly based on the attacking
   AS and/or exact censorship strategy.

## Mitigation

The authors of [2] already discussed some countermeasures that may be
implemented such as deviating from the default port or using Tor.
While the former is likely not effective enough, the latter can be generalised
to the use of length-hiding schemes such as padding.
However, as we argue in our paper, selecting an adequate form of padding is not
trivial.
For one, an adversary is likely to be able to defeat simple padding mechanisms
such as constant-length messages using knowledge of the LN protocol and timing.
On the other hand, the introduced overhead may not be negligible and should thus
be taken into consideration.

There is a lot of related work in the area of website fingerprinting defences
that we can likely make use of here.
For instance, adaptive padding mechanisms such as WTF-Pad [3] et al. or defence
frameworks such as Maybenot [4] come to mind.

Alternatively (or perhaps, alongside) exploring the above, addressing the
network-level centralisation on the application layer, e.g., by taking AS
information during pathfinding into consideration, may also be worth looking at.

[1]: https://drops.dagstuhl.de/entities/document/10.4230/LIPIcs.AFT.2024.12
[2]: https://ieeexplore.ieee.org/document/10190502
[3]: https://arxiv.org/pdf/1512.00524
[4]: https://dl.acm.org/doi/pdf/10.1145/3603216.3624953

-------------------------

