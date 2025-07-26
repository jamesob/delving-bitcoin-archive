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

JohnLaw | 2025-03-21 21:08:20 UTC | #3

Dave,

Thanks for your thoughtful response.

You're right that there's a problem with the Hold Fee if the downstream node ignores an offered HTLC.

I believe the fix is to update the downstream node's Commitment transaction in two separate steps:
* increase the burn funds to include Upfront and Hold Fees (plus matching funds) for the offered HTLC, and
* add the HTLC output to the downstream node's Commitment transaction.

The downstream node performs the first step (increasing burn funds), and revokes its previous Commitment transaction, before the second step is performed.
It's safe for the downstream node to commit to the first update (with increased burn funds), as the this update includes the upstream node's matching funds (and its Upfront funds).

From the Bolt 02 spec, the current flow for adding an HTLC goes through the following states:

* pending on the receiver
* in the receiver's latest commitment transaction
* ... and the receiver's previous commitment transaction has been revoked, and the update is pending on the sender
* ... and in the sender's latest commitment transaction
* ... and the sender's previous commitment transaction has been revoked

In order to fix the bug you found, we need to change this to:

* pending on the receiver
* increased burn funds (but no HTLC output) in the receiver's latest commitment transaction
* ... and the receiver's previous commitment transaction has been revoked, and the update is pending on the sender
* ... and increased burn funds and HTLC output in the sender's latest commitment transaction
* ... and the sender's previous commitment transaction has been revoked
* ... and increased burn funds and HTLC output in the receiver's latest commitment transaction
* ... and the receiver's previous commitment transaction (with the increased burn amount but no HTLC output) has been revoked

Thus the upstream node sends an update_add_htlc packet and then a commitment_signed packet, where the new signatures cover the increased burn funds for the new HTLC (but not a new HTLC output).
Then, once the downstream node has sent a revoke_and_ack committing to the increased burn amount, the next commitment_signed transaction sent by the upstream node includes new increased burn funds and the new HTLC output.

Note that there's still just one update to the upstream node's Commitment transaction that adds both the increased burn funds and the HTLC output.
It's only the downstream node's Commitment transaction that needs to be updated in two steps.

The downstream node commits to paying the Hold fee for the new HTLC when it commits to the increased burn funds by revoking its previous Commitment transaction (that did not have the increased burn funds).

If the downstream node hasn't revoked its previous Commitment transaction before the new HTLC's grace period expires, the upstream node fails the HTLC by never updating the downstream node's Commitment transaction to include an output for the HTLC.
In this case, the upstream node can fail the HTLC with its upstream partner prior to the HTLC's grace period in that channel, so it doesn't have to pay any Hold Fees.

For example, Bob offers an HTLC to Mallory by signing a new Commitment transaction for Mallory that increases the burn funds for the new HTLC (but doesn't include a new HTLC output).
If Mallory fails to revoke his previous Commitment transaction prior to the HTLC's grace period in the channel with Bob, Bob fails the HTLC with Alice (Bob's upstream partner) prior to the HTLC's grace period in that channel.
As a result, Bob doesn't owe any Hold Fees to Alice.

Also, note that Bob doesn't run the risk of having to pay the HTLC to Mallory, as Bob still has a Commitment transaction without the HTLC (and without increased burn funds).
Thus, if Mallory never wakes up, Bob can close the channel by putting that transaction on-chain.
If Mallory does wake up before the channel is closed, Bob and Mallory resolve blame for the failure of the latest HTLC as described in the post and the paper (e.g., using nonvolalite logs, timestampted messages, etc.).

Please let me know if you agree that this fixes the bug.
If so, there's still work to do in defining the exact set of changes to the protocol for updating the channel state.
It's probably possible to define these changes while supporting asynchronous updates to both parties' Commitment transactions.
However, I believe Rusty proposed simplifying the protocol to eliminate races between updates to the parties' Commitment transations.
It should be much easier to implement the 2-stage update to the downstream party's Commitment transaction in such a race-free channel.

Regarding the "holy grail" of charging fees as a function of how long the HTLC was held, I believe using a burn output (with matching funds) provides the missing ingredient.
In particular, once both nodes have devoted funds to a burn output, they each individually want to determine the correct division of those funds so that they won't be burned.
This is what gets around the need to "prove" to one's partner that a message was sent.

I acknowledge that there are some limitations to using burn outputs, such as the risk of having one's partner fail completely, in which case the funds in the burn output are lost.
However, this proposal (with the fix described above) does charge fees that depend on how long an HTLC was held, and it does so in a way that prevents the theft of those fees.

-------------------------

harding | 2025-03-25 06:09:28 UTC | #4

Hi John,

AFAICT, your solution does eliminate the immediate issue.  However, it
seems to significantly increase link-layer latency, as described below.

> From the Bolt 02 spec [...]

I think we can simplify by ignoring `update_add_htlc` messages and
assuming that the commitment transaction is symmetric.  In that version
of the current protocol:

1. Alice sends Bob a new half-signed commitment transaction containing a
   new HTLC.

2. Bob replies with his half of the signature and his revocation secret
   for the previous commitment transaction.

3. Alice replies with her revocation secret for the previous commitment
   transaction.

Besides simplification, phrasing it this way showcases one of the
advantages of the current commitment protocol: even though the full
process requires 1.5 network round-trip time (RTT), Bob receives
everything he needs in Step 1 to be able to safely able to forward the
HTLC to the next hop after just 0.5 RTT.  So for an _n_-hop payment, the
best-case forwarding time can be said to be `n*0.5` RTT.

> In order to fix the bug you found, we need to change this to:

Using the simplification described above, I think this is what you're
proposing:

1. Alice sends Bob a new half-signed commitment transaction committing
funds from both of them to a burn output.

2. Bob replies his half of the signature for Step 1 and his revocation
secret for the previous commitment transaction.

3. Alice sends Bob her revocation secret for the previous commitment
transaction and a new half-signed commitment transaction containing
the new HTLC.

4. Bob replies with his half of the signature for Step 3 and his
revocation for the commitment transaction created in Step 1.

5. Alice replies with her revocation secret for the transaction created
in Step 1.

We immediately see that the proposed protocol increases full resolution
time to 2.5 RTT.  That's not concerning AFAIK, but what's notable is
that the critical path increases to 1.5 RTT.  That's potentially a 3x
slow down in LN payments as seen by the ultimate receiver.  Proof of
payment in both the current and proposed protocols takes an additional
`n*0.5` RTT, so the spender-observed slow down is 2x.

