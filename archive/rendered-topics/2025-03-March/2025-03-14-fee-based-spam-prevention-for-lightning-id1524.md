# Fee-Based Spam Prevention For Lightning

JohnLaw | 2025-03-14 23:58:49 UTC | #1

TL;DR
=====
* Jager [1][2] and Teinturier [8][9] proposed Upfront Fees and Time-Dependent Reverse Hold Fees to prevent spam on the Lightning Network (LN).
  - Their protocols charged fees that paid for all significant routing costs.
  - However, routers could steal fees from others.
* This post extends their protocols by adding:
  - secrets that prove a payment reached a given routing node, and
  - burn outputs (as proposed by Riard and Naumenko [6][7]) to prevent theft and encourage cooperation.
* The resulting protocols charge fees for all significant routing costs and those fees cannot be stolen.
  - As a result, they should reduce spam and increase the efficiency of the LN.

Paper
=====
A more complete description (including figures, packet details, related work and security analysis) is given in a paper [5].

Fee Types
=========
The protocols presented here collect 3 types of fees:
  * an Upfront Fee paid by the sender to the downstream node in each channel which updates the channel state to include an HTLC for the sender's Lightning payment,
  * a Hold Fee paid by any node that delays the payment to each upstream node that they delay, and
  * a Success Fee paid by the sender to each node that helps deliver a successful Lightning payment.

The Upfront Fee pays for:
  * the computation and communication costs of routing the payment,
  * the cost of allocating a Commitment transaction output ("slot") to the payment,
  * the cost of allocating channel funds to the payment for the seconds that can be required for a non-delayed payment,
  * the risk of not fulfilling the HTLC with one's upstream partner despite one's downstream partner having fulfilled their HTLC,
  * the risk of having to pay a Hold Fee (as described below), and
  * the risk of burning funds (as described below).

Before routing or receiving a Lightning payment, each node publishes its hold_grace_period_delta_msec parameter giving the minimum time they can take without having to pay for delaying a Lightning payment.
If a router or the destination delays a Lightning payment beyond its grace period, it pays a Hold Fee to each of the upstream nodes for:
  * the cost of their capital held for the length of the delay, and
  * the inconvenience imposed on the sender for the length of the delay.

The Success Fee pays for the service of successfully completing a payment.
As a result, it motivates the routers and the destination to make the payment attempt succeed.

Fee Collection
==============
As in the current Lightning protocol, the Success Fee is paid by increasing the size of each HTLC in order to include fees for the downstream node in the current channel and in all later channels.
As a result, any node that participates in a successful payment will be rewarded with a Success Fee.

The collection of Upfront and Hold Fees relies on the addition of a burn output to the Lightning channel's transactions.
The burn output can be spent by anyone after a 20-year delay (so the burn output will be claimed by a miner 20 years in the future).
Both parties in a Lightning channel put some of their funds in the burn output, and when the payment's HTLC is resolved the protocol determines how the burn output's funds are divided between the channel partners.
The protocol allows both parties to calculate the same division of the burn output's funds, and the fact that those funds will be burned unless the parties agree on their division incentivizes cooperation.

Calculating Upfront Fees
========================
This post presents two protocols for calculating Upfront Fees.
In both protocols, the upstream node in each channel stakes (by adding them to the burn output) Upfront Fee funds that are sufficient to pay Upfront Fees to all of the nodes that are downstream of it along the payment's path.
The staked Upfront Fees are then divided according to how far the payment is routed.

Let n denote the number of payment hops, with node 0 being the sender and node n being the destination.
For 0 < k <= n, a payment "stops" at node k if node k is a downstream node in the last channel which updated its state to include an HTLC for the given payment.

In both protocols the channel partners create a contract for transferring Upfront Fee funds to the downstream node if that node provides a required secret before the contract's expiry.
As a result, the contract for transferring Upfront funds is similar to an HTLC, but with multiple cases based on which secret is provided by the downstream node.
These different secrets correspond to different nodes at which the payment could stop.

Preimage Per Stopping Node
--------------------------
The first protocol uses secrets that are hash preimages, with a separate preimage per stopping node.

The hop payload of the onion packet received by node i includes the secret X_i.

The update_add_htlc packet sent to node i includes the list of pairs (Y_i, t_{i,i}), (Y_{i+1}, t_{i,i+1}) ... (Y_n, t_{i,n}) where for each k, Y_k = hash(X_k).
The value t_{i,k} equals the sum of the Upfront Fees for nodes i through k, so t_{i,k} is the amount of Upfront Fees transferred (via the burn output) from node i-1 to node i for a payment that stops at node k.

