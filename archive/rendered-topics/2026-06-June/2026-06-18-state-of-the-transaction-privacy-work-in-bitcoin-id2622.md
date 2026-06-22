# State of the transaction privacy work in Bitcoin

nkaretnikov | 2026-06-18 21:27:51 UTC | #1

I would like to better understand what's happening around transaction privacy in Bitcoin.

My perception is that it's all happening in sidechains, L2s, or requires wallet support.
Examples: Confidential Transactions (sidechain), Silent Payments (wallet), PayJoin (wallet), CoinJoin (wallet).
There is also related work, like Cross-Input Signature Aggregation, that would make CoinJoin cheaper, but isn't a privacy feature by itself.

I assume people moved on from L1 privacy work for several reasons:

* Lightning is popular, and offers some privacy that many consider sufficient
* Implementation complexity, plus a focus on transparency and a fixed supply (see past inflation/counterfeiting bugs in privacy-coin implementations)
* Concerns about regulatory pressure if transactions become untraceable.

Is this now mostly a wallet-layer concern, or are there protocol-level gaps worth filling (like CISA)?

Anything else with a realistic shot at adoption that I've missed?

-------------------------

nkaretnikov | 2026-06-20 19:58:17 UTC | #2

It seems there's ongoing work on CoinSwap.

Summary of the protocol:
https://opensats.org/blog/developing-advancements-in-onchain-privacy
https://opensats.org/topics/coinswap

An actively developed implementation:
https://github.com/citadel-tech/coinswap
https://github.com/citadel-tech/coinswap/milestone/1

Protocol spec:
https://github.com/citadel-tech/Coinswap-Protocol-Specification
https://bitcointalk.org/index.php?topic=321228.0

An older PoC from Chris Belcher:
https://github.com/bitcoin-teleport/teleport-transactions

-------------------------

nkaretnikov | 2026-06-20 20:23:14 UTC | #3

Physical devices that allow you to pass Bitcoin offline, like cash. Not a protocol, but worth mentioning.
https://opendime.com/
https://satscard.com/

-------------------------

light | 2026-06-20 20:24:25 UTC | #4

Shielded CSV: https://eprint.iacr.org/2025/068 

Needs a soft fork that supports building a trustless bridge (e.g. BIP-441) in order to be used trustlessly with BTC.

-------------------------

cmp_ancp | 2026-06-20 20:54:41 UTC | #5

There is an interesting project concerning coinjoin coordination in nostr, making it descentralized and avoiding regulatory pressures.

joinstr https://share.google/MhV0i0CeMvv1x0RnB

Also, you commented about LN superficially, but there are different concepts and protocols inside LN that aren't broad used or implemented today and can bring more privacy in the future. Taproot channels, PTLCs, maybe a gossip that doesn't doxx UTXOs, LN over Ark, blinded payments over BOLT12 (maybe the most important BOLT12 feature in respect to privacy and one of the least implemented among wallets), trampoline nodes, splices, LSP spec, neutrino and filter based sync tech, etc.

LN over Ark may be the UX killer, resolving the inbound liquidity problem and cheapenning multiple channel maintanance (something that increases privacy).

-------------------------

nkaretnikov | 2026-06-21 00:38:32 UTC | #6

PathCoin:
https://gist.github.com/AdamISZ/b462838cbc8cc06aae0c15610502e4da
https://github.com/AdamISZ/pathcoin-poc

-------------------------

Kruw | 2026-06-21 21:18:39 UTC | #7

[quote="nkaretnikov, post:1, topic:2622"]
My perception is that it's all happening in sidechains, L2s, or requires wallet support. Examples: Confidential Transactions (sidechain), Silent Payments (wallet), PayJoin (wallet), CoinJoin (wallet).

[/quote]

Coinjoin does not require wallet support in the same way as Silent Payments or Payjoins. Receivers of coinjoin payments can use any wallet, while SPs and PJs require software support for both the sender and receiver.

Coinjoin is the only transaction type out of your list that provides full privacy. The others leak information:

**- Payjoins only hide the common input ownership heuristic from third party observers.** It does not prevent the sender or receiver from tracking each other's addresses. Unfortunately, PJ actually introduces a privacy footgun: The receiver shares one of their existing UTXOs with the sender to merge with the payment. This gives additional data to the sender that would not necessarily be revealed in a later consolidation.

* Alice pays Bob's bc1paddressx123
* Charlie payjoins with Bob, who reveals he previously received coins at bc1paddressx123
* Bob now has bc1paddressx456 which is a consolidation of Alice and Charlie's payments
* Charlie has now become a custodian of Bob's secret data (The origin address and value of Alice's payment)