More generally, although I still feel uneasy about MAD-based OPR-style
protocols, I do clearly see the benefits of hold fees.
Although described here for spam mitigation (something for which other
solutions might work roughly as well), I think there could additionally
be significant demand for [not-immediately-settled
payments](https://bitcoinops.org/en/topics/hold-invoices/)---but that's
something we can only encourage if forwarding nodes are fairly
compensated for the delay.

-------------------------

JohnLaw | 2025-04-05 19:42:07 UTC | #5

Hi Dave,

Thanks for pointing out the benefits of time-dependent Hold Fees for not-immediately-settled payments.

Regarding the increase in latency, I agree that the bug fix (which increases the burn funds before it adds the HTLC output) adds one round-trip time (RTT).
However, I believe this is a 1.67x increase (rather than a 3x increase) as the downstream node has to wait for the revocation of the upstream node's commitment transaction that doesn't include the HTLC output (step 3 in your description of the current protocol and step 5 in your description of the protocol with the bug fix).
Thus, the latency goes from 1.5 RTTs to 2.5 RTTs.

Fortunately, I think it's possible to eliminate this latency penalty while implementing time-dependent Hold Fees with the bug fix.

The idea is to increase burn funds before adding the HTLC output (as in the high-latency bug fix), but to allow the downstream node to provide signatures for both of these updates consecutively (without a RTT in between).

If the downstream node committed to the increase burn funds (by revoking their previous commitment transaction) *during* the grace period, the upstream node can commit to the addition of the HTLC output (by revoking all earlier commitment transactions).
Then the upstream node gives the downstream node a signature for a commitment transaction with the HTLC output.
At this point, the downstream node is guaranteed to be able to get payment for the HTLC if they get the hash preimage in time, so the downstream node can offer the HTLC in the payment's next channel.
Note that the overall latency is just 1.5 RTTs.

On the other hand, if the downstream node committed to the increased burn funds *after* the grace period, the upstream node does not have to commit to the addition of the HTLC output.
Instead, it sends an update_remove_htlc packet to the downstream node, followed by a revocation of only the commitment transaction that precedes the one with increased burn funds.
As a result, the upstream node still has an unrevoked commitment transaction without the HTLC output that it can put on-chain if needed.
The downstream node responds with a signature for a new commitment transaction without the increased burn funds or HTLC output and the upstream node revokes all commitment transactions that precede this latest one.
This flow uses 2.5 RTTs, but it only occurs when there's a failure to propagate the payment within the grace period.
As a result, the payment fails without having to wait beyond the grace period.

This improved protocol requires the upstream node to have up to 3 current (signed and not revoked) transactions at a time.
Therefore, the rules for providing pc_points and pc_secrets have to be modified.
In particular, the current revoke_and_ack packet includes pc_secret_x and pc_point_(x+2) (for an arbitrary value of x).
To make this new low-latency spam-prevention protocol work, we have to change the revoke_and_ack packet to include pc_secret_x and pc_point_(x+3).
In addition, the funding_locked packet is changed to include pc_point_1 and pc_point_2 (rather than just pc_point_1).

The detailed protocol is presented below.
If no one finds any problems with it, I'll update the paper to include it.

Current Protocol
================

Terminology, Notation and Packet Definitions
--------------------------------------------
* Alice is sender
* Bob is receiver
* A_i is Alice's commitment transaction using pc_point_i
* B_j is Bob's commitment transaction using pc_point_j
* open_channel packet includes pc_point_0
* accept_channel packet includes pc_point_0
* funding_locked packet includes pc_point_1
* revoke_and_ack_x packet includes pc_secret_x and pc_point_(x+2)
* commitment_signed_x is commitment_signed message providing a signature for the commitment transaction which uses pc_point_x
* commitment transaction x is:
  - "signed" if commitment_signed_x has been received, "unsigned" otherwise
  - "revoked" if revoke_and_ack_x has been received
  - "current" if it is signed and not revoked

Flow for adding an HTLC
-----------------------
* pending on the receiver
* ... in the receiver's latest commitment transaction
* ... and the receiver's previous commitment transaction has been revoked, and the update is pending on the sender
* ... and in the sender's latest commitment transaction
* ... and the sender's previous commitment transaction has been revoked

Detailed Protocol
-----------------
* a1  update_add_htlc to receiver
  * HTLC output pending on the receiver
  * Bob has 1 current commitment transaction
    - HTLC output is not in Bob's current commitment transaction
* a2  commitment_signed_j to receiver
  * ... and the HTLC output is in the receiver's latest commitment transaction
  * Bob has 2 current commitment transactions: B_(j-1) and B_j
    - HTLC output is not in B_(j-1)
    - HTLC output is in B_j
* a3  revoke_and_ack_(j-1) to sender
  * ... and the receiver's previous commitment transaction has been revoked, and the HTLC output is pending on the sender
  * Bob has 1 current commitment transaction: B_j
    - HTLC output is in B_j
* a4  commitment_signed_i to sender
  * ... and the HTLC output is in the sender's latest commitment transaction
  * Alice has 2 current commitment transactions: A_(i-1) and A_i
    - HTLC output is not in A_(i-1)
    - HTLC output is in A_i
* a5  revoke_and_ack_(i-1) to receiver
  * ... and the sender's previous commitment transaction has been revoked
  * Alice has 1 current commitment transaction: A_i
    - HTLC output is in A_i

and eventually:

* a6  update_fulfill_htlc or update_fail_htlc to sender
  * ... and the HTLC resolution is pending on the sender
* a7  commitment_signed_x to sender
  * ... and the HTLC resolution is in the sender's latest commitment transaction
  * Alice has 2 current commitment transactions: A_(x-1) and A_x
    - HTLC output is in A_(x-1)
    - resolved HTLC (without HTLC output) is in A_x
* a8  revoke_and_ack_(x-1) to receiver
  * ... and the sender's previous commitment transaction has been revoked, and the HTLC resolution is pending on the receiver
  * Alice has 1 current commitment transactions: A_x
    - resolved HTLC (without HTLC output) is in A_x
a9  commitment_signed_y to receiver
  * ... and the HTLC resolution is in the receiver's latest commitment transaction
  * Bob has 2 current commitment transactions: B_(y-1) and B_y
    - HTLC output is in B_(y-1)
    - resolved HTLC (without HTLC output) is in B_y
* a10 revoke_and_ack_(y-1) to sender
  * ... and the receiver's previous commitment transaction has been revoked
  * Bob has 1 current commitment transactions: B_y
    - resolved HTLC (without HTLC output) is in B_y

HTLC Propagation Latency
------------------------
* receiver can send update_add_htlc in next channel after receiving a5
* 1.5 RTTs:
  * a1/a2
  * a3/a4
  * a5


Spam Prevention Protocol Without Increased Latency
==================================================

Terminology, Notation and Packet Definitions
--------------------------------------------
* Alice is sender
* Bob is receiver
* A_i is Alice's commitment transaction using pc_point_i
* B_j is Bob's commitment transaction using pc_point_j
* open_channel packet includes pc_point_0
* accept_channel packet includes pc_point_0
* funding_locked packet includes pc_point_1 and pc_point_2
* revoke_and_ack_x packet includes pc_secret_x and pc_point_(x+3)
* commitment_signed_x is commitment_signed message providing a signature for the commitment transaction which uses pc_point_x
* commitment transaction x is:
  - "signed" if commitment_signed_x has been received, "unsigned" otherwise
  - "revoked" if revoke_and_ack_x has been received
  - "current" if it is signed and not revoked

Case 1: Receiver commits to increased burn amount *during* grace period
======================================================================

Flow for adding an HTLC
-----------------------
* increased burn amount is pending on the receiver
* ... and the increased burn amount is in the receiver's latest commitment transaction
* ... and the receiver's previous commitment transaction has been revoked, and the increased burn amount is pending on the sender
* ... and the increased burn amount is in the sender's latest commitment transaction, and the HTLC output is pending on the sender
* ... and the increased burn amount and HTLC output are in the sender's latest commitment transaction
* ... and the sender's earliest current commitment transaction has been revoked
* ... and the sender's previous commitment transaction has been revoked, and the HTLC output is pending on the receiver
* ... and the HTLC output is in the receiver's latest commitment transaction
* ... and the receiver's previous commitment transaction has been revoked


Detailed Protocol
-----------------
* d1  update_add_htlc to receiver
  * increased burn amount pending on the receiver
  * Bob has 1 current commitment transaction
    - increased burn amount is not in Bob's current commitment transaction
* d2  commitment_signed_j to receiver
  * ... and the increased burn amount is in the receiver's latest commitment transaction
  * Bob has 2 current commitment transactions: B_(j-1) and B_j
    - increased burn amount is not in B_(j-1)
    - increased burn amount is in B_j
* d3  revoke_and_ack_(j-1) to sender during grace period
  * ... and the receiver's previous commitment transaction has been revoked, and the increased burn amount is pending on the sender
  * Bob has 1 current commitment transaction: B_j
    - increased burn amount is in B_j
* d4  commitment_signed_i to sender
  * ... and the increased burn amount is in the sender's latest commitment transaction, and the HTLC output is pending on the sender
  * Alice has 2 current commitment transactions: A_(i-1) and A_i
    - increased burn amount is not in A_(i-1)
    - increased burn amount is in A_i
* d5  commitment_signed_(i+1) to sender
  * ... and the increased burn amount and HTLC output are in the sender's latest commitment transaction
  * Alice has 3 current commitment transactions: A_(i-1), A_i and A_(i+1)
    - increased burn amount is not in A_(i-1)
    - increased burn amount is in A_i, HTLC output is not in A_i
    - increased burn amount and HTLC output are in A_(i+1)
* d6  revoke_and_ack_(i-1) to receiver
  * ... and the sender's earliest current commitment transaction has been revoked
  * Alice has 2 current commitment transaction: A_i and A_(i+1)
    - increased burn amount is in A_i, HTLC output is not in A_i
    - increased burn amount and HTLC output are in A_(i+1)
* d7  revoke_and_ack_i to receiver
  * ... and the sender's previous commitment transaction has been revoked, and the HTLC output is pending on the receiver
  * Alice has 1 current commitment transaction: A_(i+1)
    - increased burn amount and HTLC output are in A_(i+1)
* d8  commitment_signed_(j+1) to receiver
  * ... and the HTLC output is in the receiver's latest commitment transaction
  * Bob has 2 current commitment transactions: B_j and B_(j+1)
    - increased burn amount is in B_j, HTLC output is not in B_j
    - increased burn amount and HTLC output are in B_(j+1)
* d9  revoke_and_ack_j to sender
  * ... and the receiver's previous commitment transaction has been revoked
  * Bob has 1 current commitment transaction: B_(j+1)
    - increased burn amount and HTLC output are in B_(j+1)

and eventually:

* d10 update_fulfill_htlc or update_fail_htlc to sender
  * ... and the HTLC resolution is pending on the sender
* d11 commitment_signed_x to sender
  * ... and the HTLC resolution is in the sender's latest commitment transaction
  * Alice has 2 current commitment transactions: A_(x-1) and A_x
    - increased burn amount and HTLC output are in A_(x-1)
    - resolved HTLC (without increased burn amount or HTLC output) is in A_x
* d12 revoke_and_ack_(x-1) to receiver
  * ... and the sender's previous commitment transaction has been revoked, and the HTLC resolution is pending on the receiver
  * Alice has 1 current commitment transactions: A_x
    - resolved HTLC (without increased burn amount or HTLC output) is in A_x
* d13 commitment_signed_y to receiver
  * ... and the HTLC resolution is in the receiver's latest commitment transaction
  * Bob has 2 current commitment transactions: B_(y-1) and B_y
    - increased burn amount and HTLC output are in B_(y-1)
    - resolved HTLC (without increased burn amount or HTLC output) is in B_y
* d14 revoke_and_ack_(y-1) to sender
  * ... and the receiver's previous commitment transaction has been revoked
  * Bob has 1 current commitment transactions: B_y
    - resolved HTLC (without increased burn amount or HTLC output) is in B_y


HTLC Propagation Latency
------------------------
* receiver can send update_add_htlc in next channel after receiving d8
* 1.5 RTTs:
  * d1/d2
  * d3/d4/d5
  * d6/d7/d8


Case 2: Receiver commits to increased burn amount *after* grace period
=====================================================================

Flow for adding an HTLC
-----------------------
* increased burn amount is pending on the receiver
* ... and the increased burn amount is in the receiver's latest commitment transaction
* ... and the receiver's previous commitment transaction has been revoked, and the increased burn amount is pending on the sender
* ... and the increased burn amount is in the sender's latest commitment transaction, and the HTLC output is pending on the sender
* ... and the increased burn amount and HTLC output are in the sender's latest commitment transaction
* ... and the failed HTLC (without increased burn amount or HTLC output) is pending on the receiver
* ... and the sender's earliest current commitment transaction has been revoked
* ... and the failed HTLC (without increased burn amount or HTLC output) is in the receiver's latest commitment transaction
* ... and the receiver's previous commitment transaction has been revoked, and the failed HTLC (without increased burn amount or HTLC output) is pending on the sender
* ... and the failed HTLC (without increased burn amount or HTLC output) is in the sender's latest commitment transaction
* ... and the sender's earliest current commitment transaction has been revoked
* ... and the sender's previous commitment transaction has been revoked



Detailed Protocol
-----------------
* e1  update_add_htlc to receiver
  * increased burn amount pending on the receiver
  * Bob has 1 current commitment transaction
    - increased burn amount is not in Bob's current commitment transactions
* e2  commitment_signed_j to receiver
  * ... and the increased burn amount is in the receiver's latest commitment transaction
  * Bob has 2 current commitment transactions: B_(j-1) and B_j
    - increased burn amount is not in B_(j-1)
    - increased burn amount is in B_j
* e3  revoke_and_ack_(j-1) to sender after grace period
  * ... and the receiver's previous commitment transaction has been revoked, and the increased burn amount is pending on the sender
  * Bob has 1 current commitment transaction: B_j
    - increased burn amount is in B_j
* e4  commitment_signed_i to sender
  * ... and the increased burn amount is in the sender's latest commitment transaction, and the HTLC output is pending on the sender
  * Alice has 2 current commitment transactions: A_(i-1) and A_i
    - increased burn amount is not in A_(i-1)
    - increased burn amount is in A_i
* e5  commitment_signed_(i+1) to sender
  * ... and the increased burn amount and HTLC output are in the sender's latest commitment transaction
  * Alice has 3 current commitment transactions: A_(i-1), A_i and A_(i+1)
    - increased burn amount is not in A_(i-1)
    - increased burn amount is in A_i, HTLC output is not in A_i
    - increased burn amount and HTLC output are in A_(i+1)
* e6  update_remove_htlc to receiver
  * ... and the failed HTLC (without increased burn amount or HTLC output) is pending on the receiver
* e7  revoke_and_ack_(i-1) to receiver
  * ... and the sender's earliest current commitment transaction has been revoked
  * Alice has 2 current commitment transaction: A_i and A_(i+1)
    - increased burn amount is in A_i, HTLC output is not in A_i
    - increased burn amount and HTLC output are in A_(i+1)
* e8  commitment_signed_(j+1) to receiver
  * ... and the failed HTLC (without increased burn amount or HTLC output) is in the receiver's latest commitment transaction
  * Bob has 2 current commitment transactions: B_j and B_(j+1)
    - increased burn amount is in B_j, HTLC output is not in B_j
    - failed HTLC (without increased burn amount or HTLC output) is in B_(j+1)
* e9  revoke_and_ack_j to sender
  * ... and the receiver's previous commitment transaction has been revoked, and the failed HTLC (without increased burn amount or HTLC output) is pending on the sender
  * Bob has 1 current commitment transaction: B_(j+1)
    - failed HTLC (without increased burn amount or HTLC output) is in B_(j+1)
* e10 commitment_signed_(i+2) to sender
  * ... and the failed HTLC (without increased burn amount or HTLC output) is in the sender's latest commitment transaction
  * Alice has 3 current commitment transaction: A_i, A_(i+1) and A_(i+2)
    - increased burn amount is in A_i, HTLC output is not in A_i
    - increased burn amount and HTLC output are in A_(i+1)
    - failed HTLC (without increased burn amount or HTLC output) is in A_(i+2)
* e11 revoke_and_ack_i to receiver
  * ... and the sender's earliest current commitment transaction has been revoked
  * Alice has 2 current commitment transactions: A_(i+1) and A_(i+2)
    - increased burn amount and HTLC output are in A_(i+1)
    - failed HTLC (without increased burn amount or HTLC output) is in A_(i+2)
* e12 revoke_and_ack_(i+1) to receiver
  * ... and the sender's previous commitment transaction has been revoked
  * Alice has 1 current commitment transaction: A_(i+2)
    - failed HTLC (without increased burn amount or HTLC output) is in A_(i+2)


Allowed Flows For Spam Prevention Protocol With Low Latency
===========================================================

Terminology and Notation
------------------------
* an HTLC is "speculative" at the sender/offerer until the receiver/offeree has been given a signature for their corresponding HTLC output
* a state is "failed" if it includes an HTLC output for a speculative HTLC which cannot be used because the receiver failed to commit to the increased burn amount during the grace period
* a failed state will be indicated with an exclamation mark

* Configuration 1: 1 current state (not failed)
  * x

* Configuration 2: 2 current states (neither failed)
  * x
  * x+1

* Configuration 3: 3 current states (none failed)
  * x
  * x+1
  * x+2

* Configuration 4: 3 current states (only last one failed)
  * x
  * x+1
  * x+2!

* Configuration 5: 2 current states (only last one failed)
  * x
  * x+1!

* Configuration 6: 3 current states (only second one failed)
  * x
  * x+1!
  * x+2

* Configuration 7: 2 current states (only first one failed)
  * x!
  * x+1

Note: One's partner knows exactly which configuration one has, except they can't distinguish 3 from 4.

Allowed Flows:
-------------
* State update without adding a speculative HTLC
  * 1->2->1
* State update adding a successful speculative HTLC
  * 1->2->3->2->1
* State update adding a failed speculative HTLC
  * 1->2->4->5->6->7->1

-------------------------

ClaraShk | 2025-04-15 15:55:25 UTC | #6

Hi,

I’m trying to better understand the suggestion—could you clarify a couple of points?

1. Is the idea that each channel with capacity of $m$ sats would also lock an additional $m$ sats solely as collateral to mitigate jamming attacks? Do you have any estimates of the associated costs, such as the opportunity cost of locking up these funds?

2. Regarding channel closure and the distribution of the extra funds: as I understand it, both parties would need to agree, or else the funds are lost. Have you considered behavioral dynamics, such as those studied in experiments like the ultimatum game? These may suggest that real-world outcomes could diverge from the theoretically optimal ones.

I think a numerical example that explores different waiting times could be very helpful in understanding the tradeoffs.

-------------------------

JohnLaw | 2025-04-25 21:17:55 UTC | #7

[quote="ClaraShk, post:6, topic:1524"]
Is the idea that each channel with capacity of $m$ sats would also lock an additional $m$ sats solely as collateral to mitigate jamming attacks? Do you have any estimates of the associated costs, such as the opportunity cost of locking up these funds?
[/quote]

No, there is no need to devote am amount equal lto the channel capacity as collateral to mitigate jamming. That would be very inefficient!

The amount that one has to stake for Upfront and Hold fees is just the maximum amount of those fees that would be required for the currently-outstanding HTLCs (plus some frixed fraction in matching funds, where the amount of matching funds is a function of the maximum possible Upfront and Hold fees, not the channel capacity).

Yes I've definitely thought about the behavioral dynamics. The general principle is that one doesn't like to lose one's own funds, so one is generally unlikely to do so. The ultimatum game captures one of the few cases where people are sometimes willing to give up their own funds (or at least not maximize them), namely when they feel they are bing taken advantage of. In fact, that desire to "not be a sucker" is discussed in the related OPR paper. In that paper, the main way to prevent being taken advantage of is to rely on programmatic control. However, as the utlimatum game shows, even if one gets to make a human decision, allowing oneself to be ripped off if the one time when one may be willing to forego wealth. This is required to avoid a "bullying" attack, as described in the OPR paper.

Thanks for your suggestion to include a numeric example, I'll work one up.

-------------------------

JohnLaw | 2025-04-25 21:20:40 UTC | #8

I have uploaded a revised version of the paper (version 1.1) to:

https://github.com/JohnLaw2/ln-spam-prevention

It includes a full specification of the lower-latency version of the bug fix described in an earlier post.

-------------------------

JohnLaw | 2025-05-12 17:47:12 UTC | #9


Here's a numerical example of using Upfront, Hold and Success Fees for a 10-hop payment of 10,000 sats, where node 0 is the payment's source and node 10 is the payment's destination.  

It also includes a comparison to the current protocol.

FEE-BASED SPAM REDUCTION PROTOCOL  
=================================  

Assume:  
------  
* each node on the route uses the same parameters  
* each routing node charges 10 msats as the fixed part of the Upfront Fee to cover the costs of calculation, communication and allocation of a slot  
* each routing node charges 1 * 10^(-5) of the payment amount as the proportional part of the Upfront Fee to cover:  
  - the cost of allocating channel funds to the payment for the seconds that can be required for a non-delayed payment, and  
  - the risk of losing the payment amount (due to paying the downstream HTLC without being paid the upstream HTLC)  
* each node has a cltv_expiry_delta of 10 hours (600 blocks)  
* each node charges a Hold Fee at the rate of 2 * 10^(-5) / hour  
  - this corresponds to a 19% (compounded) annual rate of return (as 1.00002^(24*365) = 1.19)  
* each node charges 1 * 10^(-4) of their nonreimbursable Hold Fee stake contribution as part of the Upfront Fee  
  - this covers the risk that the node will be responsible for paying a Hold Fee due to a delay that it causes  
  - the value 1 * 10^(-4) corresponds to causing a 10-hour delay once every 10,000 payments  
    - note that a 10-hour delay is the maximum nonreimbursable delay given the cltv_expiry_delta value of 10 hours  
    - any delay beyond 10 hours must be caused by some downstream node and thus will be paid (reimbursed) by that downstream node  
* each node charges 1 * 10^(-4) of their Hold Fee stake contribution as part of the Upfront Fee  
  - this covers the risk of burning those funds due to a partner who fails to agree to the division of the burn funds (e.g., due to a permanent failure or griefing attack)  
* each node requires their partner devote (as matching funds) 1/4 of the base funds in the burn output  
  - this guarantees one's partner loses at least 1/5 of what one loses in a griefing attack  
  - this increases the size of the burn output by 50% (as each channel partner provides matching funds that are 1/4 of the base funds)  
* each routing node charges 90 msats as the fixed part of the Success Fee  
* each routing node charges 6 * 10^(-5) of the payment amount as the proportional part of the Success Fee  

Given the above assumptions:  
* each node charges a Hold Fee of (2 * 10^(-5) / hour) * (10,000 sats) = 2 * 10^(-1) sats / hour  
* each node i, 1 <= i <= 10, pays a nonreimbursable Hold Fee to the i upstream nodes at the rate of i * (2 * 10^(-1) sats / hour) for delays caused by node i  
  - thus node i pays a maximum of 2i sats in nonreimbursed Hold Fees if node i causes its maximum possible 10-hour delay  
* each routing node i, 1 <= i <= 9, charges a Succees Fee of 90 msats + 10,000 * 6 * 10^(-5) sats = 690 msats  

     	               		Upfront Fee     	    
     	Max            		charge for      	Hold Fee    
     	nonreimbursable		nonreimburable  	base stake    
     	Hold Fee       		Hold Fee risk   	contribution    
Node:	payment (sats):		(msats):        	(sats):    
---- 	---------------		--------------- 	------------  
 0   	   0           		   0.0          	 0  
 1   	   2	           	   0.2          	20  
 2   	   4	           	   0.4          	36  
 3   	   6           		   0.6          	48  
 4   	   8           		   0.8          	56  
 5   	  10           		   1.0          	60  
 6   	  12           		   1.2          	60  
 7   	  14           		   1.4          	56  
 8   	  16           		   1.6          	48  
 9   	  18           		   1.8          	36  
10   	  20           		   2.0          	20  

(For example, node 6 pays at most 2 sats to each of the 6 upstream nodes (nodes 0..5) for delays that node 6 causes.)  
(For example, node 6 charges 12 sats * 10^(-4) = 1.2 msats in additional Upfront Fees in order to cover the risk that node 6 will have to pay a Hold Fee for a delay that it causes.)  
(For example, node 6 stakes 5 * 6 * 2 = 60 sats for the maximum 2-sat Hold Fee paid by each of the 5 downstream nodes (nodes 6..10) to each of the 6 upstream nodes (nodes 0..5).)  

     	             	Total        	Upfront Fee  
     	Hold Fee     	Hold Fee     	charge for  
     	match stake  	stake	     	Hold Fee  
     	contribution 	contribution 	burn risk  
Node:	(sats):      	(sats):	     	(msats):  
---- 	------------ 	------------ 	---------------  
 0   	5            	5            	   0.5  
 1   	14           	34           	   3.4  
 2   	21           	57           	   5.7  
 3   	26           	74           	   7.4  
 4   	29           	85           	   8.5  
 5   	30           	90           	   9.0  
 6   	29           	89           	   8.9  
 7   	26           	82           	   8.2  
 8   	21           	69           	   6.9  
 9   	14           	50           	   5.0  
10   	5            	25           	   2.5  
(For example, node 6 matches 1/4 of its 60-sat Hold Fee base stake in the channel with node 5 and 1/4 of node 7's 56-sat Hold fee base stake in that channel.)  
(For example, node 6 contributes 60 sats of base funds in the channel with node 5, plus it contributes 15 sats of matching funds in that channel and 14 sats of matching funds in its channel with node 7.)  
(For example, node 6 charges 89 sats * 10^(-4) = 8.9 msats for the risk that its 89-sat Hold Fee stake contribution to burn outputs will be lost due to burning those funds.)  

     	Upfront Fee   
     	charge       	Total       	Upfront Fee  
     	unrelated to 	Upfront Fee 	base stake  
     	Hold Fees    	charge      	contribution  
Node:	(msats):     	(msats):    	(msats):  
---- 	------------ 	----------- 	------------  
 0   	   0         	   0.5      	1,066.5  
 1   	 110         	 113.6      	  952.9  
 2   	 110         	 116.1      	  836.8  
 3   	 110         	 118.0      	  718.8  
 4   	 110         	 119.3      	  599.5  
 5   	 110         	 120.0      	  479.5  
 6   	 110         	 120.1      	  359.4  
 7   	 110         	 119.6      	  239.8  
 8   	 110         	 118.5      	  121.3  
 9   	 110         	 116.8      	    4.5  
10   	   0         	   4.5      	    0.0  
(For example, node 6 charges a fixed Upfront fee of 10 msats and a proportional Upfront Fee of 10,000 * 10^(-5) = 0.1 sats = 100 msats.)  
(For example, node 6 charges 1.2 msats for the risk of paying a nonreimbursable Hold Fee, 8.9 msats for the risk of burning its contributions to Hold Fee stakes, and 110 msats unrelated to Hold Fees.)  
(For example, node 6 stakes base funds to cover Upfront Fee transfers for payments of 119.6 + 118.5 + 116.8 + 4.5 = 359.4 msats to nodes 7..10.)  

     	               	             	Total Hold +   
     	Upfront Fee     Upfront Fee  	Upfront Fee    
     	matching stake  total stake  	stake          
     	contribution   	contribution 	contribution   
Node:	(msats):       	(msats):     	(msats):     
---- 	-------------- 	------------  	-------------  
 0   	266.625        	1,333.1      	 6,333.1        
 1   	504.850        	1,457.8      	35,457.8        
 2   	447.425        	1,284.2      	58,284.2        
 3   	388.900        	1,107.7      	75,107.7        
 4   	329.575        	  929.1      	85,929.1        
 5   	269.750        	  749.3      	90,749.3        
 6   	209.725        	  569.1      	89,569.1        
 7   	149.800        	  389.6      	82,569.1        
 8   	 90.275        	  211.6      	69,211.6        
 9   	 31.450        	   36.0      	50,036.0        
10   	  1.125        	    1.1      	25,001.1        
(For example, node 6 contributes 479.5 / 4 = 119.875 msats Upfront matching funds in its channel with node 5 and 359.4 / 4 = 89.85 msats Upfront matching fund in its channel with node 7.)  
(For example, node 6 contributes 359.4 msats base funds and 209.725 msats matching funds for the Upfront Fee.)  
(For example, node 6 contributes 89 sats for Hold Fee stakes and 569.1 msats for Upfront Fee stakes.)  

        	            	            	burn output  
        	HTLC output 	burn output 	overhead     
Channel:	(msats):    	(msats):    	(percent):   
------- 	----------- 	----------- 	-----------  
0-1     	10,006,210  	 31,600     	0.32%  
1-2    		10,005,520  	 55,429     	0.55%  
2-3    		10,004,830  	 73,255     	0.73%  
3-4    		10,004,140  	 85,078     	0.85%  
4-5    		10,003,450  	 90,899     	0.91$  
5-6    		10,002,760  	 90,719     	0.91$  
6-7    		10,002,070  	 84,539     	0.85%  
7-8    		10,001,380  	 72,360     	0.72%  
8-9    		10,000,690  	 54,182     	0.54%  
9-10   		10,000,000  	 30,007     	0.30%  
(For example, the HTLC output in channel 5-6 contains the 10,000,000 msats for the payment amount and a Success Fee of 690 msats for each of the 4 downstream routing nodes (nodes 6..9).)  
(For example, the burn output in channel 5-6 contains 479.5 msats in Upfront Fee base stake contributions from node 5 and 60,000 msats in Hold Fee base stake contributions from node 6 = 60,479.5 msats base funds plus an additional 60,479.5 / 4 = 15,119.875 msats of matching funds from each of nodes 5 and 6.)  
(For example, the overhead for the burn output in channel 5-6 is 90,719 / 10,002,760 = 0.91%.)  

This completes the analysis of the calculation of the HTLC output and the burn output for this payment in each channel.  


COMPARISON WITH CURRENT PROTOCOL  
================================  

Now consider 4 scenarios for the execution of this payment using the Fee-Based Spam Prevention protocol and the current protocol.  

For the current protocol, assume:  
--------------------------------  
* each routing node charges a 100 msats as the fixed portion of the routing fee  
* each routing node charges 7.011 * 10^(-5) as the proportional part of the routing fee  
  - this value was chosen to match the charges for the Fee-Based Spam Prevention protocol above, minus the costs associated with losing funds by burning them (as those costs are unique to that protocol)  
  - however, this value is likely too small, as it ignores the larger on-chain fees that the current protocol can have (as shown in Scenario 3 below)  
* on-chain transactions cost 10 sats/vbyte  
* timing out an HTLC on-chain requires 388.75 vbytes  
  - Commitment transaction is 168 vbytes non-witness data + 222/4 vbytes witness data  
  - HTLC-timeout transaction is 94 vbytes non-witness data + 285/4 vbytes witness data  
  - cost of timing out an HTLC on-chain is 10,000 msats/vbyte * 388.75 vbytes = 3,887,500 msats  

Given these assumptions, with the current protocol the sender attempts to create the following HTLC outputs.  

        	HTLC output  
Channel:	(msats):  
------- 	-----------  
0-1     	10,007,210  
1-2    		10,006,409  
2-3    		10,005,608  
3-4    		10,004,807  
4-5    		10,004,006  
5-6    		10,003,204  
6-7    		10,002,403  
7-8    		10,001,602  
8-9    		10,000,801  
9-10   		10,000,000  
(For example, the HTLC output in channel 5-6 contains the 10,000,000 msats for the payment amount and a routing fee of 100 + 10,000,000 * 7.011 * 10^(-5) = 801.1  msats for each of the 4 downstream routing nodes (nodes 6..9).)  

Scenario 1: Successful payment with no delay  
--------------------------------------------  

     	Spam Prevention 	Current Protocol  
     	gain from       	gain from  
     	successful      	successful  
     	non-delayed     	non-delayed  
     	payment          	payment  
Node:	(msats):         	(msats):  
---- 	-------------   	----------------  
 0   	-10,007,276.5   	-10,007,209.9  
 1   	        803.6   	        801.1  
 2   	        806.1   	        801.1  
 3   	        808.0   	        801.1  
 4   	        809.3   	        801.1  
 5   	        810.0   	        801.1  
 6   	        810.1   	        801.1  
 7   	        809.6   	        801.1  
 8   	        808.5   	        801.1  
 9   	        806.8   	        801.1  
10   	 10,000,004.5   	 10,000,000.0  
(For example, node 6 charges an Upfront Fee of 120.1 msats and a Success Fee of 690 msats.)  
(For example, node 6 charges a fixed routing fee of 100 msats and a proportional routing fee of 10,000 sats * 7.011 * 10^(-5) = 701.1 msats.)  

Note that in Scenario 1 the two protocols have very similar results, with the fees for the Spam Prevention protocol being 0.9% higher overall (as 7276.5 / 7209.5 = 1.0092).  


Scenario 2: Successful payment delayed 1 hour at the destination  
----------------------------------------------------------------  

     	Spam Prevention 	Spam Prevention 	Spam Prevention  
     	gross gain from 	capital costs   	net gain from  
     	successful      	of successful   	successful  
     	payment         	payment         	payment  
     	delayed 1 hour  	delayed 1 hour   	delayed 1 hour  
     	at destination 		at destination  	at destination  
Node:	(msats):       		(msats):        	(msats):  
---- 	-------------- 		--------------- 	---------------  
 0   	-10,007,076.5  		200.3           	-10,007,276.8  
 1   	      1,003.6  		200.8           	        802.8  
 2   	      1,006.1  		201.3           	        804.8  
 3   	      1,008.0  		201.6           	        806.4  
 4   	      1,009.3  	  	201.8           	        807.5  
 5   	      1,010.0  	  	201.9           	        808.1  
 6   	      1,010.1  	  	201.8           	        808.3  
 7   	      1,009.6  	  	201.7           	        807.9  
 8   	      1,008.5  	  	201.4           	        807.1  
 9   	      1,006.8  	   	201.0           	        805.8  
10   	  9,998,004.5  	    	  0.5           	  9,998,004.0  
(For example, node 6 charges an Upfront Fee of 120.1 msats, a Hold Fee of 10,000,000 msats * (2 * 10^(-5) / hour) * 1 hour = 200 msats, and a Success Fee of 690 msats.)  
(For example, node 6 has capital costs of (10,002,070 + 89,569.1) msats * (2 * 10^(-5) / hour) * 1 hour = 201.8 msats.)  
(For example, node 6 receives 1,010.1 msats in fees and loses 201.8 msats in capital costs for a net gain of 808.3 msats.)  

     	Current Protocol 	Current Protocol 	Current Protocol  
     	gross gain from  	capital costs    	net gain from  
     	successful       	of successful    	successful  
     	payment          	payment          	payment  
     	delayed 1 hour   	delayed 1 hour   	delayed 1 hour  
     	at destination 	 	at destination   	at destination  
Node:	(msats):       	 	(msats):         	(msats):  
---- 	-------------- 	 	--------------- 	---------------  
 0   	-10,007,209.9  	 	200.1           	-10,007,410.0  
 1   	        801.1  	 	200.1           	        601.0  
 2   	        801.1  	 	200.1           	        601.0  
 3   	        801.1  	 	200.1           	        601.0  
 4   	        801.1  	 	200.1           	        601.0  
 5   	        801.1  	  	200.1           	        601.0  
 6   	        801.1  	  	200.0           	        601.1  
 7   	        801.1  	  	200.0           	        601.1  
 8   	        801.1  	  	200.0           	        601.1  
 9   	        801.1  	 	200.0           	        601.1  
10   	 10,000,000.0  	 	  0.0           	 10,000,000.0  
(For example, node 6 charges a fixed routing fee of 100 msats and a proportional routing fee of 10,000 sats * 7.011 * 10^(-5) = 701.1 msats.)  
(For example, node 6 has capital costs of 10,002,403 msats * (2 * 10^(-5) / hour) * 1 hour = 200.0 msats.)  
(For example, node 6 receives 801.1 msats in fees and loses 200.0 msats in capital costs for a net gain of 601.1 msats.)  

Note that in Scenario 2 the Spam Prevention protocol has a much more fair outcome, as the sender and the routing nodes are compensated for their capital costs by the node (namely the destination) that imposed those costs on them. This is an important scenario, as it is an example of a not-immediately-settled payment (as Dave Harding pointed out above).  

Scenario 3: Failed payment due to an unresponsive destination  
-------------------------------------------------------------  

     	Spam Prevention  
     	gain from  
     	failed payment  
     	due to  
     	unresponsive  
     	destination  
Node:	(msats):  
---- 	--------------  
 0   	-1,062.0       
 1   	   113.6       
 2   	   116.1       
 3   	   118.0       
 4   	   119.3       
 5   	   120.0       
 6   	   120.1       
 7   	   119.6       
 8   	   118.5       
 9   	   116.8       
10   	     0.0       
(For example, node 6 charges an Upfront Fee of 120.1 msats.)  

     	Current Protocol 	Current Protocol  	Current Protocol  
     	capital costs of 	on-chain fees for 	net gain from  
     	failed payment   	failed payment    	failed payment  
     	due to           	due to            	due to  
     	unresponsive     	unresponsive      	unresponsive 
     	destination      	destination       	destination  
Node:	(msats):         	(msats):          	(msats):  
---- 	---------------- 	----------------- 	---------------  
 0   	 2,001.4         	        0.0       	    -2,001.4   
 1   	 2,001.3         	        0.0       	    -2,001.3   
 2   	 2,001.1         	        0.0       	    -2,001.1   
 3   	 2,001.0         	        0.0       	    -2,001.0   
 4   	 2,000.8         	        0.0       	    -2,000.8   
 5   	 2,000.6         	        0.0       	    -2,000.6   
 6   	 2,000.5         	        0.0       	    -2,000.5   
 7   	 2,000.3         	        0.0       	    -2,000.3   
 8   	 2,000.2         	        0.0       	    -2,000.2  
 9   	 2,000.0          	3,887,500.0       	-3,889,500.0   
10   	     0.0         	        0.0       	         0.0   
(For example, node 6 has capital costs of 10,002,403 msats * (2 * 10^(-5) / hour) * 10 hours = 2,000.5 msats.)  
(For example, node 6 is able to resolve the failed payment off-chain with nodes 5 and 7 and therefore has no on-chain costs.)  
(For example, node 6 loses 2,000.5 msats in capital costs for a net loss of 2,000.5 msats.)  

Note that in Scenario 3 the Spam Prevention protocol has a far better outcome than the current protocol. In particular, with the Spam Prevention protocol the sender is notified of the failed payment within seconds and no node is required to devote their capital to the payment for more than a few seconds. In contrast, with the current protocol the sender has to wait 10 hours (the time required for node 9 to time out node 10 on-chain) before discovering that the payment has failed. Furthermore, with the current protocol nodes 0 through 9 have to devote their capital to this failed payment for those 10 hours and (even more significantly) node 9 has to pay millions of msats in on-chain fees in order to resolve the payment  

The protocols behave so differently because the Spam Prevention protocol allows node 9 to fail the payment off-chain and within seconds when the destination fails to increase the burn output for the payment. In contrast, the current protocol requires node 9 to go on-chain to resolve the payment. (Note that this difference between the protocols did not exist in the original paper; it was enabled by the bug fix that separates the increase in burn funds from the creation of the HTLC output.)  

Scenario 4: Failed payment due to insufficient funds at node 6  
--------------------------------------------------------------  

     	Spam Prevention 	Current Protocol  
     	gain from       	gain from  
     	failed payment  	failed payment  
     	due to          	due to  
     	insufficient     	insufficient  
     	funds at node 6 	funds at node 6  
Node:	(msats):         	(msats):  
----  	--------------- 	----------------  
 0   	-707.1          	          0.0  
 1   	 113.6          	          0.0  
 2   	 116.1          	          0.0  
 3   	 118.0          	          0.0  
 4   	 119.3          	          0.0  
 5   	 120.0          	          0.0  
 6   	 120.1          	          0.0  
 7   	   0.0          	          0.0  
 8   	   0.0          	          0.0  
 9   	   0.0          	          0.0  
10   	   0.0          	          0.0  
(For example, node 6 charges an Upfront Fee of 120.1 msats.)  
(For example, node 6 does not receive a routing fee because the payment was unsuccessful.)  

Note that in Scenario 4 the Spam Prevention protocol compensates nodes 1..6 for their costs, while the current protocol fails to do so.  

Hopefully the above analysis helps to clarify the details of the Spam Prevention protocol and to give some quantitative sense of how it operates. The comparison with the current protocol helps to demonstrate the advantages of the Spam Prevention protocol.  

Please let me know if anything isn't clear or if you feel the example parameters are unrealistic.

-------------------------

JohnLaw | 2025-05-22 21:20:03 UTC | #10

I've updated the associated paper to include the numerical example given above: 

https://github.com/JohnLaw2/ln-spam-prevention

The tables are better formatted there, so it may be easier to read.

-------------------------

ClaraShk | 2025-06-02 14:22:43 UTC | #11

Hi John,

Thanks for this!

Could you clarify some details?

When exactly is the hold fee charged, and what ensures that the necessary funds are available to cover it? 

If the funds are pre-locked, what’s the maximum amount that can be charged?

Is the hold fee calculated by amount, by slot, or both?

-------------------------

JohnLaw | 2025-06-30 00:57:12 UTC | #12

Hi Clara,

Thanks for your questions.

"When exactly is the hold fee charged, and what ensures that the necessary funds are available to cover it?"

The charging of the hold fee consists of 3 phases:  
* the channel state is updated by staking funds for the meximum possible hold fee charge:  
  - the downstream node moves funds from their output to the burn output that cover the maximum possible hold fee charge, and  
  - the upstream node moves matching funds (which are a fixed fraction of the funds staked by the downstream node) by moving them from their output to the burn output  
* both nodes calculate the correct hold fee charge when the HTLC is resolved with an update_fulfill_htlc or update_fail_htlc message (or the HTLC's expiry is reached), and  
* both nodes update the channel state by transferring the correct hold fee charge (plus the upfront node's matching funds) from the burn output to the upstream node and refunding the remainder of the downstream node's stake to the downstream node.  

The fact that the maximum possible hold fee charge is staked (placed in the burn output) ensures that the funds are available. The maximum possible hold fee charge is the product of the rate at which the hold fee is charged to the downstream node and how long the payment was delayed at this hop. The payment delay is the time from when the hop's hold grace period ends until an update_fulfill_htlc or update_fail_htlc message is sent, but it cannot extend past the HTLC's expiry (thus providing a maximum possible charge).

"If the funds are pre-locked, what’s the maximum amount that can be charged?"

They are pre-locked (by staking them in the burn output).

The maximum amount that can be charged is the product of the rate at which the downstream node is charged a hold fee and the maximum possible delay of the payment at this hop. The maximum possible delay of the payment at this hop is the time from the end of the payment's grace period to the HTLC's expiry.

Note that the downstream node is charged a hold fee at a rate that covers the cost of capital held by *all* of the nodes upstream of it (not just its upstream partner).

"Is the hold fee calculated by amount, by slot, or both?"

The hold fee charge covers the entire cost of capital for the time that capital was delayed. Thus, it depends on the amount of capital in all of the upstream nodes (rather than just the payment amount).

Of course, one could easily modify the protocol to use a different function for calculating hold fee charges. For example, a fixed lump sum charge could be added for any delay past the grace period. As another example, the charge could be based on a nonlinear function of the delay.

I chose the simple linear charge in order to capture the costs of capital. However, more complex charges (such as nonlinear ones) could cover the "free call option" cost when using the LN to make a Taproot Assets payment. This may be of interest given the recent news about using Taproot Assets for stable coins payments.

The costs associated with allocation of a slot are covered by Upfront fees and are paid by the sender.

If you want more detail, the calculation of hold fees is covered in Section 5 of the paper and the collection of hold fees is covered in Section 3.2 of the paper.

Please let me know if any of this isn't clear or if you have other questions.

-------------------------

ClaraShk | 2025-06-30 17:09:09 UTC | #13

To understand this better, I tried to work through a simple example where all payments are for 10k sats.

Let’s assume the node typically routes payments of around 10k sats, earning about 1 sat per transaction. Also, assume that 2 minutes is a reasonable time for an HTLC to resolve.

Now, suppose someone wants to lock a slot on the channel for a payment that could take up to 2 weeks to resolve (ignoring liquidity for now). Then, the opportunity cost for the slot is:
$
2*7*24*30*1 \approx  10,000
$ sats.

So to send 10k sats, they need to lock an extra 10k sats?

If I got this wrong, can you do the calculation given these parameters?

-------------------------

JohnLaw | 2025-07-01 22:03:02 UTC | #14

Hi Clara,

Even if we assume that a router typically routes 10k sat payments, that most HTLCs resolve in 2 minutes, and the router typically charges 1 sat per payment, we can't just use those numbers to determine that the router's cost of capital is 1 sat/10k sats per 2 minutes = 10^(-4) per 2 minutes = 3 * 10^(-3) per hour.

First, there are costs and risks that the router's 1 sat fee has to cover, including:  
* expected cost of on-chain fees to resolve the payment,  
* risk of losing the payment's funds due to a crash or failure,  
* computation costs, and  
* communication costs.

These costs and risks are paid for with the Upfront Fee and Success Fee. 

In addition, utilization of the router's capital isn't 100%, so we cant' just assume that a 2-week delay prevents the router from routing 10k other payments.

In fact, your cost of capital assumption is that 10k sats of capital returns 10k sats of profit every 2 weeks, thus doubling capital 26 times per year, resulting in a factor of 64 million annual return on capital. I don't believe that is a realistic risk-free rate of return.

In my example, I assumed a cost of capital of 2 * 10^(-5)/hour, which corresponds to a 19% compounded annual rate of return.  This sounds more realistic to me.

Finally, I should note that the Hold fee actually has to pay for the cost of capital for all of the capital that is being held. As a result, the 10th downstream node has to stake an amount that covers the maximum cost of capital for all 10 upstream nodes, thus increasing the amount staked by a factor of 10 at that node. All of this is included in my example.

Please let me know if any of this doesn't make sense or isn't clear.

-------------------------

ClaraShk | 2025-07-02 13:03:14 UTC | #15

Can you provide a number (and the calculation) for the example above? How much funds would need to be locked?

-------------------------

JohnLaw | 2025-07-13 00:20:28 UTC | #16

Hi Clara,

When you say "the example above", I guess you mean the example you gave where each routing node expects to be paid 1 sat every 2 minutes for routing 10k sat payments, has no risks and no expenses in doing so, and therefore expects to double there funds in a risk-free manner every 2 weeks. 

If this is what you mean, the answer is that the first routing node needs to lock 10k sats in order to be able to pay the maximum possible hold fee, and the 10th routing node has to lock 100k sats in order to pay the maximum possible hold fee. The calculations are exactly the ones you gave.

However, I was hoping that you would agree that being able to double your money every 2 weeks without expenses and without risk is not reasonable. As I noted, actual Lightning routing involves expenses and risks, and actual Lightning channels do not have 100% utilization, so even if a router charges 1 sat to route a 10k sat payment and such a payment typically takes 2 minutes, that in no way implies that they double their money every 2 weeks in a risk-free manner.

If we make the more reasonable assumption that routing nodes have a cost of capital of 19% per year, that translates to a 2 * 10^(-5) rate of return per hour. Therefore, the maximum hold fee that the first router could have to pay is 2 * 7 * 24 = 336 hours * 2 * 10^(-5) sats/hour * 10k sats = 67.2 sats. Thus, the first router would lock 67.2 sats to cover the maximum hold fee when routing a 10k sat payment. The 10th router would lock 10 times as much (that is, 672 sats) to route a 10k sat payment, as the 10th router could have to pay hold fees to the 10 upstream nodes that they could delay.

I hope this answers your question.

-------------------------

ClaraShk | 2025-07-14 13:26:38 UTC | #17

To make the conversation easier, let's focus on a simple case with just a few hops:
$$
M_1 \rightarrow A \rightarrow B \rightarrow M_2
$$
Where the attacker controlls $M_1$ and $ M_2 $, and wants to jam the channel $A - B$.

If there are 120 slots in the channel, it will cost an attacker 67.2*120 ~ 8k sats to jam it for two weeks?

Does the this change if the payments the attacker is sending is only 1 sat? Do they need to lock 0.8 sats?

-------------------------

JohnLaw | 2025-07-26 19:36:06 UTC | #18

Hi Clara,

"
If there are 120 slots in the channel, it will cost an attacker 67.2*120 ~ 8k sats to jam it for two weeks?

Does the this change if the payments the attacker is sending is only 1 sat? Do they need to lock 0.8 sats?
"

Yes, if there are just 120 slots, the Hold Fee charged by one channel for 120 10k-sat payments would just be 8k sats to jam for 2 weeks, and this drops to 0.8 sats for 1-sat payments.
(I believe BOLT 02 sets max_accepted_htlc to 483, so these values would be about 4 times as much, namely 32k sats for 10k-sat payments and 3.2 sats for 1-sat payments).

These would be low costs for a successful jamming attack.
However, the Hold Fee isn't the only fee that is charged.
There's also an Upfront Fee and a Succss Fee.

The Upfront Fee includes a charge for consuming a slot:

"
The Upfront Fee pays for:

    the computation and communication costs of routing the payment,. 
    the cost of allocating a Commitment transaction output (“slot”) to the payment,. 
    the cost of allocating channel funds to the payment for the seconds that can be required for a non-delayed payment,. 
    the risk of not fulfilling the HTLC with one’s upstream partner despite one’s downstream partner having fulfilled their HTLC,. 
    the risk of having to pay a Hold Fee (as described below), and. 
    the risk of burning funds (as described below).  
"

As long as the per-slot component of the Upfront Fee is set high enough, jamming the slots with 10k-sat (or 1-sat) payments will be expensive enough to compensate the routers for their stranded capital.

Still, you make an excellent point about the potential to jam a channel for 2 weeks by consuming all of the slots!

As you noted, the impact of running out of slots lasts the entire time the payment is being routed.
As a result, it makes sense to include a charge for consuming a slot as part of the Hold Fee (the per-slot charge within the Upfront Fee can be kept to cover the loss of the slot for a payment that isn't delayed).

The simplest way to do this is to make the Hold Fee a function of the time the payment is held and the maximum of:
1) the value of the payment, and
2) the capital in the channel divided by 483.

This approach forces the party that causes the delay to pay for the costs that they impose on others, which is the idea behind the fee-based approach.

An optimization may be to separate the channel's funds into two channels which are advertised separately. One of these channels would have a small amount of capital and wouuld be devoted to making small payments only. With this optimization, even if each channel charges a Hold Fee that depends on the max of the value of the payment and the capital in the channel divided by 483, the Hold Fee charged for small payments will be low.

Finally, another approach for routing small payments is to use the OPR protocol (https://github.com/JohnLaw2/ln-opr), as it eliminates the potential to delay the payment as well as all on-chain charges for resolving the payment.

-------------------------