When node i-1 sends an update_add_htlc packet to node i, both nodes update the channel state to include the given HTLC.
In addition, node i-1 stakes t_{i,n} funds (which is the maximum possible transfer of Upfront Fees to node i) to the burn output, and both nodes contribute a fixed fraction of t_{i,n} funds (called "matching funds") to the burn output.

When the payment stops at some node k, node k includes the secret X_k in the update_fulfill_htlc or update_fail_htlc packet that it sends to its upstream partner.
Each node i, 0 < i < k, then includes this same secret X_k in the update_fulfill_htlc or update_fail_htlc packet that it sends to its upstream partner.

Nodes i-1 and i then update their channel state to reflect the resolution of the HTLC and the division of the Upfront funds.
Specifically, nodes i-1 and i determine the value of k such that Y_k = hash(X_k), transfer t_{i,k} Upfront funds from the burn output to node i, refund the remaining staked Upfront funds to node i-1, and refund the matching funds to both nodes.

This simple protocol allows each node to receive its correct Upfront Fee, but it reduces privacy because two routers can use the list of pairs in the update_add_htlc packet to determine they are routing the same payment.

Discrete Logarithms
-------------------
The second protocol uses discrete logarithms of points (rather than hash preimages) in order to eliminate this loss of privacy.

Let u_i denote the Upfront Fee paid to node i if the payment reaches node i.
Let f_i denote the amount of Upfront Fees staked by node i-1 for potential transfer to node i.

For each i, 0 < i <= n, node i-1 includes the Upfront stake amount f_i and a list of points P_{i,i}, P_{i,i+1} ... P_{i,n} in the update_add_htlc packet that it sends to node i.

Nodes i-1 and i create the following contract:
  * node i-1 stakes f_i Upfront funds for potential transfer to node i, and
  * if node i includes a discrete log p_{i,k} in the update_fulfill_htlc or update_fail_htlc packet it sends to node i-1, where:
    * p_{i,k}*G = P_{i,k} for some k <= n,
    * t_{i,k} is the value of the 32 most significant bits of p_{i,k}, and
    * t_{i,k} <= f_i,
then node i-1 will transfer t_{i,k} Upfront funds to node i.

The onion payload for node i provides two types of secret.

First, it provides the discrete log p_{i,i} where p_{i,i}*G = P_{i,i} and the most significant 32 bits of p_{i,i} equal u_i.
Therefore, p_{i,i} is the discrete log that node i gives node i-1 if the payment stops at node i.

Second, if i < n the onion payload provides delta values d_{i,k}, i < k <= n, which are the differences between the discrete log of P_{i,k} and the discrete log of P_{i+1,k} (where P_{i+1,k} is a point in the list of points in the update_add_htlc packet sent to node i+1).
While the onion payload does not allow node i to determine the discrete log of P_{i,k}, it does allow node i to determine that P_{i,k} = P_{i+1,k} + d_{i,k}*G.
Therefore, d_{i,k} is the difference between the discrete log of P_{i+1,k} and the discrete log of P_{i,k}, so if node i+1 gives node i the discrete log of P_{i+1,k}, node i will be able to calculate the discrete log of P_{i,k}.
Furthermore, the most significant 32 bits of d_{i,k} are a lower bound on the difference between t_{i,k} (the Upfront transfer to node i from node i-1) and t_{i+1,k} (the Upfront transfer from node i to node i+1) [5].
As a result, node i can verify that it will receive a sufficiently large net Upfront transfer of t_{i,k} - t_{i+1,k} by examining the most significant 32 bits of each d_{i,k}.

When the payment stops at some node k, node k includes the discrete log p_{k,k} in the update_fulfill_htlc or update_fail_htlc packet that it sends to its upstream partner.
Each node i, 0 < i < k, then takes the discrete log p_{i+1,k} that it received from its downstream partner and determines k using the formula p_{i+1,k}*G = P_{i+1,k}.
Next, each node i, 0 < i < k, calculates the discrete log p_{i,k} to include in the update_fulfill_htlc or update_fail_htlc packet that it sends to its upstream partner using the formula p_{i,k} = p_{i+1,k} + d_{i,k} [5].
Nodes i-1 and i then update their channel state to reflect the resolution of the HTLC and the division of the Upfront funds.
Specifically, nodes i-1 and i calculate t_{i,k} equals the 32 most significant bits of p_{i,k}, verify that t_{i,k} <= f_i, transfer t_{i,k} Upfront funds from the burn output to node i, refund the remaining staked Upfront funds to node i-1, and refund the matching funds to both nodes.

