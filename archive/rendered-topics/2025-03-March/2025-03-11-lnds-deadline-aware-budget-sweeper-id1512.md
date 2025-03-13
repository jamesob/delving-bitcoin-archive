# LND's Deadline-Aware Budget Sweeper

morehouse | 2025-03-11 16:38:50 UTC | #1

*The following is copied verbatim from a [blog post](https://morehouse.github.io/lightning/lnd-deadline-aware-budget-sweeper/) on [morehouse.github.io](http://morehouse.github.io), reproduced here to facilitate discussion.  Special thanks to @yyforyongyu for contributing to this post.*

Starting with [v0.18.0](https://github.com/lightningnetwork/lnd/releases/tag/v0.18.0-beta), LND has a completely rewritten *sweeper* subsystem for managing transaction batching and fee bumping.
The new sweeper uses HTLC deadlines and fee budgets to compute a *fee rate curve*, dynamically adjusting fees (fee bumping) to prioritize urgent transactions.
This new fee bumping strategy has some nice security benefits and is something other Lightning implementations should consider adopting.

# Background

When an unreliable (or malicious) Lightning node goes offline while HTLCs are in flight, the other node in the channel can no longer claim the HTLCs *off chain* and will eventually have to force close and claim the HTLCs *on chain*.
When this happens, it is critical that all HTLCs are claimed before certain deadlines:

- Incoming HTLCs need to be claimed before their timelocks expire; otherwise, the channel counterparty can submit a competing timeout claim.
- Outgoing HTLCs need to be claimed before their corresponding upstream HTLCs expire; otherwise, the upstream node can reclaim them on chain.

If HTLCs are not claimed before their deadlines, they can be entirely lost (or stolen).

Thus Lightning nodes need to pay enough transaction fees to ensure timely confirmation of their commitment and HTLC transactions.
At the same time, nodes don't want to *overpay* the fees, as these fees can become a major cost for node operators.

The solution implemented by all Lightning nodes is to start with a relatively low fee rate for these transactions and then use RBF to increase the fee rate as deadlines get closer.

# RBF Strategies

Each node implementation uses a slightly different algorithm for choosing RBF fee rates, but in general there's two main strategies:

- external fee rate estimators
- exponential bumping

## External Fee Rate Estimators

This strategy chooses fee rates based on Bitcoin Core's (or some other) fee rate estimator.
The estimator is queried with the HTLC deadline as the confirmation target, and the returned fee rate is used for commitment and HTLC transactions.
Typically the estimator is requeried every block to update fee rates and RBF any unconfirmed transactions.

[CLN](https://github.com/ElementsProject/lightning/blob/b5eef8af4db9f2a58f435bb5beb54299b2800e67/lightningd/chaintopology.c#L419-L440) and [LND](https://github.com/lightningnetwork/lnd/blob/f8211a2c3b3d2112159cd119bd7674743336c661/sweep/sweeper.go#L470-L493) prior to v0.18.0 use this strategy exclusively.
[eclair](https://github.com/ACINQ/eclair/blob/95bbf063c9283b525c2bf9f37184cfe12c860df1/eclair-core/src/main/scala/fr/acinq/eclair/channel/publish/ReplaceableTxPublisher.scala#L221-L248) uses this strategy until deadlines are within 6 blocks, after which it switches to exponential bumping.
[LDK](https://github.com/lightningdevkit/rust-lightning/blob/3a5f4282468e6148e592e324c2a72405bdb4b193/lightning/src/chain/package.rs#L1361-L1369) uses a combined strategy that sometimes uses the fee rate from the estimator and other times uses exponential bumping.

## Exponential Bumping

In this strategy, the fee rate estimator is used to determine the initial fee rate, after which a fixed multiplier is used to increase fee rates for each RBF transaction.

[eclair](https://github.com/ACINQ/eclair/blob/95bbf063c9283b525c2bf9f37184cfe12c860df1/eclair-core/src/main/scala/fr/acinq/eclair/channel/publish/ReplaceableTxPublisher.scala#L221-L248) uses this strategy when deadlines are within 6 blocks, increasing fee rates by 20% each block while capping the total fees paid at the value of the HTLC being claimed.
When [LDK](https://github.com/lightningdevkit/rust-lightning/blob/3a5f4282468e6148e592e324c2a72405bdb4b193/lightning/src/chain/package.rs#L1361-L1369) uses this strategy, it increases fee rates by 25% on each RBF.

## Problems

While external fee rate estimators can be helpful, they're not perfect.
And relying on them too much can lead to missed deadlines when unusual things are happening in the mempool or with miners (e.g., increasing mempool congestion, pinning, replacement cycling, miner censorship).
In such situations, higher-than-estimated fee rates may be needed to actually get transactions confirmed.
Exponential bumping strategies help here but can still be ineffective if the original fee rate was too low.

# The Deadline and Budget Aware RBF Strategy

LND's new sweeper subsystem, released in v0.18.0, takes a novel approach to RBFing commitment and HTLC transactions.
The system was designed around a key observation: for each HTLC on a commitment transaction, there are specific *deadline* and *budget* constraints for claiming that HTLC.
The **deadline** is the block height by which the node needs to confirm the claim transaction for the HTLC.
The **budget** is the maximum absolute fee the node operator is willing to pay to sweep the HTLC by the deadline.
In practice, the budget is likely to be a fixed proportion of the HTLC value (i.e. operators are willing to pay more fees for larger HTLCs), so LND's budget [configuration parameters](https://docs.lightning.engineering/lightning-network-tools/lnd/sweeper) are based on proportions.

The sweeper operates by aggregating HTLC claims with matching deadlines into a single batched transaction.
The budget for the batched transaction is calculated as the sum of the budgets for the individual HTLCs in the transaction.
Based on the transaction budget and deadline, a **fee function** is computed that determines how much of the budget is spent as the deadline approaches.
By default, a linear fee function is used which starts at a low fee (determined by the minimum relay fee rate or an external estimator) and ends with the total budget being allocated to fees when the deadline is one block away.
The initial batched transaction is published and a "fee bumper" is assigned to monitor confirmation status in the background.
For each block the transaction remains unconfirmed, the fee bumper broadcasts a new transaction with a higher fee rate determined by the fee function.

The sweeper architecture looks like this:

![LND's sweeper architecture|690x253, 100%](upload://cTeJ85fOpYsxxHktlGr2GHK50kH.png)

For more details about LND's new sweeper, see the [technical documentation](https://github.com/lightningnetwork/lnd/blob/04c76101dd53696dc0c527d1b58eb4647df315d1/sweep/README.md).
In this blog post, we'll focus mostly on the sweeper's deadline and budget aware RBF strategy.

## Benefits

LND's new sweeper system provides greater security against replacement cycling, pinning, and other adversarial or unexpected scenarios.
It also fixed some bad bugs and vulnerabilities present with LND's previous sweeper system.

### Replacement Cycling Defense

Transaction rebroadcasting is a simple mitigation against [replacement cycling attacks](https://bitcoinops.org/en/topics/replacement-cycling/) that has been adopted by all implementations.
However, rebroadcasting alone does not guarantee that such attacks become uneconomical, especially when HTLC values are much larger than the fees Lightning nodes are willing to pay when claiming them on chain.
By setting fee budgets in proportion to HTLC values, LND's new sweeper is able to provide much stronger guarantees that any replacement cycling attacks will be uneconomical.

#### Cost of Replacement Cycling Attacks

With LND's default parameters an attacker must generally spend at least 20x the value of the HTLC to successfully carry out a replacement cycling attack.

Default parameters:

- fee budget: 50% of HTLC value
- CLTV delta: 80 blocks

Assuming the attacker must do a minimum of one replacement per block:

$$
attack\_cost \ge \sum_{t = 0}^{80} fee\_function(t) \\
attack\_cost \ge \sum_{t = 0}^{80} 0.5 \cdot htlc\_value \cdot \frac{t}{80} \\
attack\_cost \ge 20 \cdot htlc\_value
$$

LND also rebroadcasts transactions every minute by default, so in practice the attacker must do ~10 replacements per block, making the cost closer to 200x the HTLC value.

### Partial Pinning Defense

Because LND's new default RBF strategy pays up to 50% of the HTLC value, LND now has a much greater ability to outbid [pinning attacks](https://bitcoinops.org/en/topics/transaction-pinning/), especially for larger HTLCs.
It is unfortunate that significant fees need to be burned in this case, but the end result is still better than losing the full value of the HTLC.

### Reduced Reliance on Fee Rate Estimators

As explained earlier, fee rate estimators are not always accurate, especially when mempool conditions are changing rapidly.
In these situations, it can be very beneficial to use a simpler RBF strategy, especially when deadlines are approaching.
LDK and eclair use exponential bumping in these scenarios, which helps in many cases.
But ultimately the fee rate curve for an exponential bumping strategy still depends heavily on the starting fee rate, and if that fee rate is too low then deadlines can be missed.
The exponential bumping strategy also ignores the value of the HTLC being claimed, which means that larger HTLCs get the same fee rates as smaller HTLCs, even when deadlines are getting close.

LND's budget-based approach takes HTLC values into consideration when establishing the fee rate curve, ensuring that budgets are never exceeded and that HTLCs are never lost before an attempt to spend the full budget has been made.
As such, the budget-based approach provides more consistent results and greater security in unexpected or adversarial situations.

### LND-Specific Bug and Vulnerability Fixes

LND's new sweeper fixed some bad bugs and vulnerabilities that existed with the previous sweeper.

#### Fee Bump Failures

Previously, LND had an inconsistent approach to broadcasting and fee bumping urgent transactions.
In some places transactions would get broadcast with a specific confirmation target and would never be fee bumped again.
In other places transactions would be RBF'd if the fee rate estimator determined that mempool fee rates had gone up, but the *confirmation target* given to the estimator would not be adjusted as deadlines approached.

Perhaps the worst of these fee bumping failures was a [bug](https://github.com/lightningnetwork/lnd/issues/8522) reported by [Carsten Otto](https://github.com/C-Otto), where LND would fail to use the anchor output to CPFP a commitment transaction if the initial HTLC deadlines were far enough in the future.
While this behavior is desirable to save on fees initially, it becomes a major problem when deadlines get closer and the commitment hasn't confirmed on its own.
Because LND did not adjust confirmation targets as deadlines approached, the commitment transaction would remain un-CPFP'd and could fail to confirm before HTLCs expired and funds could be lost.
To make matters worse, the bug was trivial for an attacker to exploit.

LND's sweeper rewrite took the opportunity to correct and unify all the transaction broadcasting and fee bumping logic in one place and fix all of these fee bumping failures at once.

#### Invalid Batching

LND's previous sweeper also sometimes generated invalid or unsafe transactions when batching inputs together.
This could happen in a couple ways:

- Inputs that were invalid or had been double-spent could be batched with urgent HTLC claims, making the whole transaction invalid.
- Anchor spends could be [batched together](https://github.com/lightningnetwork/lnd/issues/8433), thereby violating the CPFP carve out and enabling channel counterparties to pin commitment transactions.

Rather than addressing these issues directly, the previous sweeper would use *exponential backoff* to regroup inputs after random delays and hope for a valid transaction.
If another invalid transaction occurred, longer delays would be used before the next regrouping.
Eventually, deadlines could be missed and funds lost.

LND's new sweeper fixed these issues by being more careful about which inputs could be grouped together and by removing double-spent inputs from transactions that failed to broadcast.

## Risks

The security of a Lightning node depends heavily on its ability to resolve HTLCs on chain when necessary.
And unfortunately proper on-chain resolution can be tricky to get right (see [1](https://morehouse.github.io/lightning/ldk-invalid-claims-liquidity-griefing/), [2](https://morehouse.github.io/lightning/ldk-duplicate-htlc-force-close-griefing/), [3](https://morehouse.github.io/lightning/lnd-excessive-failback-exploit/)).
Making changes to the existing on-chain logic runs the risk of introducing new bugs and vulnerabilities.

For example, during code reviews of LND's new sweeper there were many serious bugs discovered and fixed, ranging from catastrophic [fee function failures](https://github.com/lightningnetwork/lnd/issues/8738) to new [fund-stealing exploits](https://github.com/lightningnetwork/lnd/pull/8514#discussion_r1554270229) and more ([1](https://github.com/lightningnetwork/lnd/pull/8148#discussion_r1542012530), [2](https://github.com/lightningnetwork/lnd/pull/8424#pullrequestreview-1961358576), [3](https://github.com/lightningnetwork/lnd/pull/8422#discussion_r1528832418), [4](https://github.com/lightningnetwork/lnd/issues/8715), [5](https://github.com/lightningnetwork/lnd/issues/8737), [6](https://github.com/lightningnetwork/lnd/issues/8741)).
Node implementers should tread carefully when touching these parts of the codebase and remember that simplicity is often the best security.

# Conclusion

LND's new deadline-aware budget sweeper provides more secure fee bumping in adversarial situations and more consistent behavior when mempools are rapidly changing.
Other implementations should consider incorporating budget awareness into their fee bumping strategies to improve defenses against replacement cycling and pinning attacks, and to reduce reliance on external fee estimators.
At the same time, implementers would do well to avoid complete rewrites of the on-chain logic and instead keep the changes small and review them well.

-------------------------

instagibbs | 2025-03-12 18:54:19 UTC | #2

[quote="morehouse, post:1, topic:1512"]
By setting fee budgets in proportion to HTLC values, LND’s new sweeper is able to provide much stronger guarantees that any replacement cycling attacks will be uneconomical.
[/quote]

Glad to see this common-sense approach which also is additional defense against HTLC expiry bribe attacks. "miner harvesting" is a thing, but so is bribing miners via a timeout tx; you need to make it worth it for a miner one way or another.

-------------------------

ismaelsadeeq | 2025-03-13 10:57:18 UTC | #3

[quote="morehouse, post:1, topic:1512"]
By default, a linear fee function is used which starts at a low fee (determined by the minimum relay fee rate or an external estimator) and ends with the total budget being allocated to fees when the deadline is one block away. The initial batched transaction is published and a “fee bumper” is assigned to monitor confirmation status in the background. For each block the transaction remains unconfirmed, the fee bumper broadcasts a new transaction with a higher fee rate determined by the fee function.
[/quote]


> This fee estimator is called using the deadline as the conf target

It seems that nodes are calling the long-term fee estimator (`conf_target > 2`) while expecting the transaction to confirm as soon as possible.

When you call `estimateSmartFee` with a confirmation target of `n`, you should not expect the transaction to confirm in the next block but rather within `n` blocks.

However, in this case, the transaction is time-sensitive, and you strictly want it to confirm before `n` blocks have elapsed.

Calling `estimateSmartFee` with `n`, and expecting it to confirm ASAP is not not ideal, and would likely not confirm ASAP.
Instead, I they should call `estimateSmartFee` with a `conf_target` of `1`. If the transaction does not confirm within the next 1–2 blocks, they can then call the fee function to determine the fee rate they are willing to pay at that target based on the available budget (curve).

[quote="morehouse, post:1, topic:1512"]
 **Problems**
>While external fee rate estimators can be helpful, they’re not perfect. And relying on them too much can lead to missed deadlines when unusual things are happening in the mempool or with miners (e.g., increasing mempool congestion, pinning, replacement cycling, miner censorship).
[/quote]

See the API proposal for Bitcoin Core’s improved fee estimator response, which provides the current state of the mempool in past with respect to miner's mempool. This information could help clients make more informed decisions: https://github.com/bitcoin/bitcoin/issues/30392#issuecomment-2717491587



cc @ClaraShk

-------------------------

