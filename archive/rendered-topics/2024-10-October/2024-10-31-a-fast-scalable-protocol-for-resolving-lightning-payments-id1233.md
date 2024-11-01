# A Fast, Scalable Protocol For Resolving Lightning Payments

JohnLaw | 2024-10-31 22:50:15 UTC | #1


The security of all known Lightning payment protocols breaks down when the incremental cost of resolving a payment on-chain exceeds the value of the payment.

For example, if Alice uses the current Lightning protocol to offer an HTLC to Bob, and Bob fulfills the HTLC by providing the required hash preimage before its expiry, Alice is supposed to update the channel state off-chain to reflect the payment of the HTLC to Bob.
However, if Alice fails to do so, Bob's only recourse is to put an HTLC-success transaction on-chain.
If the cost of putting the HTLC-success transaction on-chain exceeds the value of the payment, resolving the HTLC on-chain is more costly for Bob than allowing Alice to claim the payment.
Therefore, Alice is incentivized to violate the protocol by not giving the payment's funds to Bob off-chain, as she will receive those funds unless Bob acts against his immediate self-interest.

This post presents a new Lightning payment resolution protocol, called the Off-chain Payment Resolution (OPR) protocol, that has the following properties:
1) all payments are resolved correctly, provided both parties are self-interested and able to implement the protocol, regardless of how small the payment is,
2) all payments are resolved within seconds (as compared to hours for the current protocol),
3) there is never a need for a party to go on-chain to resolve a payment (thus greatly improving scalability), and
4) casual users can send and receive payments without having a watchtower service (and without maintaining high availability).

As a result, the OPR protocol appears to be well-suited to supporting small, everyday Lightning payments.
In addition, the OPR protocol can be used for large payments, either directly or by dividing the large payment into many smaller ones.

