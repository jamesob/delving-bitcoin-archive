# Depots: Theft-Proof, Self-Custodial Bitcoin For Billions Of Users

JohnLaw | 2026-09-15 20:43:11 UTC | #1

```
TL;DR
-----
In order to allow many billions of users to safely own, send and receive bitcoin, a scaling protocol must:
* have a tiny on-chain footprint (e.g., 1 or 2 vbytes/user/year),
* allow users to participate even if they don't own enough bitcoin to pay the fee for putting a transaction on-chain, and
* avoid creating more off-chain transactions than can be put on-chain before theft occurs.

No trust-free scaling protocol meets these requirements, so it's widely believed that most users will have to rely on custodial, trust-based solutions such as exchanges.
This post presents a protocol for depots that meets all three of the above requirements.
A single depot allows many thousands or millions of users to own bitcoin in off-chain Lightning channels.

Depots rely on griefer penalization for their security.
In a depot a large number of users partner with an operator, with neither the users nor the operator being able to steal funds.
The users can force the operator to lose some funds (and vice versa), but only by losing a proportional expected amount of their own funds.
As a result, someone who only pairs with self-interested parties will never be griefed.

Depots require the CTV (CheckTemplateVerify) [BIP-119] and CSFS (CheckSigFromStack) [BIP-348][Roose25] enhancements to the Bitcoin protocol.
Depots are much more efficient if the PC (PairCommit) [BIP-PC][Black24], multiply (OP_MUL) [Russell24] and mod (OP_MOD) [Russell24] operators are supported, but they don't require those operators.

Depot Overview
--------------
In order to create a depot, an operator puts a Funding transaction with a single Taproot output on-chain.
This output can be used to create probabilistic off-chain Lightning channels for many thousands or millions of users.
The output's internal key is the root of a merkle tree containing the parameters of the depot and the root of a merkle-sum tree [Osunto21] that specifies the users of the depot and the allocation of a maximum number of channels to each user.

To acquire bitcoin in a depot, a user makes a Lightning payment to the operator to purchase a number of channels not exceeding the user's maximum allocation.
Each channel is owned by the user and the operator and has a 1/P chance of being a hit, where P is a large prime (e.g., between a hundred thousand and a billion).
The channel can only be put on-chain if it's a hit, but if it is put on-chain it pays the user P times as much as the user paid for the channel.

Each depot has a predefined expiry and the depot's channels can't be put on-chain until after the expiry.
Users should drain their funds prior to the expiry, either by making Lightning payments or by transferring their funds to another depot.
After the expiry, the operator claims all funds in the depot unless a user who didn't drain has a hit.
If exactly one channel is a hit, the user owning that channel can put it on-chain, thus splitting the depot's funds between the user and the operator.
On the other hand, if more than one channel is a hit, all funds in the depot are burned. 

Depots combine 3 key ideas:
* a time-limited output that's funded and put on-chain by an operator and supports a large number of users (as in Timeout Trees [Law21][Law23], Ark [Keceli23] or SuperScalar [ZmnSC24]),
* randomization that replaces a small deterministic output with a much larger probabilistic output having the same expected value [Rivest97][Cal12], and
* griefer penalization to encourage cooperative behavior [Riard22][Law24][Law25].

These ideas work together:
* the use of an operator eliminates the need to coordinate large numbers of users, as each user only interacts with the operator,
* the use of a time limit allows users to obtain Lightning channels that never have to be put on-chain (as drained channels can be reclaimed by the operator without putting them on-chain),
* randomization allows users who have insufficient funds for putting a transaction on-chain to (probabilistically) obtain high-value transactions that can be put on-chain,
* randomization results in a small constant-size on-chain footprint per depot even in the worst case (thus supporting an unbounded number of users per depot and eliminating the risk of a "thundering herd" [Towns23] of users attempting to go on-chain if many depots reach their expiry without users draining their funds),
* griefer penalization incentivizes users to drain their funds (and the operator to cooperate in draining those funds) before the expiry,
* griefer penalization penalizes an operator who attempts to steal funds by putting the wrong transactions on-chain, and
* griefer penalization allows Lightning sends and receives to be resolved entirely off-chain.

Depots are a remarkably scalable way of supporting users who own small amounts of bitcoin.
As is shown below, depots require at most a small fraction of the block space to create Lightning channels for 10 billion users, where:
* each user owns a Lightning channel with 10 different operators, and
* the users send and receive hundreds of thousands of payments per second to and from those Lightning channels.

Throughout the remainder, let U be a user and O be the operator.

Paper And Programs
------------------
A more complete description (including transaction details, examples, formulas, and analysis), is given in a paper [Law26], along with programs that automate the configuration, operation, and analysis of depots.

Lightning Security
------------------
The security of depots is best understood by first examining the security of the Lightning protocol.
A Lightning user always has transactions which give them their full channel payment, minus fees, provided those transactions can be put on-chain within a predefined time window.

If the value of a Lightning channel or payment equals the cost of claiming that channel or payment on-chain, Lightning's security properties depend on the behavior of the channel partners.
In particular, if Bob is forced by Alice to go on-chain to claim his Lightning balance or payment, and if Bob has to pay the on-chain fees, Bob will have to pay the entire value of his balance or payment to claim it, thus losing his funds.
Note that Alice loses none of her funds by performing this attack.
Despite the lack of any explicit penalty, such griefing attacks are rare in practice [Harding24] because Alice doesn't benefit from the attack and her reputation is damaged.
As a result, Lightning payments that are comparable to the on-chain cost of securing the payment are made routinely and are secure.

If the value of a Lightning payment is smaller than the cost of claiming the payment on-chain, choosing to claim the payment results in a net loss to the party claiming it.
Therefore, a self-interested party may be expected to forgo claiming the payment.
Also, note that in this case the attacker has a potential self-interest in performing the attack, because they may gain the value of the payment (due to its not being claimed on-chain).
Such an attack will be called a bullying attack.

In reality, parties do choose to go on-chain to claim Lightning payments, even when the cost of claiming the payment exceeds the value of the payment.
This is because a party who succumbs to a bullying attack proves that they can be victimized by similar attacks in the future.
Furthermore, standard Lightning wallet software claims the payment on-chain (for the reason given above) [Moreh24], so in practice users of such software don't make a conscious decision to accept a one-time loss in order to preserve their reputation.
However, even if a conscious decision were required, there are strong human emotions that protect people against bullying.
No one wants to be seen as a sucker who can be taken advantage of.
This has been demonstrated with the Ultimatum Game [Ultimatum] in which, even when presented with a one-time opportunity to obtain free money, many users will forgo that money if they feel that accepting it would allow another party to take advantage of them.
Another way of explaining why users resist bullying is that refusing to be bullied is in one's long-term interest (because one's reputation is preserved), despite the short-term costs.
Because Lightning users refuse to be bullied and their partners know they will refuse to be bullied, Lightning payments are routinely made that are smaller than the on-chain cost of claiming them.
```