While the above protocol allows the nodes to calculate the correct division of Upfront Fees, including the delta values d_{i,k}, i < k <= n, in the onion payload would force the onion size to grow quadratically in n.
Instead, the actual protocol uses hash functions to eliminate this quadratic growth.
The full details are given in the paper [5].

Calculating Hold Fees
=====================
Hold Fees are paid by nodes to all upstream nodes that are delayed beyond the payment's hold grace period.
The downstream node in each channel stakes the maximum Hold Fee it could transfer by putting those funds in the burn output.
In addition, both nodes add matching funds to the burn output that are a fixed fraction of the staked Hold Fee funds.
If the downstream node provides an update_fulfill_htlc or update_fail_htlc packet to its upstream partner after the payment's hold grace period expires, the downstream node makes a transfer of Hold Fee funds that is the product of the downstream node's hold_usat_per_hour value and the number of hours by which the payment was delayed, regardless of whether or not the downstream node was the cause of the delay.

The update_add_htlc packet sent to each node i includes:
  * the grace period expiry g_i and
  * the stake amount h_i.
  
When it receives an update_add_htlc packet, node i calculates h_i divided by the maximum delay possible at node i (which is the difference between the expiry of the HTLC offered to node i and the grace period expiry g_i) to obtain y_i (which is the rate at which node i transfers Hold Fee funds to its upstream partner).
If i < n, node i also calculates y_{i+1} and verifies that y_{i+1} - y_i (which is the rate at which node i is paid a net Hold Fee for the cost of delaying its use of its capital) is sufficiently large.
Node i also verifies that its hold grace period, g_i - g_{i+1}, is sufficiently long.

The Hold funds that are staked in the burn output by node i have to be divided according to how quickly the HTLC offered to node i is resolved.
If that HTLC is resolved by g_i, then all of those funds are refunded to node i.
However, if the HTLC is resolved after g_i, node i calculates the length of the delay, d_i, and transfers d_i * y_i Hold funds to its upstream partner and refunds the remaining staked Hold funds to itself.
In addition, the matching funds are returned to both nodes.
The full details are given in the paper [5].

Risk Of Burning Funds
=====================
There are four ways in which a party that follows the protocol can be forced to burn funds.

First, one may have to put a Commitment transaction with a burn output on-chain in order to resolve an HTLC.
For example, this can happen when using the current Lightning protocol.
However, this possibility can be eliminated by using a separate Payment transaction [5], a factory-optimized channel protocol [5][3], or the Off-chain Payment Resolution protocol [5][4].

Second, if one's channel partner fails and cannot update the channel state for a very long time (e.g., months), one may have to close the channel unilaterally by putting their channel state transactions with a burn output on-chain.
The amount of time one is willing to wait is based on the likelihood of their partner becoming responsive versus the cost of the channel's capital, and is independent of the lengths of the HTLCs' expiries.

Third, a dishonest channel partner can put channel state transactions with a burn output on-chain in order to grief their partner (while also griefing themselves).
In this case, the griefer loses (at least) their matching funds, which are set high enough to discourage most or all such griefing attacks.
Specifically, if each party devotes a fraction m of the amount staked for Upfront and Hold Fees as matching funds, the griefer must lose at least m/(1+m) as many funds as their partner loses.

Fourth, if both parties are honest but they fail in such a way that they don't agree on the determination of Upfront and Hold Fees, they will be forced to burn those fees and their matching funds.
Fortunately, there are many techniques that reduce the likelihood of such failures, including:
  * adding buffers for clock skew and maximum packet transmission times when determining when the downstream partner sent the payment's update_fulfill_htlc or update_fail_htlc packet,
  * synchronizing the channel partners' clocks by exchanging frequent time stamp messages,
  * maintaining a nonvolatile log of when all update_fulfill_htlc or update_fail_htlc packets are sent or received, and
  * sending multiple encrypted copies of update_fulfill_htlc or update_fail_htlc packets via independent third-parties.
  
If all of the multiple copies are sent but fail to reach the offerer, it is likely there was either a complete loss of communication by the offeree or a failure of the offerer.
These cases can be differentiated by looking at the nonvolatile logs for the nodes' other channels.
For example, if a node has 20 channels with different partners and stops receiving time-stamped messages from just one of those partners, they can conclude that the failure was likely caused by that partner.
On the other hand, if the node stops receiving messages on all 20 channels, the node can conclude that it's likely the cause of the failures (and as a result, it should agree with its partner's division of the burn output funds).