The OPR protocol has weaker security than a trust-free protocol (in which one cannot lose funds if they follow the protocol), but stronger than a trust-based protocol (in which one's funds can be stolen).
The OPR protocol is a *griefer-penalized* protocol, as an honest party can be made to lose certain funds by a dishonest channel partner who griefs them.
However, the griefer also has to lose a comparable amount of funds.
As a result, an honest party who only creates channels with self-interested parties will never be subject to a griefing attack.

A more complete description, including figures, is given in a paper [14].

Security Of The Current Lightning Protocol
==========================================

As noted above, with the current Lightning protocol an offeree of an HTLC may be forced to put an HTLC-success transaction on-chain.
The HTLC-success transaction requires about 702 vbytes [3], and the additional input in the transaction that spends the HTLC-sucess transaction's output requires about 316 vbytes, for a total of about 1018 incremental vbytes to claim a payment on-chain.
Assuming $50k per BTC and the historically common fee rates of 10 sat/vbyte to 100 sat/vbyte [17], the incremental cost for claiming a payment on-chain is in the range of $5 to $50.

Of course, even if the incremental cost of claiming a payment exceeds the value of the payment, that doesn't mean that all Lightning nodes will try to steal the payment.
For example, the expected value of future routing fees could exceed the value of the currently-outstanding small payments, thus preventing thefts.
Also, there are significant software engineering barriers to stealing small payments, as the offeree of an HTLC *will* choose to lose funds by securing a small payment on-chain if the offeree is running standard software that implements the Lightning protocol.

On the other hand, there are situations where the current Lightning protocol could lead to an attempt to steal funds.
For example, if Alice wants to close a channel with Bob, she could create one large payment and very many small payments for which Alice offers Bob HTLCs.
Next, Alice stops responding to Bob's messages after receiving the HTLCs' hash preimages.
As a result, Bob will put his Commitment transaction and the large payment's HTLC-success transaction on-chain.
In addition, Bob must either lose funds by claiming all of the small payments on-chain, or forfeit those payments to Alice.

This attack is costless for Alice, has at least some possibility of allowing her to steal funds, appears to be due to an involuntary failure (rather than malicious behavior), and is performed when she no longer needs to maintain her reputation with Bob in order to obtain future routing fees.

Off-chain Payment Resolution (OPR) Protocol
===========================================

The OPR protocol operates in conjunction with a channel protocol that creates a series of signed off-chain transactions reflecting the current channel state.
Old channel states can be replaced by a new channel state via a revocation key (as in the current Lightning protocol [2]) or a per-commitment key (as in the Tunable Penalty [9] or Fully-Factory-Optimized protocol [10]).
The details of how to maintain the channel state are presented in the paper [14].

When the channel is created, each party gets signed transactions with three outputs, one of which pays to each party, with the third one being a *burn output*.
The burn output pays to anyone after a 20-year delay (so the burn output will be claimed by a miner 20 years in the future).
Each party must contribute a fixed amount (called "base funds") to the burn output.

In order to route a new payment, an HTLC is created for the amount of the payment plus routing fees.
Each party gets new channel state transactions where the value of the HTLC is moved from the offerer's output to the burn output.
In addition, each party adds a fixed fraction of the value of the HTLC (called "matching funds") to the burn output.

As in the current Lightning protocol, the HTLC pays to the offeree only if the offeree provides the offerer with a hash preimage before the HTLC's expiry.
However, the expiry is set at most seconds in the future, based on the parties' htlc_expiry_delta_msec and min_final_expiry_delta_msec parameters.
These parameters are set to provide enough time to resolve the HTLC off-chain by receiving an update_fulfill_htlc or update_fail_htlc message, signing updated channel state transactions, and propagating the payment resolution to the upstream channel.
These parameters also include buffers for communication and computation delays caused by unusually heavy traffic, but they *do not* include time for putting channel state transactions on-chain.

If an HTLC is resolved successfully before its expiry, the HTLC's funds are moved to the offeree's output and both parties' matching funds for the HTLC are returned to them.
If an HTLC fails, the HTLC's funds are moved to the offerer's output and both parties' matching funds for the HTLC are returned to them.

As long as both parties agree on whether the HTLC succeeded or failed, they will agree on the new channel state transactions that resolve the HTLC and they will both get back their matching funds for the HTLC.
Finally, if both parties agree on the resolution of all HTLCs and they want to close the channel, they can create a Cooperative Close transaction that returns their base funds to them and eliminates the burn output.

Burned Funds
============

There are three ways in which a party that follows the OPR protocol can be forced to burn funds.

First, a dishonest channel partner can put channel state transactions with a burn output on-chain in order to grief their partner (while also griefing themselves).
In this case, the griefer loses (at least) their base and matching funds, which are set high enough to discourage most or all such griefing attacks.
Specifically, if each party devotes a fraction m of the HTLC amount as matching funds, the griefer must lose at least m/(1+m) as many funds as their partner loses (and at least their base funds).

Second, if an honest party's channel partner completely fails and can't update the channel state for a very long time (e.g., months), the honest party can be forced to close the channel unilaterally by putting their channel state transactions with a burn output on-chain.
The amount of time an honest party is willing to wait is based on the likelihood of their partner becoming responsive versus the cost of the channel's capital, and is independent of the lengths of the HTLCs' expiries.

Third, if both parties are honest but they fail in such a way that they don't agree on the resolution of an HTLC, they will be forced to burn the HTLC's funds and their base and matching funds.
Fortunately, there are many techniques that reduce the likelihood of such failures.

In order to determine if an HTLC was resolved successfully, a node has to determine if the required hash preimage was provided before the HTLC's expiry.
All hash preimage messages can include a time stamp recording when they were sent, and each node can keep a time-stamped nonvolatile log of each hash preimage that it sends or receives.
This log can be used to determine the result of an HTLC, even if the node crashes when the HTLC was being resolved.
Channel partners can keep their clocks synchronized by exchanging frequent time stamp messages, and the htlc_expiry_delta_msec parameters can include a buffer for clock skew.

The greatest challenge in getting agreement on whether or not an HTLC was resolved successfully occurs when the offeree sends the hash preimage before the expiry, but the offerer doesn't receive the preimage until after the expiry.
This case can be made less frequent by calculating the maximum communication latency L between channel partners and never sending a hash preimage less than L before the HTLC's expiry,

Also, multiple encrypted copies of hash preimage messages can be sent using different communication paths.
If all of these multiple copies are sent but fail to reach the offerer before the HTLC's expiry, it's likely there was either a complete loss of communication by the offeree or a failure of the offerer.
These cases can be differentiated by looking at the nonvolatile logs for the nodes' other channels.
For example, if a node has 20 channels with different partners and stops receiving time-stamped messages from just one of those partners, they can conclude that the failure was likely caused by that partner.
On the other hand, if the node stops receiving time-stamped messages on all 20 channels, the node can conclude that it is likely the cause of the failures.

HTLC Failures
=============

In addition to the risk of burning funds, a routing node can lose funds if it causes an HTLC to fail.

If a node failure prevents it from fulfilling an upstream HTLC, the node will lose the value of the HTLC because it's still liable for paying the corresponding downstream HTLC.
The risk of losing HTLC funds by having to pay a downstream HTLC without receiving an upstream HTLC exists in the current Lightning protocol.
However, the OPR protocol greatly increases this risk because it commits the node to delaying the resolution of the HTLC by at most seconds, as opposed to hours in the current Lightning protocol.

This risk is covered by Lightning routing fees, so it's worth quantifying the risk in order to determine whether or not those fees will be prohibitive.
If we assume the average incremental latency required to resolve an HTLC is 100 milliseconds and the average payment traverses 11 hops, each HTLC will be resolved an average of 600 milliseconds after it's created.
If each node fails (due to a crash, delay, or protocol error) randomly 10 times a day, thus causing it to lose the value of all unresolved HTLCs, that node will lose the value of one out of every 14,400 HTLCs.
Therefore, increasing the routing fee by 1/14,400 = 0.007% of the HTLC value per node (and thus 0.077% per payment) will cover the cost of node failures that cause HTLC failures.

As a result, it seems plausible that the OPR protocol could be used to implement fast off-chain payments without imposing excessive routing fees.

Bullying
========

Finally, a party can lose funds if they can be psychologically manipulated to allow their partner to steal from them.
For example, if Alice and Bob should each receive two million sats from the burn output, Bob could refuse to update the channel state unless he gets three million sats from the burn output (and Alice could agree in order to at least get the remaining one million sats).

This type of bullying attack can be prevented by ensuring that all channel updates are performed automatically, rather than under human control.

Scalability
===========

The OPR protocol has remarkable scaling properties.

First, the addition of an HTLC to a channel doesn't add any outputs to the channel state transactions.
As a result, even if the channel state is put on-chain, there's no incremental on-chain footprint due to the HTLC.
In addition, there's no protocol-defined upper bound on the number of active HTLCs per channel.

Second, the OPR protocol's resolution of an HTLC never requires any transactions to be put on-chain.
This is extremely important for scalability, as channel factories [1][4][11] and timeout-trees [7][13] have been proposed for opening and closing channels off-chain, and hierarchical channels [12] have been proposed for resizing channels off-chain.
As a result, without the OPR protocol, the on-chain resolution of HTLCs is likely to be the greatest limitation to Lightning's scalability.

Usability
=========

The OPR protocol's guaranteed resolution of a payment attempt within seconds makes it much more attractive to casual users than the current Lightning protocol, which could require waiting hours to find out that a payment attempt failed.
The fact that the payment's receiver never has to go on-chain to resolve the payment means that the OPR protocol supports one-shot receives [8].
Asynchronous trampoline payments (as described in Section 3.6 of [8]) can be used for payments between casual users that aren't online at the same time.

Finally, the OPR protocol can be combined with either revocation keys or per-commitment keys in a way that allows casual users to perform watchtower-free sends and receives.
The details are given in the paper [14].

Summary
=======

Combining the OPR protocol with per-commitment keys [14] yields the following properties:

* tunable penalties for attempting to put an old channel state on-chain [9],
* a watchtower service only requires storage that's logarithmic (as opposed to linear) in the number of channel states,
* if the channel is created in a channel factory or timeout-tree, the maximum latency of the payment is independent of the time required to put any channel factory or timeout-tree transactions on-chain [10],
* casual users can receive payments in a one-shot manner [8],
* casual users can send and receive payments without using a watchtower service (and without requiring them to be online more than once every three months) [8],
* arbitrarily small payments are routed securely, provided the parties routing the payment are self-interested,
* there is never any incremental on-chain footprint required to resolve a payment, and
* all payments are resolved within seconds.

Griefer-Penalized Protocols For Other Applications
==================================================

In the OPR protocol, both parties move funds to a burn output and they only receive those funds if they agree on how the funds should be divided.
The technique of devoting funds to a burn output can be used for any protocol where both parties automatically calculate the same division of funds (at least in the vast majority of cases) and both parties always receive a significant portion of those funds (in order to motivate cooperation).

Using a burn output in this manner has many advantages over using pre-signed transactions to determine fund divisions, including:
* elimination of the need for calculating and pre-signing transactions,
* the ability to resolve the division of funds in seconds,
* the ability to use computations that aren't supported by Bitcoin script when dividing funds, and
* the lack of any on-chain footprint when dividing funds.

For example, burn outputs could be used as an alternative to Discreet Log Contracts [5] for resolving smart contracts.
This could be attractive as it eliminates the need for an oracle that creates a Schnorr signature, as well as the need to create a large number of pre-signed contracts [6].

Another example is the creation of a fee-based protocol for reducing spam on Lightning.
Such a protocol will be presented in a future post.

Related Work
============

The OPR protocol's use of a burn output to obtain cooperation is similar to the use of security bonds.
However, the OPR protocol differs by having both partners contribute to the burn output, and by apportioning burn output funds to the partners based on a condition that they calculate off-chain (as opposed to the result of an on-chain Bitcoin script).
It's this difference that allows the OPR protocol to resolve payments so quickly and efficiently.

The idea of using a burn output to encourage cooperation between channel partners was suggested by Riard [15] in the context of preventing channel jamming.
However, Riard and Naumenko [16] left the creation of a protocol based on a burn output as future work.

Conclusions
===========

The ability to support small, everyday payments is essential if Lightning is to become a widely-used means of payment.
Although the OPR protocol has weaker security than the current Lightning protocol when making large payments, the comparison reverses with small payments.

The OPR protocol's ability to route small payments, coupled with its excellent scalability and usability, makes it an attractive candidate for supporting large numbers of casual Lightning users. 
The OPR protocol can also be used for large payments, either directly or by dividing the large payment into many smaller ones.

References
==========
[1] Burchert, Decker and Wattenhofer, "Scalable Funding of Bitcoin Micropayment Channel Networks", http://dx.doi.org/10.1098/rsos.180089
[2] BOLT 02 - Peer Protocol, https://github.com/lightning/bolts/blob/247e83d528a2a380e533e89f31918d7b0ce6a0c1/02-peer-protocol.md
[3] BOLT 03 - Transactions, https://github.com/lightning/bolts/blob/247e83d528a2a380e533e89f31918d7b0ce6a0c1/03-transactions.md
[4] Decker, Russell and Osuntokun, "eltoo: A Simple Layer2 Protocol for Bitcoin", https://blockstream.com/eltoo.pdf
[5] Dryja, "Discreet Log Contracts", https://adiabat.github.io/dlc.pdf
[6] Fournier, "CTV dramatically improves DLCs", https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-January/019808.html
[7] Law, "Scaling Bitcoin With Inherited IDs", https://github.com/JohnLaw2/btc-iids
[8] Law, "Watchtower-Free Lightning Channels For Casual Users", https://github.com/JohnLaw2/ln-watchtower-free
[9] Law, "Lightning Channels With Tunable Penalties", https://github.com/JohnLaw2/ln-tunable-penalties
[10] Law, "Factory-Optimized Channel Protocols For Lightning", https://github.com/JohnLaw2/ln-factory-optimized
[11] Law, "Efficient Factories For Lightning Channels", https://github.com/JohnLaw2/ln-efficient-factories
[12] Law, "Resizing Lightning Channels Off-Chain With Hierarchical Channels", https://github.com/JohnLaw2/ln-hierarchical-channels
[13] Law, "Scaling Lightning With Simple Covenants", https://github.com/JohnLaw2/ln-scaling-covenants
[14] Law, "A Fast, Scalable Protocol For Resolving Lightning Payments", https://github.com/JohnLaw2/ln-opr
[15] Riard, "Unjamming lightning (new research paper)", https://www.mail-archive.com/lightning-dev@lists.linuxfoundation.org/msg02996.html
[16] Riard and Naumenko, "Lightning Jamming Book", https://jamming-dev.github.io/book/
[17] Transaction Fees, https://blockchain.com/explorer/charts/transaction-fees

-------------------------

