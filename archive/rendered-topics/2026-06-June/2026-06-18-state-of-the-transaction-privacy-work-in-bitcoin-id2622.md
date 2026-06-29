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

Kruw | 2026-06-22 19:36:26 UTC | #11

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
* JoinMarket defends takers against Sybil attacks with fidelity bonds.
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

AdamISZ | 2026-06-22 15:02:32 UTC | #12

[quote="Kruw, post:11, topic:2622"]
“Solvable” is not an overstatement imo, I tried to choose my words carefully here so I don’t spread too much optimism on a technical board.

The claim “These issues are **solved** when using the WabiSabi coinjoin protocol” would be inaccurate since there’s still work to do on improving the current implementation.
[/quote]

Funnily enough I was about to write a much more agreeing-with-you post where I pointed out that you'd said "solvable" and not "solved" :laughing: ... before deciding that no, it's too pedantic and it ultimately is not realistic to say it's "in the process of being solved". Your counterpoints are noted - good information - but I still think you're significantly overstating with the term "solvable". Disagree.

-------------------------

Kruw | 2026-06-22 15:32:22 UTC | #13

[quote="AdamISZ, post:12, topic:2622"]
Your counterpoints are noted - good information - but I still think you're significantly overstating with the term "solvable". Disagree.

[/quote]

Could you be more specific about what your remaining concerns are? Are they privacy related or scaling related?

-------------------------

craigraw | 2026-06-23 06:05:26 UTC | #14