```


In conclusion, there's extensive practical evidence that Lightning payments are secure, even when that security depends on the parties acting in their own (long-term) self-interest.
Specifically, there doesn't appear to be evidence of Lightning users performing griefing attacks, despite there being are no financial penalties for such attacks.

Depot Security
--------------
A depot's security relies on users putting transactions on-chain within predefined time windows and on the parties acting in their own (long-term) self-interest.
As a result, the observed security of the Lightning protocol (even when fees exceed payment amounts) is strong evidence for the security of depots.

Quantified Griefer Penalization
===============================
In fact, unlike the Lightning protocol, depots penalize any party that performs a griefing attack.
Specifically, each depot defines two griefing security parameters, GrO and GrU. At any give time, let
* LossO = the operator's expected loss and
* LossU = the sum of the expected losses of all users.

It will be required that at all times:
* LossO/LossU >= GrO and
* LossU/LossO >= GrU
where 0 < GrO, 0 < GrU and GrO*GrU < 1.
For example, a value of GrU = 0.30 means that users who perform a griefing attack must expect to lose at least 30% as much as the operator loses.

These security parameters are set large enough to discourage both the operator and the users from griefing each other.
Note that O devotes significant resources by funding the depot and O can create a series of depots, thus building a reputation for following the depot protocol.
In contrast, users may be new to Bitcoin and they do not fund depots.
As a result, it's likely that GrO will be smaller than GrU in practice.
Also, note that even very small values of GrO and GrU should be effective, as any positive value will deter a self-interested party from performing a griefing attack.
In particular, values of GrO up to 0.10 and GrU up to 0.50 yield relatively efficient depots.

Simplified Depot Protocol For Creating And Revoking Channels
------------------------------------------------------------
This section presents a simplified version of the protocol for creating and revoking channels in a depot.

In order to deposit funds in a depot, U purchases one or more off-chain Lightning channels from O.
To purchase channel i, U makes a guess G_i that's a number in the range 0..P-1 and gets O's signature for that guess.
When getting O's signature, U doesn't reveal G_i to O.
Instead, U selects a salt value Salt_i that's hashed together with G_i (using the PC operator) before getting O's signature, thus preventing O from learning G_i. 

After the expiry, O reveals the depot's target t, which is a number in the range 0..P-1.
(If O fails to reveal a target within a fixed window of time, a default target is used.)
A channel is a hit if the user's guess for that channel matches the target value.

Once U has purchased channel i, U can use channel i to send and receive bitcoin.
If U and O follow the depot protocol, U will drain the channel before the depot's expiry by making payments from U and by moving U's funds to a channel in another depot.
As part of draining channel i, U will revoke channel i by revealing G_i and Salt_i to O, thus allowing O to select a target that differs from the channel's guess and eliminating the possibility of the channel being a hit.
A channel i is said to be active from the time U gets O's signature for guess G_i until U revokes the channel.
Note that O can't sell an unbounded number of channels, because when channels are revoked they could reveal guesses for every possible target value, at which point it would be impossible for O to select a target that doesn't hit a revoked channel.

In order to purchase a channel that pays X sats to U, U makes a Lightning payment of X/P sats to O.
Thus, a channel's cost exactly equals its expected value to U if it's the only channel that's active after the depot's expiry, while a channel's cost is an upper bound on its expected value to U if there are multiple active channels after the depot's expiry.
```

