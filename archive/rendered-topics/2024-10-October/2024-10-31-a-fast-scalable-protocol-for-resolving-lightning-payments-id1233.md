# A Fast, Scalable Protocol For Resolving Lightning Payments

JohnLaw | 2024-11-03 01:56:53 UTC | #1


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

morehouse | 2024-11-04 23:59:52 UTC | #2

[quote="JohnLaw, post:1, topic:1233"]
For example, if Alice uses the current Lightning protocol to offer an HTLC to Bob, and Bob fulfills the HTLC by providing the required hash preimage before its expiry, Alice is supposed to update the channel state off-chain to reflect the payment of the HTLC to Bob. However, if Alice fails to do so, Bob’s only recourse is to put an HTLC-success transaction on-chain. If the cost of putting the HTLC-success transaction on-chain exceeds the value of the payment, resolving the HTLC on-chain is more costly for Bob than allowing Alice to claim the payment. Therefore, Alice is incentivized to violate the protocol by not giving the payment’s funds to Bob off-chain, as she will receive those funds unless Bob acts against his immediate self-interest.
[/quote]

All lightning implementations today will force close in this case even though it's not cost effective (though they might not claim the HTLC output on chain), and for good reason.  If Bob adopts a policy of "forgiving" small HTLCs in an attempt to stay off chain, it enables Alice to steal Bob's entire channel balance one small HTLC at a time.  Besides, if Alice is unresponsive to HTLC claims for 5+ hours (i.e. the typical CLTV delta), she's a poor channel partner even if she's honest.

OPR actually makes this problem worse.  Because Bob's cost of force closing is  higher with OPR (due to burned fees), there is a larger incentive for Bob to forgive small HTLCs that Alice refuses to resolve before expiry.

[quote="JohnLaw, post:1, topic:1233"]
In order to determine if an HTLC was resolved successfully, a node has to determine if the required hash preimage was provided before the HTLC’s expiry. All hash preimage messages can include a time stamp recording when they were sent, and each node can keep a time-stamped nonvolatile log of each hash preimage that it sends or receives. This log can be used to determine the result of an HTLC, even if the node crashes when the HTLC was being resolved. Channel partners can keep their clocks synchronized by exchanging frequent time stamp messages, and the htlc_expiry_delta_msec parameters can include a buffer for clock skew.
[/quote]

Seems tricky.  Since networks are inherently unreliable, can't an attacker easily lie about the actual time the message was sent?  How does a node distinguish between occasional network lag and a malicious peer?

Even if everyone is honest, I fear that the higher complexity of determining success/failure will lead to implementation bugs and more force closes, which are extra costly with OPR.

[quote="JohnLaw, post:1, topic:1233"]
# Usability

The OPR protocol’s guaranteed resolution of a payment attempt within seconds makes it much more attractive to casual users than the current Lightning protocol, which could require waiting hours to find out that a payment attempt failed.
[/quote]

Yes, this is good UX for casual users.

But the capital requirements of OPR are also quite bad UX for casual users.  To properly disincentivize cheating, the amount of funds each party contributes to the burn output must always be higher than the total value of outstanding HTLCs (otherwise users could profit from force closing without forwarding preimages).  This means that in order to *receive* payments, users must *already* have at least as much balance on their side of the channel as they want to receive.  Casual lightning users already get frustrated by the 1% reserve requirement, so this new requirement is sure to cause even more frustration.

-------------------------

harding | 2024-11-05 10:14:19 UTC | #3

I think OPR needs some sort of unconditional fees to work.

In the current LN protocol, the party that creates a payment only pays forwarding fees if the payment is successful.  Unsuccessful payments, and HTLCs designed to fail (called _probes_), cost their creator nothing.