[quote="Kruw, post:7, topic:2622"]
**Silent Payments only generate new receiver addresses automatically,** it's mainly a UX improvement for donations... Alice pays Bob's sp1addressx123 (Robert's pseudonym), Charlie pays Robert's sp1addressx123 (Bob's real name)

[/quote]

This is at best a confused analysis which misses the fact that it is trivial to create a second account (e.g. m/352'/0'/'1) with a second and unlinkable silent payments address for Bob.

-------------------------

Kruw | 2026-06-23 08:15:33 UTC | #15

[quote="craigraw, post:14, topic:2622"]
This is at best a confused analysis which misses the fact that it is trivial to create a second account (e.g. m/352'/0'/'1) with a second and unlinkable silent payments address for Bob.

[/quote]

You seem even more confused: Generating a new unlinkable address is already the status quo for privacy, the point of a silent payment address is that **you can reuse it.** In practice, you can only reuse it for two specific scenarios:

[quote="Kruw, post:9, topic:2622"]
1. Receiving non-public donations
2. Receiving multiple payments from the same entity.

[/quote]

-------------------------

craigraw | 2026-06-23 09:40:30 UTC | #16

You are conflating two completely different things - having two entities with two static reusable addresses, and the status quo of needing a new onchain address for every payment to maintain privacy.

If you go to the trouble of creating a pseudonym, creating a second unlinkable static address is a trivial step. That second static address can then privately receive an unlimited number of payments from any number of different entities. In order words, address reuse with privacy.

I'm not sure why you claim this is only useful for receiving multiple payments from the same entity. This is untrue.

-------------------------

Kruw | 2026-06-23 10:25:01 UTC | #17

[quote="craigraw, post:16, topic:2622"]
If you go to the trouble of creating a pseudonym, creating a second unlinkable static address is a trivial step.

[/quote]

The fact that generating a new address is trivial is one that undermines the argument for silent payments, not strengthens it. As I already said: The current status quo is that every transaction gets its own pseudonym.

[quote="craigraw, post:16, topic:2622"]
That second static address can then privately receive an unlimited number of payments from any number of different entities.
...
I'm not sure why you claim this is only useful for receiving multiple payments from the same entity. This is untrue.

[/quote]

It is true, I already explained why this behavior is risky:

[quote="Kruw, post:7, topic:2622"]
Worse, this leak occurs passively if Alice and Charlie both use the same custodian. The custodian is able to correlate payment relationships between Alice, Bob, and Charlie even if Alice and Charlie don't know each other.

[/quote]

-------------------------

craigraw | 2026-06-23 10:42:57 UTC | #18

[quote="Kruw, post:17, topic:2622"]
The fact that generating a new address is trivial is one that undermines the argument for silent payments, not strengthens it.

[/quote]

This is just nonsense.

> Worse, this leak occurs passively if Alice and Charlie both use the same custodian.

Well, of course if a single entity has a complete view of all the transaction history for all parties they can see payment relationships. Stating this as a risk for Silent Payments is nonsensical. If a single entity has all the transaction history for all parties using coinjoin wallets the same fact is true.

-------------------------

Kruw | 2026-06-23 10:49:58 UTC | #19

[quote="craigraw, post:18, topic:2622"]
This is just nonsense.

[/quote]

What is "nonsense"? You just spent the past 2 posts repeating that generating new addresses is trivial, and I agreed with you: It is trivial.

[quote="craigraw, post:18, topic:2622"]
Well, of course if a single entity has a complete view of all the transaction history for all parties they can see payment relationships.

[/quote]

Again, you are confused: **Bob is not a user of the custodian in the scenario.** If Bob uses regular addresses, the custodian does not know that Alice and Charlie sent payments to the same person. If Bob uses a silent payment address, the custodian learns his common input ownership information immediately, even if the UTXOs are never consolidated in a later payment.

-------------------------

craigraw | 2026-06-23 11:00:35 UTC | #20

[quote="Kruw, post:19, topic:2622"]
If Bob uses regular addresses, the custodian does not know that Alice and Charlie sent payments to the same person. If Bob uses a silent payment address, the custodian learns his common input ownership information immediately

[/quote]

Ah, I see your point more clearly now, but there is still a fundamental misunderstanding. The addresses that Bob receives on can be calculated locally by the wallet software running on Alice/Charlie's computer - there is no need for the custodian to have any record of the silent payments addresses.

In any case, all this rather misses the point. Address reuse is currently near 70% because it's difficult to share/validate a new address for every payment. Silent Payments removes this problem, and if you can't see the difference between a new address for every payment and a new address for every persona, I don't think we are ever going to agree.

-------------------------

Kruw | 2026-06-23 11:11:06 UTC | #21

[quote="craigraw, post:20, topic:2622"]
The addresses that Bob receives on can be calculated locally by the wallet software running on Alice/Charlie's computer

[/quote]

Wouldn't this require the custodian to first share the UTXO(s) they will use for the silent payment with Alice/Charlie so they can calculate the end destination address?

[quote="craigraw, post:20, topic:2622"]
it's difficult to share/validate a new address for every payment.

[/quote]

This is the exact opposite of the claim you made multiple times:

[quote="craigraw, post:14, topic:2622"]
it is trivial to create a second account

[/quote]

[quote="craigraw, post:16, topic:2622"]
creating a second unlinkable static address is a trivial step.

[/quote]

-------------------------

craigraw | 2026-06-23 11:24:09 UTC | #22

[quote="Kruw, post:21, topic:2622"]
Wouldn't this require the custodian to first share the UTXO(s) they will use for the silent payment with Alice/Charlie so they can calculate the end destination address?

[/quote]

Indeed, that is necessary for wallet software to create a signed transaction.

> This is the exact opposite of the claim you made multiple times:

No, I said it is trivial to generate a new address. I did not say it is trivial to share it, or have the counterparty validate it as correct. This is the part you are missing. With Silent Payments, you can do this once per persona. With HD wallets, you have to do this once per payment - and that makes all the difference. This is borne out in the data - because it is so difficult/inconvenient, address reuse is prevalent. With 70% address reuse, the current reality of your example is that Alice and Charlie are sending to the same onchain address for Bob, creating a cryptographic link visible to everyone - strictly worse than the Silent Payments alternative.

-------------------------

Kruw | 2026-06-23 12:13:40 UTC | #23

[quote="craigraw, post:22, topic:2622"]
Indeed, that is necessary for wallet software to create a signed transaction.

[/quote]

I highly doubt any custodian would implement this since it is probing mechanism that worsens the custodian's privacy.

[quote="craigraw, post:22, topic:2622"]
I did not say it is trivial to share it, or have the counterparty validate it as correct.

[/quote]

With the exception of donations, you already have to be in contact with the counterparty to negotiate the terms of the transaction.

Psychologically, I would hesitate more to send a donation to a SP address compared to a BTCPay generated address because I don't know if the recipient still has the keys for the SP address (which does not encode an expiration date).

[quote="craigraw, post:22, topic:2622"]
With Silent Payments, you can do this once per persona. With HD wallets, you have to do this once per payment - and that makes all the difference.

[/quote]

This functionality is already possible by just sharing a new xpub with each sender, it doesn't have to be a new SP address. Even still, this marginal UX benefit is risky compared to generating a new address for each payment: I personally check to see if my existing BTC is still in my wallet before accepting more coins. Things like automated weekly payroll are a disaster if the recipient's keys are stolen during the week.

-------------------------

craigraw | 2026-06-23 12:24:21 UTC | #24

> With the exception of donations, you already have to be in contact with the counterparty to negotiate the terms of the transaction.

Many (most?) transactions involve repeat payments where the amount is already determined. This is one of the reasons address reuse is so high - negotiating a new onchain address for each repeat payment is simply too inconvenient.

> This functionality is already possible by just sharing a new xpub with each sender, it doesn't have to be a new SP address.

Sharing an SP address per persona is much better than sharing an xpub per sender, both in terms of privacy and convenience. 

[quote="Kruw, post:23, topic:2622"]
Psychologically, I would hesitate more to send a donation to a SP address compared to a BTCPay generated address because I don't know if the recipient still has the keys for the SP address... Even still, the marginal benefit is still risky compared to generating a new address for each payment

[/quote]

These arguments remind me of the objections to HD wallets back in the day (xpub-as-total-exposure privacy problem, single-point-of-failure critique of the seed etc) and miss the fact that benefits vastly exceed the possible pitfalls. This is a privacy thread, and if you fail to acknowledge that the current approach is leading to high address reuse then you fail to acknowledge reality.

-------------------------

Kruw | 2026-06-23 12:49:19 UTC | #25

[quote="craigraw, post:24, topic:2622"]
Many (most?) transactions involve repeat payments where the amount is already determined.

[/quote]

...Citation? What data set exists to derive this estimation from?

[quote="craigraw, post:24, topic:2622"]
This is a privacy thread, and if you fail to acknowledge that the current approach is leading to high address reuse then you fail to acknowledge reality.

[/quote]

The reality is that a lot of people use Electrum based light wallets and every address gets completely exposed to chain surveillance anyway. Generating new addresses doesn't even begin to benefit the user unless their wallet is a BIP157/158 light client, or a full node.

-------------------------

AdamISZ | 2026-06-29 15:15:47 UTC | #26

Oh, I forgot to add in this thread: PathCoin is not relevant to the discussion, as it was only a theoretical concept that might push towards some interesting offline protocol (it *could* be relevant to privacy if it was practically usable, but it wasn't, it was no more than a "starting idea for future research", really).

-------------------------