```
If an active Lightning channel pays X sats to U when put on-chain, U's balance in the Lightning channel is X/P.
U's balance in the depot equals the total of U's balances in all of U's active channels.
Finally, if an active Lightning channel pays Y sats to O when put on-chain, O's balance in the Lightning channel is Y/P.

Transactions
============
The on-chain transactions that implement the simplified protocol for creating and revoking channels are shown in Figures 1 and 2 below.
Figure 1 shows the transactions put on-chain when O provides the target value t, and Figure 2 shows the transactions put on-chain when O fails to provide such a target value within the required time window.

In these figures, black values come from the tapleaf script and red values come from the witness stack.
Each transaction output shows the conditions required for spending the output.
The first condition, if required, is a relative delay of one or more to_self_delay (tsd) safety parameters or 20 years (for a burn), or a parenthesized absolute delay until the expiry (E) of the depot.
In addition, outputs show the user(s) (if any) who must sign for the child transaction (e.g., user U or V, operator O, or both U and O, or the owner of a Lightning channel's revocation key).

The notation CTV(X) denotes that the CTV operator forces the child transaction to be X, where X is either fixed in advance by the parent transaction (in which case the value X comes from the tapleaf script and is shown as black) or selected dynamically by the party creating the child transaction (in which case the value X comes from the witness stack and is shown as red).
The notation CSFS(O,<X_1,X_2,..,X_n>,Sig) denotes that the CSFS operator requires O's signature (Sig) for the value PC(X_1, PC(X_2 .. PC(X_n-1, X_n))) where PC is the PairCommit operator.
![figure1|386x500](upload://sXsyWyXMdIKHAJvaqVLOG7IIlUz.jpeg)
Figure 1.  Normal resolution of a simplified depot using an operator-selected target.
![figure2|386x500](upload://qvLxuUiwSy72tAdSYhb4tfycbMv.jpeg)
Figure 2.  Default resolution of a simplified depot when no operator-selected target is provided.

In Figure 1, O creates a depot by putting a single-output Funding (F) transaction on-chain.
After the depot's expiry E and before E+tsd, O uses a Target (Targ) transaction to spend F's output.
This Target transaction reveals the target value t by providing O's target signature, TaSig, for t.

If no user has a channel that hits the target t, O waits tsd and then claims the Target transaction's output.
On the other hand, if there's a user U with a channel i that hits the target t, U puts a Hit transaction on-chain that reveals that channel i is a hit.
More precisely, the witness for the Hit transaction includes the target value t, where t is proven to be the target by providing O's target signature, TaSig, for t which U obtained when the Target transaction was put on-chain.
The witness for the Hit transaction also includes O's guess signature, GuSig_i, for the value <Clm_Ui, Clm'_Ui, i, Salt_i, G_i> where Clm_Ui and Clm'_Ui are the hashes of Claim transactions that allow U to put a Lightning channel on-chain, i is the index of the channel that's a hit, and G_i = t.
The signature GuSig_i was given by O to U when U purchased channel i.

The Hit transaction has two outputs, numbered 0 and 1, each of which has exactly half of the Hit transaction's input value (ignoring fees).

If U is the owner of the only channel that hits, U puts a Claim transaction (Clm_Ui) on-chain that spends the Hit transaction's output 1.
Then, after a delay of tsd, either U or O can put the final Commitment transaction (Com_Uif or Com_Oif) for channel i's associated Lightning channel on-chain.

On the other hand, if there's more than one hit, users burn both of the Hit transaction's outputs.
In particular, if a user has a channel j ≠ i that hits, that user will burn both of the Hit transaction's outputs by providing the guess i signature GuSig_i that was revealed in the witness for the Hit transaction and the guess j signature GuSig_j that the user obtained from O when purchasing channel j.

A final possibility is that O puts an arbitrary transaction Arb_Oi on-chain that spends the Hit transaction's output 1, thus griefing U by making U lose the value that Com_Uif would pay to U.
However, in doing so O must reveal a guess signature ArbGuSig_i ≠ GuSig_i, thus allowing U to burn the Hit transaction's output 0.
The depot protocol puts a lower bound (OperatorChannelMin) on O's balance in a Lightning channel, thus guaranteeing that O's use of an Arb_Oi transaction will result in a net loss to O that's large enough (based on the griefing parameter GrO) to prevent O from putting Arb_Oi on-chain.

Figure 2 shows the operation of the protocol when O fails to put a Target transaction on-chain before E+tsd.
In this case, any user can use the default Target' (Targ') transaction to spend the Funding transaction's output.
The default Target' (Targ') transaction's witness does not provide a target value t.
Instead, the protocol in Figure 2 matches the protocol in Figure 1 except it uses default transactions (indicated with a prime) that require a target with a fixed value of 0.

Actual Depot Protocol For Creating And Revoking Channels
--------------------------------------------------------
While the above simplified protocol captures the essence of how channels are created, the actual depot protocol differs in four ways:* rather than having only one target t, O creates a vector of h targets (thus allowing O to sell more channels),
* O's signature for channel i covers not only U's guess G_i (and its associated salt value) but also a vector of g random factors that are multiplied by another vector of g numbers provided by O after the expiry and added to U's guess (mod P) (thus preventing U from knowing if U's active guesses match any revoked guesses),
* rather than O giving U O's signature for channel i, O gives U an adaptor signature for channel i (thus allowing U's purchase of channel i to be atomic with U's Lightning payment to O for channel i), and
* the salt value that U combines with U's guess for channel i is a joint signature, by U and O, for the value i (thus allowing U's revocation of channel i to be atomic with O getting a Lightning PTLC receipt for a payment that matches the revoked channel's value).

The actual depot protocol is given in Section 3.2 of [Law26].

The Off-Chain Lightning Protocol
--------------------------------
In order to make payments to and from U's Lightning channels in a depot, U and O put the value of the payment, plus a fixed fraction of the payment amount (called matching funds), into burn outputs in U's Lightning channels [Law24][Law26].
The key is that during the entire process both U and O have sufficient funds in the burn output to incentivize them to resolve the payment correctly.
Formulas for determining U's and O's contributions to the burn output are given in Appendix F of [Law26] and routines for implementing payments by moving funds to and from burn outputs are given in the program depot.py [Law26].

Securing Against Failures To Drain
----------------------------------
If a depot has 2 or more hits, the entire value of the depot is burned.
As a result, users could attempt to grief the operator by holding active channels past the depot's expiry.
Similarly, the operator could attempt to grief users by preventing them from draining their funds from the depot prior to its expiry.
These griefing attacks are prevented by placing a lower bound (UserChannelMin) on U's balance in each active channel and an upper bound on U's balance in the depot as a function of the number of U's active Lightning channels and U's channel allocation.
These bounds guarantee that if one or more users fail to drain their channels in an attempt to grief O and if O's expected loss is X, then the users performing the griefing attack will expect to lose at least GrU*X.
Similarly, if O fails to allow one or more users to drain their channels in an attempt to grief the users and if the users' expected loss is X, then O will expect to lose at least GrO*X.

Example
-------
Assume a depot has the following parameters:
* GrO = 0.05 (so if O causes users to lose an expected value of x, O must expect to lose at least 0.05*x of O's own funds)
* GrU = 0.30 (so if users cause O to lose an expected value of x, users must expect to lose at least 0.30*x of their own funds)
* balance_range = 1.5 (so given any fixed number of active channels, the maximum possible user's balance is at least 50% larger than the minimum possible user's balance),
* n = 1,000,000 users,
* MaxA = 10,000,000 maximum active channels (so an average user can be allocated 10 channels),
* MaxC = 1,000,000,000 maximum cumulative number of channels created over the lifetime of the depot (so an average user can acquire and revoke up to 1,000 channels during the lifetime of the depot),
* supported_user_balances = 100,000,000 sats (so users can have balances that total at least 100,000,000 sats),
* to_self_delay = 10,000 blocks (about 2.5 months) (that is, users that attempt to go on-chain must monitor the blockchain at least once every 2.5 months), and
* E = the depot's expiry is 100,000 blocks (about 2 years) after the depot is put on-chain.

Given the above parameters, the following parameters can be calculated by the program depot.py [Law26]:
* D = 107,718,596 sats (so the depot is funded by the operator with 107,718,596 sats),
* P = 1,864,361 (so each channel has a 1/1,864,361 chance of being a hit),
* OperatorChannelMin = 2,564,729/P = 1.376 sats (so every active channel must pay O at least 2,564,729 sats and O's balance in every active channel must be at least 1.376 sats),
* UserChannelMin = 12,429,069/P = 6.667 sats (so every active channel must pay a user at least 12,429,069 sats and each user's balance in every active channel must be at least 6.667 sats),
* MaxH = 5.364 maximum expected number of hits (so there are 5.364 expected hits if all MaxA channels are active),
* g = 25 (so the vectors that randomize the guesses each have 25 components),
* h = 54 targets (so each channel is assigned a target t_y where 1 ≤ y ≤ 54), and
* max_utilization = 0.9283 (so up to 92.83% of the depot's value can be sold to users).

Even if there is only a single depot created per block, depots with the above parameters (except varying in the number of sats per depot) can provide 10 Lightning channels to each of 10 billion users and can support over 900,000 payments per second continuously (see Section 5 of [Law26]).

Dynamic Channel Allocation And Reallocation
-------------------------------------------
While the depot protocol given above requires that each user's channel allocation is fixed for the entire life of the depot, it's easy to modify the protocol to support dynamic allocation and reallocation of channels to users (see Section 6.1 of [Law26]).

Limitations
-----------
Depots have a number of limitations, including:
* Reliance on griefer penalization,
* Reliance on expected values,
* Potential to delay access to users' funds,
* Bounds on the size of large atomic sends and receives,
* Capital inefficiency due to use of matching funds for payments, and
* Capital inefficiency due to inability to allocate all depot funds to users.
These and other limitations are examined in greater detail in Section 7 of [Law26].

Related Work
------------
Depots are based on a number of previously-published ideas.

The idea of a time-limited output that's funded and put on-chain by an operator for a large number of users was presented by Law [Law21][Law23] (in the context of timeout trees), Keceli [Keceli23] (in the context of Ark) and ZmnSCPxj [ZmnSC24] (in the context of SuperScalar).

Rivest proposed replacing a small deterministic output with a probabilistic output with the same expected value in order to reduce processing overhead in the context of micropayments [Rivest97] and Caldwell made a similar proposal for small bitcoin payments [Cal12].

The idea of using a burn output to encourage channel partners to cooperate was introduced by Riard [Riard22] (in the context of unjamming Lightning) and utilized by Law [Law24][Law25] (in the context of resolving payments off-chain and unjamming Lightning).

Finally, the depot protocol has some commonalities with the Last Mile protocol [Hynek24], as both protocols achieve security by appealing to each party's self-interest.
However, the Last Mile protocol differs from the depot protocol in many ways.
For example, the Last Mile protocol motivates cooperation by having both parties contribute to fees that are paid immediately to a miner (rather than putting funds in a burn output) and it requires constant monitoring of the mempool so that high-fee transactions can be sent to miners in time to get them included in the blockchain (as opposed to giving users multiple months in which to respond to on-chain transactions).

Conclusions
-----------
Bitcoin allows anyone to hold their own funds and to make safe, censorship-resistant online payments.
However, due to block space limitations, only a tiny fraction of the world's payments can be performed on-chain.
Protocols for Lightning channels, channel factories, timeout trees, Ark and SuperScalar have been proposed for improving Bitcoin's scalability by allowing most payments to be made off-chain.
These protocols greatly decrease the number of on-chain transactions (at least in the common case), but they don't allow everyone in the world to make frequent casual online payments.

First, all of the above protocols give users transactions that they may need to put on-chain in order to prevent theft.
Therefore, if a user doesn't own enough bitcoin to pay the fees for putting a transaction on-chain, the above protocols don't allow the user to prevent the theft of their funds.

Second, all of the above protocols prevent theft by forcing users to put certain transactions on-chain within a bounded number of blocks (or time).
While the protocols are designed to keep these theft-preventing transactions off-chain in the common case, there's no guarantee that they won't have to be put on-chain.
In fact, once the number of theft-preventing transactions held off-chain exceeds the block space available for putting them on-chain within the required window, the protocols' safety guarantees disappear [Towns23][Carval26].
As a result, rather than providing safe long-term scaling solutions, these protocols provide safe scaling of payments until their success undermines their safety, at which point a catastrophic breakdown and massive theft become possible.

This post presents depots that allow large numbers of casual users to self-custody bitcoin in a theft-free manner and to make a huge number of casual, censorship-resistant online payments.
Depots don't suffer from either of the limitations described above, as they prevent theft even for users who own tiny amounts of bitcoin and they don't create more off-chain transactions than can be put on-chain before theft occurs.
Depots also support on-boarding new users with balances that are a small fraction of a sat.
As a result, depots have the potential to transform Bitcoin into a widely-used means of payment.

Depots require the CTV (CheckTemplateVerify) [BIP-119] and CSFS (CheckSigFromStack) [BIP-348][Roose25] enhancements to the Bitcoin protocol.
In addition, depots are much more efficient if the PC (PairCommit) [BIP-PC][Black24], multiply (OP_MUL) [Russell24] and mod (OP_MOD) operators [Russell24] are supported.
It's hoped that the ability to support depots will help to motivate these enhancements to the Bitcoin protocol.

Acknowledgments
---------------
The author would like to thank David Harding for bringing to the author's attention previous work on probabilistic low-value payments [Harding24] and Clara Shikhelman for noting the connection between the Ultimatum Game and the behavior of Lightning users.

References
----------
[BIP-119] BIP-119, "BIP 119: OP_CHECKTEMPLATEVERIFY", https://github.com/bitcoin/bips/blob/173f386dfb8b4f1c4b16d69190a753802802ac4d/bip-0119.mediawiki

[BIP-348] BIP-348, "BIP 348: OP_CHECKSIGFROMSTACK", https://github.com/bitcoin/bips/blob/050d422b2ac24d8221edab0ff0053e0f585409f7/bip-0348.md

[BIP-PC] BIP-PC, "OP_PAIRCOMMIT", https://github.com/bitcoin/bips/pull/1699

[Black24] Black, "LNHANCE bips and implementation", https://delvingbitcoin.org/t/lnhance-bips-and-implementation/376

[Cal12]	Caldwell, "Sustainable nanopayment idea: Probabilistic Payments", https://bitcointalk.org/index.php?topic=62558.0

[Carval26] Carvalho, "Conservation of Blockspace", https://blockspace.science/

[Harding24] Harding, Response to: "A Fast, Scalable Protocol For Resolving Lightning Payments", https://delvingbitcoin.org/t/a-fast-scalable-protocol-for-resolving-lightning-payments/1233/3

[Hynek24] Hynek, "Last Mile Transactions", https://github.com/hynek-jina/Hynek/blob/0301c90015f9d66fb22e347f21924dc9d04f0bc8/blog/Last%20mile%20transactions.md

[Keceli23] Keceli, "Ark: An Alternative Privacy-preserving Second Layer Solution", https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-May/021694.html

[Law21] Law, "Scaling Bitcoin With Inherited IDs", https://github.com/JohnLaw2/btc-iids

[Law23] Law, "Scaling Lightning With Simple Covenants", https://github.com/JohnLaw2/ln-scaling-covenants

[Law24]	Law, "A Fast, Scalable Protocol For Resolving Lightning Payments", https://github.com/JohnLaw2/ln-opr

[Law25]	Law, "Fee-Based Spam Prevention For Lightning", https://github.com/JohnLaw2/ln-spam-prevention

[Law26]	Law, "Depots: Theft-Proof, Self-Custodial Bitcoin For Billions Of Users" and the associated programs depot.py, burn_depot.py and coupon.py, https://github.com/JohnLaw2/ln-depots

[Moreh24] Morehouse, Response to: "A Fast, Scalable Protocol For Resolving Lightning Payments", https://delvingbitcoin.org/t/a-fast-scalable-protocol-for-resolving-lightning-payments/1233/2

[Osunto21] Osuntokun, "Taro: A Taproot Asset Representation Overlay", https://github.com/Roasbeef/bips/blob/d9b07c16d76be6967c67a678dc2f61f5e891c218/bip-taro.mediawiki

[Riard22] Riard, "Unjamming lightning (new research paper)", https://www.mail-archive.com/lightning-dev@lists.linuxfoundation.org/msg02996.html

[Rivest97] Rivest, "Electronic Lottery Tickets As Micropayments", https://people.csail.mit.edu/rivest/pubs/Riv97b.pdf

[Roose25] Roose, "CTV+CSFS: Can we reach consensus on a first step towards covenants?", https://delvingbitcoin.org/t/ctv-csfs-can-we-reach-consensus-on-a-first-step-towards-covenants/1509

[Russell24] Russell, "Restoring Bitcoin’s Full Script Power", https://rusty.ozlabs.org/2024/01/19/the-great-opcode-restoration.html

[Towns23] Towns, "Re: Scaling Lightning With Simple Covenants", https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-September/021943.html

[Ultimatum] "Ultimatum Game", https://en.wikipedia.org/wiki/Ultimatum_game

[ZmnSC24] ZmnSCPxj, "SuperScalar: Laddered Timeout-Tree-Structured Decker-Wattenhofer Factories With Pseudo-Spilman Leaves", https://delvingbitcoin.org/t/superscalar-laddered-timeout-tree-structured-decker-wattenhofer-factories-with-pseudo-spilman-leaves/1242

-------------------------

JohnLaw | 2026-09-15 20:45:12 UTC | #2

I'm not sure that the figures and the text after the figures came through in the post above.

In case they didn't, I'll try reposting them below.

-------------------------

JohnLaw | 2026-09-15 20:46:59 UTC | #3

![figure1|386x500](upload://sXsyWyXMdIKHAJvaqVLOG7IIlUz.jpeg)

![figure2|386x500](upload://qvLxuUiwSy72tAdSYhb4tfycbMv.jpeg)

-------------------------

JohnLaw | 2026-09-15 20:55:54 UTC | #4

```