Worse, if Charlie is using an Electrum based mobile wallet to payjoin, Bob's linked address data also gets [harvested by third parties.](https://x.com/Kruwed/status/2062195325561938295)

**- Silent Payments only generate new receiver addresses automatically,** it's mainly a UX improvement for donations. Unfortunately, SP addresses actually introduce a new privacy footgun: A receiver who shares the same sp1... address with different senders leaks their common on chain identity with off chain data.

* Alice pays Bob's sp1addressx123 (Robert's pseudonym)
* Charlie pays Robert's sp1addressx123 (Bob's real name)

Even if Bob is careful not to merge these inputs in future payments, Alice and Charlie can each compare the silent payment addresses they have previously sent to and reveal that Robert's pseudonym is actually Bob.

Worse, this leak occurs passively if Alice and Charlie both use the same custodian. The custodian is able to correlate payment relationships between Alice, Bob, and Charlie even if Alice and Charlie don't know each other.

**- Confidential Transactions only hide the value of coins being transferred.** It does not prevent the sender or receiver from tracking each other's addresses using common input ownership or change heuristics.

[quote="nkaretnikov, post:1, topic:2622"]
There is also related work, like Cross-Input Signature Aggregation, that would make CoinJoin cheaper, but isn't a privacy feature by itself.

[/quote]

Similarly, confidential transactions also eliminates the higher relative cost of coinjoins compared to regular payments since creation of even-denomination outputs would no longer be necessary.

-------------------------

nkaretnikov | 2026-06-21 21:24:14 UTC | #8

I think there are some inaccuracies in what you said. Please correct me if I'm wrong.

[quote="Kruw, post:7, topic:2622"]
Coinjoin is the only transaction type out of your list that provides full privacy.
[/quote]

"Full privacy" is an overstatement. CoinJoin is a mixing scheme. If the anonymity set is not good, then privacy is not good. It can be vulnerable to amount correlation, post-mix consolidation, address reuse, and timing/Sybil attacks.

[quote="Kruw, post:7, topic:2622"]
Unfortunately, PJ actually introduces a privacy footgun: The receiver shares one of their existing UTXOs with the sender to merge with the payment.
[/quote]

This is a valid observation, but PayJoin still can be used to improve privacy if:
- Bob uses a UTXO that doesn't deanonymize a different counterparty (e.g., Bob's own transfer from an exchange)
- Bob uses a UTXO already associated with Charlie.

This breaks the common-input-ownership heuristic, obscures the payment amount, and can mislead change-identification heuristics.

The example you provide is the worst case example. It's not a PayJoin issue, but rather a UX issue with UTXO management and privacy. But it's a fair point for real world usage. So I'm glad you pointed it out.