Finally, a party can lose funds if they can be psychologically manipulated to allow their partner to steal from them.
For example, if Alice and Bob should each receive two thousand sats from the burn output, Bob could refuse to update the channel state unless he gets three thousand sats from the burn output (and Alice could agree in order to at least get the remaining one thousand sats).
This type of bullying attack can be prevented by ensuring that all channel updates are performed automatically, rather than under human control.

Conclusions
===========
This post presents protocols for assigning and collecting fees for Lightning services.
These protocols force the parties to pay for all significant costs that they impose on others and the fees they charge cannot be stolen by self-interested parties.
It's hoped these protocols will be used to both reduce spam and improve the efficiency of the Lightning Network.

References
==========

[1] Jager, "Hold fees: 402 Payment Required for Lightning itself", https://www.mail-archive.com/lightning-dev@lists.linuxfoundation.org/msg02080.html

[2] Jager, "Hold fee rates as DoS protection (channel spamming and jamming)", https://www.mail-archive.com/lightning-dev@lists.linuxfoundation.org/msg02212.html

[3] Law, "Factory Optimized Protocols For Lightning", https://github.com/JohnLaw2/ln-factory-optimized

[4] Law, "A Fast, Scalable Protocol For Resolving Lightning Payments", https://github.com/JohnLaw2/ln-opr

[5] Law, "Fee-Based Spam Prevention For Lightning", https://github.com/JohnLaw2/ln-spam-prevention

[6] Riard, "Unjamming lightning (new research paper)", https://www.mail-archive.com/lightning-dev@lists.linuxfoundation.org/msg02996.html

[7] Riard and Naumenko, "Lightning Jamming Book", https://jamming-dev.github.io/book/

[8] Teinturier, "Re: Hold fees: 402 Payment Required for Lightning itself", https://www.mail-archive.com/lightning-dev@lists.linuxfoundation.org/msg02106.html

[9] Teinturier, "Spam Prevention", https://github.com/t-bast/lightning-docs/blob/398a1b78250f564f7c86a414810f7e87e5af23ba/spam-prevention.md

-------------------------

harding | 2025-03-18 15:11:31 UTC | #2

I don't think this proposal addresses the main criticism of past proposals.  [Quoting](https://diyhpl.us/~bryan/irc/bitcoin/bitcoin-dev/linuxfoundation-pipermail/lightning-dev/2023-July/004033.txt) @ClaraShk:

> [T]he ?holy grail? is indeed charging fees as a
function of the time the HTLC was held. As for now, we are not aware of a
reasonable way to do this. There is no universal clock, and there is no way
for me to prove that a message was sent to you, and you decided to pretend
you didn't. It can easily happen that the fee for a two-week unresolved
HTLC is higher than the fee for a quickly resolving one.

Specifically, what stops a downstream node from ignoring an incoming HTLC and thus forcing its upstream peer to pay hold fees until they broadcast a force close transaction and get it confirmed to a certain depth?

For example, Bob forwards a half-signed HTLC to Mallory (with burn output).  Mallory doesn't reply with either acceptance or rejection.  Now Bob can't safely cancel the HTLC unless he force closes the channel.  Bob will probably need to pay onchain transaction fees to force close the channel, so he'll be incentivized to wait a bit and to use a non-urgent fee estimation---but until the channel confirms to a depth he finds sufficient to prevent double spending, he'll be liable for hold fees.

In this situation, Bob gains an upfront fee (presumably small compared to 6+ blocks of hold fees), he pays hold fees to his upstream node (which might be controlled by Mallory), and he loses onchain transaction fees to close the channel.  He does not earn a success fee.

A more specific example includes two nodes controlled by Mallory (M0, M1) and Bob's node.  M0 forwards funds through Bob to M1.  M0 pays Bob an upfront fee, Bob pays M0 a hold fee that's greater than than the upfront fee, and Bob pays the onchain transaction fee to force close.

M1 may have had to pay an onchain transaction fee to open the channel with Bob, but that could have been inexpensive compared to amount of money M0 will earn from several blocks of hold fees---and potentially _very_ inexpensive compared to Bob's total losses as victim of this attack.

I'll also note that any sort of hold fees seems fraught if LSPs continue offering [JIT channels](https://bitcoinops.org/en/topics/jit-channels/).  Mallory can forward a payment through an LSP to a fake customer, who neither accepts nor rejects the HTLC.  The LSP is liable for paying hold fees to Mallory until they can double spend the channel opening transaction and it confirms to a sufficient depth.  Mallory's only cost in this case is upfront fee.

-------------------------