The Hit transaction has two outputs, numbered 0 and 1, each of which has exactly half of the Hit transaction's input value (ignoring fees).

If U is the owner of the only channel that hits, U puts a Claim transaction (Clm_Ui) on-chain that spends the Hit transaction's output 1.
Then, after a delay of tsd, either U or O can put the final Commitment transaction (Com_Uif or Com_Oif) for channel i's associated Lightning channel on-chain.

On the other hand, if there's more than one hit, users burn both of the Hit transaction's outputs.
In particular, if a user has a channel j ≠ i that hits, that user will burn both of the Hit transaction's outputs by providing the guess i signature GuSig_i that was revealed in the witness for the Hit transaction and the guess j signature GuSig_j that the user obtained from O when purchasing channel j.

A final possibility is that O puts an arbitrary transaction Arb_Oi on-chain that spends the Hit transaction's output 1, thus griefing U by making U lose the value that Com_Uif would pay to U.
However, in doing so O must reveal a guess signature ArbGuSig_i ≠ GuSig_i, thus allowing U to burn the Hit transaction's output 0.
The depot protocol puts a lower bound (OperatorChannelMin) on O's balance in a Lightning channel, thus guaranteeing that O's use of an Arb_Oi transaction will result in a net loss to O that's large enough (based on the griefing parameter GrO) to prevent O from putting Arb_Oi on-chain.