[quote="Kruw, post:7, topic:2622"]
Worse, if Charlie is using an Electrum based mobile wallet to payjoin, Bob’s common input ownership data also gets [harvested by third parties.](https://x.com/Kruwed/status/2062195325561938295)
[/quote]

Not a PayJoin issue, but an important point for real world usage. Appreciate you mentioning this.

[quote="Kruw, post:7, topic:2622"]
**Silent Payments just generate new receiver addresses automatically,** it’s mainly a UX improvement for donations.
[/quote]

The word "just" makes it seem like a convenience feature. Making it easier to use the same address without linking transactions is huge for privacy. As a side effect, this also reduces public key exposure for unspent outputs. Exposed public keys can become an issue with quantum computers.

-------------------------

Kruw | 2026-06-22 00:19:42 UTC | #9

[quote="nkaretnikov, post:8, topic:2622"]
"Full privacy" is an overstatement. CoinJoin is a mixing scheme. If the anonymity set is not good, then privacy is not good. It can be vulnerable to amount correlation, post-mix consolidation, address reuse, and timing/Sybil attacks.

[/quote]

These issues are solvable when using the WabiSabi coinjoin protocol.

Papers measuring coinjoin data can be found here, which shows that the anonymity set continuously increases over time from remixes:

[Analysis of input-output mappings in coinjoin transactions with arbitrary values by Jiri Gavenda, Petr Svenda, Stanislav Bobon, and Vladimir Sedlacek (Masaryk University, Czechia)](https://arxiv.org/pdf/2510.17284)

[CoinJoin ecosystem insights for Wasabi 1.x, Wasabi 2.x and Whirlpool coordinator-based privacy mixers by Petr Svenda + Jiri Gavenda (Masaryk University, Czechia) and Vasilios Mavroudis + Chris Hicks (The Alan Turing Institute)](https://petsymposium.org/popets/2026/popets-2026-0061.pdf)

[quote="nkaretnikov, post:8, topic:2622"]
This is a valid observation, but PayJoin still can be used to improve privacy if: Bob uses a UTXO that doesn't deanonymize a different counterparty (e.g., Bob's own transfer from an exchange)

[/quote]

Changing Alice's name to MtGox in the example doesn't make any difference, an exchange is still a counterparty.

[quote="nkaretnikov, post:8, topic:2622"]
Bob uses a UTXO already associated with Charlie. This breaks the common-input-ownership heuristic, obscures the payment amount, and can mislead change-identification heuristics.

[/quote]

Those heuristics are broken from a third party perspective, but using Payjoin in this scenario doesn't make any difference for the conventional threat model: Bob can already consolidate these two UTXOs without revealing any new information to Charlie.

[quote="nkaretnikov, post:8, topic:2622"]
Not a PayJoin issue, but an important point for real world usage. Appreciate you mentioning this.

[/quote]

It is a Payjoin issue - If Bob accepts a regular payment from Charlie's mobile wallet instead of a payjoin, less data will be revealed to Charlie (and end up in the hands of surveillance companies).

[quote="nkaretnikov, post:8, topic:2622"]
The word "just" makes it seem like a convenience feature. Making it easier to use the same address without linking transactions is huge for privacy.

[/quote]

Silent payments have no privacy advantages compared to a payment sent to a new address that was generated manually. It's strictly a convenience improvement that allows reuse in two specific scenarios:

1. Receiving non-public donations
2. Receiving multiple payments from the same entity.

[quote="nkaretnikov, post:8, topic:2622"]
As a side effect, this also reduces public key exposure for unspent outputs. Exposed public keys can become an issue with quantum computers.

[/quote]

This is incorrect. Silent payment outputs use Taproot keys, which are exposed onchain.

-------------------------

AdamISZ | 2026-06-22 11:21:16 UTC | #10

[quote="Kruw, post:9, topic:2622"]
These issues are solvable when using the WabiSabi coinjoin protocol.
[/quote]

Seems way overstated. I agree with @nkaretnikov . First, you cannot solve the problem with a central server: the central server has an unstoppable ability to near-fully Sybil you in any given join. It's true that that risk is not as bad as it seems at first glance, given all the "natural" circumstantial defenses that exist (in particular probing the server with anonymous entities), but it always exists to some extent or other.

Then there's the fact that for actually *using* bitcoin, denomination matters. WabiSabi is a huge leap forward from Wasabi 1 in that respect, it's a great innovation, but, transactions that make actual payments are not close to fully solved with this denomination splitting [1].

Then there's the fact that coinjoin at big anon sets simply does not scale (and at small anon sets does not give good anonymization ofc).

Where I do think you have a very solid basis for the gist of your claim is: large enough anon sets (as we have seen with recent Wasabi) means you push outside feasible computation limits; the coinjoins do really become a black box (in general; there can always be specific failures ofc). That's an important observation but as per above (1) it can't scale far and (2) it doesn't remove the other problems (*) with onchain transactions, especially if people don't pay each other inside the joins.

(*) timing correlation, amount correlation, wallet fingerprinting, network level trace and a bunch of other fingerprints, combined with the unreasonable effectiveness element from set-intersection, make it obviously false to claim any onchain privacy technique, even the best one, is going to let you say "solved".

Offchain it's much more defensible to say "solved", I think.

[1] Little side note: as many of us who studied coinjoin over the years, I became very interested in the 'one participant pays other inside the coinjoin' model; it counters the unscalability of 'useless extra transactions', it breaks surveillance of the flows etc. But it is not just ironic but telling, that exactly this model is the one where you lose the property you noted upthread: it means the receiver now *does* need new software.

-------------------------

Kruw | 2026-06-22 14:52:16 UTC | #11

[quote="AdamISZ, post:10, topic:2622"]
Seems way overstated.

[/quote]

"Solvable" is not an overstatement imo, I tried to choose my words carefully here so I don't spread too much optimism on a technical board.

The claim "These issues are **solved** when using the WabiSabi coinjoin protocol" would be inaccurate since there's still work to do on improving the current implementation. The claim "These issues are solvable when **participating in a WabiSabi coinjoin transaction**" would also be inaccurate because a single transaction is not guaranteed to enhance privacy. Whales face a relative disadvantage compared to smaller participants, so they will need to pay for and wait for additional remixes.

[quote="AdamISZ, post:10, topic:2622"]
First, you cannot solve the problem with a central server: the central server has an unstoppable ability to near-fully Sybil you in any given join.

[/quote]

You are somewhat correct, WabiSabi assumes server availability as a premise:

![575447450-81ba6df2-da81-4333-b87f-631a5e2feb9b|690x172](upload://ekWMrns7xJ8UhQaRyUqwJlO0Ka6.png)

An unstable server can slowly partition users by dropping connections and preventing rounds from ever completing. Each failed round leaks some metadata off chain, and if the server disappears permanently then there's extra friction while users switch to a new coordinator.

As the paper mentions, the incentives for clients and servers are already aligned since coordinators may profit from successful rounds: My coordinator server has a higher % uptime than both Zcash's Orchard pool and Litecoin's MWEB chain.

However, I would argue that the WabiSabi protocol is uniquely resistant to Sybil attacks, for the exact  reason you mentioned:

[quote="AdamISZ, post:10, topic:2622"]
It's true that that risk is not as bad as it seems at first glance, given all the "natural" circumstantial defenses that exist (in particular probing the server with anonymous entities), but it always exists to some extent or other.

[/quote]

Expanding my above claim with a comparison to other coinjoin protocols:

* Whirlpool is uniquely vulnerable to Sybil attacks: The coordinator chooses all inputs for the round.
* JoinMarket defends takers against Sybil attacks with poodl.
* WabiSabi coordinators can't execute a Sybil attack against a target input without being detected since the user would notice their other inputs are not included in the round. Excluded users would migrate their liquidity to a new coordinator, and the malicious one would die off.

[quote="AdamISZ, post:10, topic:2622"]
Then there's the fact that for actually *using* bitcoin, denomination matters. WabiSabi is a huge leap forward from Wasabi 1 in that respect, it's a great innovation, but, transactions that make actual payments are not close to fully solved with this denomination splitting \[1\].

[/quote]

Wasabi v2.8 is yet another huge leap forward over previous implementations of WabiSabi because it adds the Pay in Coinjoin feature to the GUI:

![HKKAnF-WsAA9UEV|613x500](upload://78XANg6wZJghTe1BDKNUQ0Wimzs.png)

This tactic eliminates multiple privacy concerns:

* The age of the sender's inputs is not revealed
* The number of inputs consolidated by the sender to make the payment is not revealed
* The origin rounds of inputs consolidated by the sender are not revealed
* The value of the sender's leftover change is not revealed
* The sender can batch multiple payments without revealing a common origin

Even newbies who don't understand the concept of UTXOs don't need any further instruction to use Bitcoin anonymously (pending UX issue[\*](https://github.com/WalletWasabi/WalletWasabi/issues/14664)).

[quote="AdamISZ, post:10, topic:2622"]
Then there's the fact that coinjoin at big anon sets simply does not scale

[/quote]

The base layer simply does not scale in general. Kukks made a prototype for sending anonymous coinjoin credentials directly to the end user, turning the coinjoin round into a non custodial ecash layer 2: [Kompaktor](https://github.com/Kukks/Kompaktor)

[quote="AdamISZ, post:10, topic:2622"]
That's an important observation but as per above (1) it can't scale far and (2) it doesn't remove the other problems (\*) with onchain transactions, especially if people don't pay each other inside the joins.

[/quote]

Even ignoring the potential benefits of CISA and CT, coinjoins can be scaled reasonably. My average coinjoin size over the past year was 270 inputs, 325 outputs, 30k vBytes. The largest coinjoin was 498 inputs, 51k vBytes. Considering the standard relay limit for individual transactions is 100k vBytes, there's relatively little room left for optimization.

100k isn't the scaling limit though. After a round becomes full, coordinators optimize the liquidity by running two rounds simultaneously: One for whales, and one for smaller users. When clients sort themselves by relative value, you can squeeze out even more performance.

[quote="AdamISZ, post:10, topic:2622"]
\[1\] Little side note: as many of us who studied coinjoin over the years, I became very interested in the 'one participant pays other inside the coinjoin' model; it counters the unscalability of 'useless extra transactions', it breaks surveillance of the flows etc. But it is not just ironic but telling, that exactly this model is the one where you lose the property you noted upthread: it means the receiver now *does* need new software.

[/quote]

Yes, JoinMarket benefits strongly if the recipient is also a JoinMarket user. A trail of consecutive payjoin transactions also gradually decays possible analysis.

Also, the downsides of payjoin are eliminated if both participants are using coinjoined outputs for their payjoin inputs.

-------------------------