However, if that model was adopted by OPR, then Mallory could send a constant stream of probes through Bob to random endpoints.  Each of those endpoints would fail the probes (because they wouldn't know the payment preimage) in the normal case, but it would occasionally be the case that there would be a problem downstream and a penalty payment would propagate back to Mallory.

Even with unconditional fees, OPR would seem to create an incentive for an attacker to create disruptions within the network.  Defending against such attacks could become a centralizing factor (e.g. in the same way that many popular websites use CloudFlare today because it's one of the few services big enough to survive cheap-to-perform DDoS attacks).

[quote="JohnLaw, post:1, topic:1233"]
If we assume the average incremental latency required to resolve an HTLC is 100 milliseconds and the average payment traverses 11 hops, each HTLC will be resolved an average of 600 milliseconds after it’s created. If each node fails (due to a crash, delay, or protocol error) randomly 10 times a day, thus causing it to lose the value of all unresolved HTLCs, that node will lose the value of one out of every 14,400 HTLCs. Therefore, increasing the routing fee by 1/14,400 = 0.007% of the HTLC value per node (and thus 0.077% per payment) will cover the cost of node failures that cause HTLC failures.
[/quote]

If the risk-free interest rate is 2% per annum, then we would expect the forwarding fee in a mature system to approach 0.00000004% per hop for a 600 ms average settlement time (`2 / (86400 / 0.6) / 365.25`), or 0.0000004% per payment assuming 11 hops. The OPR fee is 5 orders of magnitude higher.  https://1ml.com/statistics says the current median percentage fee (not factoring in the base fee) is "0.000072 sat/sat", or 0.0072% per hop, or 0.08% per payment assuming 11 hops, so OPR accidental routing failures is roughly expected to double the routing cost over the current system or 10,000x increase the cost over a (very theoretical) expected floor.

That does not count the costs of non-accidental failures (i.e. attacks) or the costs of preventing attacks.

[quote="JohnLaw, post:1, topic:1233"]
The security of all known Lightning payment protocols breaks down when the incremental cost of resolving a payment on-chain exceeds the value of the payment.
[/quote]

One theoretical solution, [proposed years ago](https://docs.google.com/presentation/d/1G4xchDGcO37DJ2lPC_XYyZIUkJc2khnLrCaZXgvDN0U/edit?pref=2&pli=1#slide=id.g85f425098_0_195), is for low-value payments to be encapsulated in higher-value outputs using probabilistic payments.  For example: when it would cost $10 in onchain fees to settle an HTLC, Alice offers Bob a $100 HTLC with an HTLC-Success transaction that gives him a 1% chance of being able to claim it onchain (the other 99% of the time, Alice reclaims the money).  Bob accepts that and forwards $1 to Carol (perhaps also using a probabilistic payment).  If the HTLC is settled offchain (either success or failure), then the $100 output is reallocated $1 to Bob and $99 to Alice.  If it goes onchain, then Bob gets his 1% chance at $100 (i.e., an expected value before fees of $1) which will cost him $10 in fees if he's successful; in the other 99% of the time, Alice gets her $100 back (expected value $99) by paying $10 in fees.

Probabilistic payments are possible today on Elements-based sidechains using `OP_DETERMINISTIC_RANDOM`.

It's my belief (possibly ill-informed) that there's been minimal work in this direction due to HTLC settlement cost griefing not being perceived as a major problem at present.

[quote="JohnLaw, post:1, topic:1233"]
all payments are resolved within seconds (as compared to hours for the current protocol),
[/quote]

[Opportunistic overpayments](https://bitcoinops.org/en/topics/redundant-overpayments/) also has this property for _almost_ all payments.  If, in your proposal, the burn output is expected to be a multiple of the maximal OPR HTLC amount, then the same amount of funds could be used instead for an refundable overpayment that would significantly increase the odds of rapid payment success.

Most discussion about overpayments that I'm aware of has preferred a PTLC-based mechanism, which is something that's still being developed for LN.

-------------------------

JohnLaw | 2024-11-09 23:21:48 UTC | #4

[quote="morehouse, post:2, topic:1233"]
All lightning implementations today will force close in this case even though it’s not cost effective (though they might not claim the HTLC output on chain), and for good reason. If Bob adopts a policy of “forgiving” small HTLCs in an attempt to stay off chain, it enables Alice to steal Bob’s entire channel balance one small HTLC at a time.
[/quote]

Yes, it makes sense that all lightning implementations prevent theft of small HTLCs, even when it's not cost effective.
As you point out, this is required to avoid becoming a sucker who can be slowly cheated out of their funds.
As a result, the lightning protocol is fairly secure, even for small payments.

It seems the main benefits of the OPR protocol are its speed and scalability.
I probably should have emphasized those benefits, rather than security for small payments, in the post and paper.
Thanks for your feedback.

[quote="morehouse, post:2, topic:1233"]
OPR actually makes this problem worse. Because Bob’s cost of force closing is higher with OPR (due to burned fees), there is a larger incentive for Bob to forgive small HTLCs that Alice refuses to resolve before expiry
[/quote]

Actually, with the OPR protocol, choosing to stay off-chain (rather than force closing) in no way leads to accepting the incorrect resolution of an HTLC.
That type of logic (having to go on-chain for resolve an HTLC in one's favor) applies to the current lightning protocol, but not to the OPR protocol.

[quote="morehouse, post:2, topic:1233"]
Since networks are inherently unreliable, can’t an attacker easily lie about the actual time the message was sent? How does a node distinguish between occasional network lag and a malicious peer?
[/quote]

Yes, an attacker can lie.
However, with the OPR protocol they have absolutely no incentive to do so (and will actually be penalized if they do so).

There's no reason for a peer to be malicious (unless they're griefing while self-griefing, which should be very rare).

The more relavent question is how both non-malicious peers can agree on whether or not an HTLC was resolved.
A number of techniques for doing this are presented in the post at the end of the Burned Funds section.

[quote="morehouse, post:2, topic:1233"]
Even if everyone is honest, I fear that the higher complexity of determining success/failure will lead to implementation bugs and more force closes, which are extra costly with OPR.
[/quote]

OPR never requires a forced close, even if the peers fail to agree on the resolution of an HTLC.
In such a case, they can keep the channel open and process additional HTLCs.

When they finally close the channel, they can create a cooperative close that returns their base funds and only burns the single HTLC on which they disagreed.
As a result, the OPR protocol strictly improves scalability, as it completely eliminates the need to resolve HTLCs on-chain.

[quote="morehouse, post:2, topic:1233"]
To properly disincentivize cheating, the amount of funds each party contributes to the burn output must always be higher than the total value of outstanding HTLCs (otherwise users could profit from force closing without forwarding preimages).
[/quote]

Users can *never* profit from force closing a channel with the OPR protocol (as long as their peer follows the protocol and avoids being bullied, as described in the post).
With the OPR protocol, neither the offerer nor the offeree receives the HTLC's funds if they force close the channel with a peer that has not agreed on the HTLC's resolution.
Therefore, there is no such minimum burn contribution that's required to secure the channel.

-------------------------

JohnLaw | 2024-11-09 23:59:58 UTC | #5

[quote="harding, post:3, topic:1233"]
I think OPR needs some sort of unconditional fees to work.

In the current LN protocol, the party that creates a payment only pays forwarding fees if the payment is successful. Unsuccessful payments, and HTLCs designed to fail (called *probes*), cost their creator nothing.

However, if that model was adopted by OPR, then Mallory could send a constant stream of probes through Bob to random endpoints. Each of those endpoints would fail the probes (because they wouldn’t know the payment preimage) in the normal case, but it would occasionally be the case that there would be a problem downstream and a penalty payment would propagate back to Mallory.
[/quote]

I'm afraid I'm not following you here.
What do you mean by a "penalty payment"?

Do you mean the case where Bob does not get the hash preimage but Bob fails to provide a timely update_fail_htlc message?
In this case, the HTLC fails with the OPR protocol (even though Bob didn't explicitly fail it quickly).

As long as Bob eventually realizes that he failed to resolve the HTLC by its expiry, he will realize that the HTLC failed and he will agree to refund the HTLC to Mallory.

However, in this case Mallory only gets back the funds Mallory put in the burn output (namely the HTLC payment amount and Mallory's matching funds), and Bob's matching funds for this HTLC are returned to *Bob*.
Thus, there is no penalty payment from Bob to Mallory.

[quote="harding, post:3, topic:1233"]
Even with unconditional fees, OPR would seem to create an incentive for an attacker to create disruptions within the network.
[/quote]

Mallory would never attack Bob in this manner, as Mallory would never benefit from such an attack.

It's true that someone outside of the channel partners could mount a DOS attack on a channel in order to make a payment fail.
Such an attack would be an attack on lightning (and not an attempt to steal funds), as it would fail a payment and it would make one of the routing nodes lose the value of the payment (they would pay in the channel in which they offered the HTLC, but would not collect in the channel in which they were offered the HTLC).

I see your point about the potential to rely on high-availability infrastructure (like CloudFlare) to be able to prevent such attacks.
It's hard to fully quantify the tradeoffs between the need for high availability with OPR versus the current lightning protocol.
On the one hand, OPR is more susceptible to such attacks, given its very short expiries.
On the other hand, the amount at risk with OPR could be much smaller if OPR replaces a single large HTLC with a stream of tiny, fast HTLCs.

[quote="harding, post:3, topic:1233"]
says the current median percentage fee (not factoring in the base fee) is “0.000072 sat/sat”, or 0.0072% per hop, or 0.08% per payment assuming 11 hops, so OPR accidental routing failures is roughly expected to double the routing cost over the current system or 10,000x increase the cost over a (very theoretical) expected floor.
[/quote]

Thanks for the data and analysis.

Yes, the numbers I gave are much higher in terms of relative fees, but the base fees are quite important when considering small payments.
The OPR protocol should have much lower base fees (especially if on-chain feerates increase), as there is never an on-chain fee required to resolve an OPR payment.

In any case, I think it's quite possible that a user would choose to pay an additional $0.01 in order to guarantee that their $10.00 payment will be resolved within seconds, rather than hours.

-------------------------

JohnLaw | 2024-11-10 00:06:59 UTC | #6

[quote="harding, post:3, topic:1233"]
It’s my belief (possibly ill-informed) that there’s been minimal work in this direction due to HTLC settlement cost griefing not being perceived as a major problem at present.
[/quote]

Fair enough.

Let's assume the current protocol is safe for payments that are smaller than the on-chain fees required to claim them.
I think the reason they are safe is:

  1) no one is willing to be shown to be a sucker from whom funds can be stolen, therefore:
  2) everyone is willing to claim the funds that are owed to them, even if this results in a net loss due to fees, therefore:
  3) Alice will never try to steal Bob's funds by using the attack given at the beginning of this post.

Regarding point 3, Alice won't try to steal because doing so won't succeed (due to point 2) and would make it look like she has low availability (even though she wouldn't lose funds if she tried).

If we believe the above reasoning, then I think it's fair to say that the OPR protocol is safe (for any size payment) because:

  4) no one is willing to be shown to be a sucker from whom funds can be stolen, therefore:
  5) everyone is willing to prevent a bullying attack, even if they would obtain more funds if they could be bullied, therefore:
  6) no one will ever attack their peer by failing to agree to the correct resolution of an HTLC.

Regarding point 6, no one will try such an attack, both because it won't succeed (due to point 5) and it would make the attacker lose funds and reputation.

If we agree to the reasoning above, that means the OPR protocol is safe in the sense that one's peer will always try to resolve each HTLC correctly.

Does that sound right?

There are tradeoffs with the OPR protocol's use of a burn output because one's peer can fail permanently or the partners may fail to agree on an HTLC's resolution (despite their best efforts).
However, those costs provide the benefits of resolving payments in a completely off-chain and scalable manner.
I could certainly image that those costs could be worth paying someday, especially if lightning becomes widely used and the blockchain becomes very congested.

There's also a potential cost to supporting short HTLC expiries, as they increase the risk of losing the payment because of a failure.
However, this is tunable.

For example, consider the case where each router sets their htlc_expiry_delta_msec parameter to match the off-chain (G) portion of their CLTV delta parameter.
In this case, there's no obvious risk from having shorter expiries, and users are likely to appreciate the faster payment resolutions.
In practice, I'm guessing that the htlc_expiry_delta_msec parameters will be much shorter, but that's just a guess for now.

In any case, prior to the OPR protocol, I wasn't aware of a way to resolve payments quickly and safely without ever having to go onchain.

I hope that adding these capabilities to our toolkit will help lightning scale and become more widely adopted.

-------------------------