Figure 2 shows the operation of the protocol when O fails to put a Target transaction on-chain before E+tsd.
In this case, any user can use the default Target' (Targ') transaction to spend the Funding transaction's output.
The default Target' (Targ') transaction's witness does not provide a target value t.
Instead, the protocol in Figure 2 matches the protocol in Figure 1 except it uses default transactions (indicated with a prime) that require a target with a fixed value of 0.

Actual Depot Protocol For Creating And Revoking Channels
--------------------------------------------------------
While the above simplified protocol captures the essence of how channels are created, the actual depot protocol differs in four ways:
* rather than having only one target t, O creates a vector of h targets (thus allowing O to sell more channels),
* O's signature for channel i covers not only U's guess G_i (and its associated salt value) but also a vector of g random factors that are multiplied by another vector of g numbers provided by O after the expiry and added to U's guess (mod P) (thus preventing U from knowing if U's active guesses match any revoked guesses),
* rather than O giving U O's signature for channel i, O gives U an adaptor signature for channel i (thus allowing U's purchase of channel i to be atomic with U's Lightning payment to O for channel i), and
* the salt value that U combines with U's guess for channel i is a joint signature, by U and O, for the value i (thus allowing U's revocation of channel i to be atomic with O getting a Lightning PTLC receipt for a payment that matches the revoked channel's value).

The actual depot protocol is given in Section 3.2 of [Law26].

The Off-Chain Lightning Protocol
--------------------------------
In order to make payments to and from U's Lightning channels in a depot, U and O put the value of the payment, plus a fixed fraction of the payment amount (called matching funds), into burn outputs in U's Lightning channels [Law24][Law26].
The key is that during the entire process both U and O have sufficient funds in the burn output to incentivize them to resolve the payment correctly.
Formulas for determining U's and O's contributions to the burn output are given in Appendix F of [Law26] and routines for implementing payments by moving funds to and from burn outputs are given in the program depot.py [Law26].

Securing Against Failures To Drain
----------------------------------
If a depot has 2 or more hits, the entire value of the depot is burned.
As a result, users could attempt to grief the operator by holding active channels past the depot's expiry.
Similarly, the operator could attempt to grief users by preventing them from draining their funds from the depot prior to its expiry.
These griefing attacks are prevented by placing a lower bound (UserChannelMin) on U's balance in each active channel and an upper bound on U's balance in the depot as a function of the number of U's active Lightning channels and U's channel allocation.
These bounds guarantee that if one or more users fail to drain their channels in an attempt to grief O and if O's expected loss is X, then the users performing the griefing attack will expect to lose at least GrU*X.
Similarly, if O fails to allow one or more users to drain their channels in an attempt to grief the users and if the users' expected loss is X, then O will expect to lose at least GrO*X.

Example
-------
Assume a depot has the following parameters:
* GrO = 0.05 (so if O causes users to lose an expected value of x, O must expect to lose at least 0.05*x of O's own funds)
* GrU = 0.30 (so if users cause O to lose an expected value of x, users must expect to lose at least 0.30*x of their own funds)
* balance_range = 1.5 (so given any fixed number of active channels, the maximum possible user's balance is at least 50% larger than the minimum possible user's balance),
* n = 1,000,000 users,
* MaxA = 10,000,000 maximum active channels (so an average user can be allocated 10 channels),
* MaxC = 1,000,000,000 maximum cumulative number of channels created over the lifetime of the depot (so an average user can acquire and revoke up to 1,000 channels during the lifetime of the depot),
* supported_user_balances = 100,000,000 sats (so users can have balances that total at least 100,000,000 sats),
* to_self_delay = 10,000 blocks (about 2.5 months) (that is, users that attempt to go on-chain must monitor the blockchain at least once every 2.5 months), and
* E = the depot's expiry is 100,000 blocks (about 2 years) after the depot is put on-chain.

Given the above parameters, the following parameters can be calculated by the program depot.py [Law26]:
* D = 107,718,596 sats (so the depot is funded by the operator with 107,718,596 sats),
* P = 1,864,361 (so each channel has a 1/1,864,361 chance of being a hit),
* OperatorChannelMin = 2,564,729/P = 1.376 sats (so every active channel must pay O at least 2,564,729 sats and O's balance in every active channel must be at least 1.376 sats),
* UserChannelMin = 12,429,069/P = 6.667 sats (so every active channel must pay a user at least 12,429,069 sats and each user's balance in every active channel must be at least 6.667 sats),
* MaxH = 5.364 maximum expected number of hits (so there are 5.364 expected hits if all MaxA channels are active),
* g = 25 (so the vectors that randomize the guesses each have 25 components),
* h = 54 targets (so each channel is assigned a target t_y where 1 ≤ y ≤ 54), and
* max_utilization = 0.9283 (so up to 92.83% of the depot's value can be sold to users).

Even if there is only a single depot created per block, depots with the above parameters (except varying in the number of sats per depot) can provide 10 Lightning channels to each of 10 billion users and can support over 900,000 payments per second continuously (see Section 5 of [Law26]).

Dynamic Channel Allocation And Reallocation
-------------------------------------------
While the depot protocol given above requires that each user's channel allocation is fixed for the entire life of the depot, it's easy to modify the protocol to support dynamic allocation and reallocation of channels to users (see Section 6.1 of [Law26]).

Limitations
-----------
Depots have a number of limitations, including:
* Reliance on griefer penalization,
* Reliance on expected values,
* Potential to delay access to users' funds,
* Bounds on the size of large atomic sends and receives,
* Capital inefficiency due to use of matching funds for payments, and
* Capital inefficiency due to inability to allocate all depot funds to users.
These and other limitations are examined in greater detail in Section 7 of [Law26].

Related Work
------------
Depots are based on a number of previously-published ideas.

The idea of a time-limited output that's funded and put on-chain by an operator for a large number of users was presented by Law [Law21][Law23] (in the context of timeout trees), Keceli [Keceli23] (in the context of Ark) and ZmnSCPxj [ZmnSC24] (in the context of SuperScalar).

Rivest proposed replacing a small deterministic output with a probabilistic output with the same expected value in order to reduce processing overhead in the context of micropayments [Rivest97] and Caldwell made a similar proposal for small bitcoin payments [Cal12].

The idea of using a burn output to encourage channel partners to cooperate was introduced by Riard [Riard22] (in the context of unjamming Lightning) and utilized by Law [Law24][Law25] (in the context of resolving payments off-chain and unjamming Lightning).

Finally, the depot protocol has some commonalities with the Last Mile protocol [Hynek24], as both protocols achieve security by appealing to each party's self-interest.
However, the Last Mile protocol differs from the depot protocol in many ways.
For example, the Last Mile protocol motivates cooperation by having both parties contribute to fees that are paid immediately to a miner (rather than putting funds in a burn output) and it requires constant monitoring of the mempool so that high-fee transactions can be sent to miners in time to get them included in the blockchain (as opposed to giving users multiple months in which to respond to on-chain transactions).

Conclusions
-----------
Bitcoin allows anyone to hold their own funds and to make safe, censorship-resistant online payments.
However, due to block space limitations, only a tiny fraction of the world's payments can be performed on-chain.
Protocols for Lightning channels, channel factories, timeout trees, Ark and SuperScalar have been proposed for improving Bitcoin's scalability by allowing most payments to be made off-chain.
These protocols greatly decrease the number of on-chain transactions (at least in the common case), but they don't allow everyone in the world to make frequent casual online payments.

First, all of the above protocols give users transactions that they may need to put on-chain in order to prevent theft.
Therefore, if a user doesn't own enough bitcoin to pay the fees for putting a transaction on-chain, the above protocols don't allow the user to prevent the theft of their funds.

Second, all of the above protocols prevent theft by forcing users to put certain transactions on-chain within a bounded number of blocks (or time).
While the protocols are designed to keep these theft-preventing transactions off-chain in the common case, there's no guarantee that they won't have to be put on-chain.
In fact, once the number of theft-preventing transactions held off-chain exceeds the block space available for putting them on-chain within the required window, the protocols' safety guarantees disappear [Towns23][Carval26].
As a result, rather than providing safe long-term scaling solutions, these protocols provide safe scaling of payments until their success undermines their safety, at which point a catastrophic breakdown and massive theft become possible.

This post presents depots that allow large numbers of casual users to self-custody bitcoin in a theft-free manner and to make a huge number of casual, censorship-resistant online payments.
Depots don't suffer from either of the limitations described above, as they prevent theft even for users who own tiny amounts of bitcoin and they don't create more off-chain transactions than can be put on-chain before theft occurs.
Depots also support on-boarding new users with balances that are a small fraction of a sat.
As a result, depots have the potential to transform Bitcoin into a widely-used means of payment.

Depots require the CTV (CheckTemplateVerify) [BIP-119] and CSFS (CheckSigFromStack) [BIP-348][Roose25] enhancements to the Bitcoin protocol.
In addition, depots are much more efficient if the PC (PairCommit) [BIP-PC][Black24], multiply (OP_MUL) [Russell24] and mod (OP_MOD) operators [Russell24] are supported.
It's hoped that the ability to support depots will help to motivate these enhancements to the Bitcoin protocol.

Acknowledgments
---------------
The author would like to thank David Harding for bringing to the author's attention previous work on probabilistic low-value payments [Harding24] and Clara Shikhelman for noting the connection between the Ultimatum Game and the behavior of Lightning users.

References
----------
[BIP-119] BIP-119, "BIP 119: OP_CHECKTEMPLATEVERIFY", https://github.com/bitcoin/bips/blob/173f386dfb8b4f1c4b16d69190a753802802ac4d/bip-0119.mediawiki

[BIP-348] BIP-348, "BIP 348: OP_CHECKSIGFROMSTACK", https://github.com/bitcoin/bips/blob/050d422b2ac24d8221edab0ff0053e0f585409f7/bip-0348.md

[BIP-PC] BIP-PC, "OP_PAIRCOMMIT", https://github.com/bitcoin/bips/pull/1699

[Black24] Black, "LNHANCE bips and implementation", https://delvingbitcoin.org/t/lnhance-bips-and-implementation/376

[Cal12]	Caldwell, "Sustainable nanopayment idea: Probabilistic Payments", https://bitcointalk.org/index.php?topic=62558.0

[Carval26] Carvalho, "Conservation of Blockspace", https://blockspace.science/

[Harding24] Harding, Response to: "A Fast, Scalable Protocol For Resolving Lightning Payments", https://delvingbitcoin.org/t/a-fast-scalable-protocol-for-resolving-lightning-payments/1233/3

[Hynek24] Hynek, "Last Mile Transactions", https://github.com/hynek-jina/Hynek/blob/0301c90015f9d66fb22e347f21924dc9d04f0bc8/blog/Last%20mile%20transactions.md

[Keceli23] Keceli, "Ark: An Alternative Privacy-preserving Second Layer Solution", https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-May/021694.html

[Law21] Law, "Scaling Bitcoin With Inherited IDs", https://github.com/JohnLaw2/btc-iids

[Law23] Law, "Scaling Lightning With Simple Covenants", https://github.com/JohnLaw2/ln-scaling-covenants

[Law24]	Law, "A Fast, Scalable Protocol For Resolving Lightning Payments", https://github.com/JohnLaw2/ln-opr

[Law25]	Law, "Fee-Based Spam Prevention For Lightning", https://github.com/JohnLaw2/ln-spam-prevention

[Law26]	Law, "Depots: Theft-Proof, Self-Custodial Bitcoin For Billions Of Users" and the associated programs depot.py, burn_depot.py and coupon.py, https://github.com/JohnLaw2/ln-depots

[Moreh24] Morehouse, Response to: "A Fast, Scalable Protocol For Resolving Lightning Payments", https://delvingbitcoin.org/t/a-fast-scalable-protocol-for-resolving-lightning-payments/1233/2

[Osunto21] Osuntokun, "Taro: A Taproot Asset Representation Overlay", https://github.com/Roasbeef/bips/blob/d9b07c16d76be6967c67a678dc2f61f5e891c218/bip-taro.mediawiki

[Riard22] Riard, "Unjamming lightning (new research paper)", https://www.mail-archive.com/lightning-dev@lists.linuxfoundation.org/msg02996.html

[Rivest97] Rivest, "Electronic Lottery Tickets As Micropayments", https://people.csail.mit.edu/rivest/pubs/Riv97b.pdf

[Roose25] Roose, "CTV+CSFS: Can we reach consensus on a first step towards covenants?", https://delvingbitcoin.org/t/ctv-csfs-can-we-reach-consensus-on-a-first-step-towards-covenants/1509

[Russell24] Russell, "Restoring Bitcoin’s Full Script Power", https://rusty.ozlabs.org/2024/01/19/the-great-opcode-restoration.html

[Towns23] Towns, "Re: Scaling Lightning With Simple Covenants", https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-September/021943.html

[Ultimatum] "Ultimatum Game", https://en.wikipedia.org/wiki/Ultimatum_game

[ZmnSC24] ZmnSCPxj, "SuperScalar: Laddered Timeout-Tree-Structured Decker-Wattenhofer Factories With Pseudo-Spilman Leaves", https://delvingbitcoin.org/t/superscalar-laddered-timeout-tree-structured-decker-wattenhofer-factories-with-pseudo-spilman-leaves/1242

-------------------------

cmp_ancp | 2026-09-15 21:27:50 UTC | #5

Excuse me, but the text is written in a format that is unreadable from the middle on, at least from my cellphone.

-------------------------

