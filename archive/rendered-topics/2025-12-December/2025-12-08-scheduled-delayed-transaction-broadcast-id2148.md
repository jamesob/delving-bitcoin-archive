# Scheduled (Delayed) Transaction Broadcast

optout | 2025-12-08 14:46:40 UTC | #1

I’ve came across the feature request for **scheduled transaction broadcast**, and I’ve decided to look into implementing it in Bitcoin Core.

First I’d like to **ask for comments**:

* Any history for this feature, past attempts?
* Any big issues, gotchas, blockers?
* Any current features this may interfere with? I’m aware of broadcasting over temporary connections (PR #29415)

I reproduce below the description of the proposed feature:

==================

Overview. In certain cases the user creating a transaction does not want to broadcast it to the bitcoin network right away, but later in the future. This scheduling can be done on the client side, but sometimes it would be practical if this handled by the node itself.

With Scheduled Transaction Broadcast the bitcoin node accepts a transaction for later broadcast, stores it, and broadcasts automatically at the specified time.

Details
Uses. Delayed transaction broadcast can be useful for/when:

* hiding the real time of transaction creation
* there is some precondition that is expected to occur in the future (e.g. occurrence of a transaction), which condition is not possible or desired to be expressed in bitcoin scripts.

Result, error check. As the actual broadcast happens later, the broadcast result – success or some error – is not available at acceptance time. Only parsing checks are done at acceptance time. The client can check the result only by monitoring the bitcoin network.

Absolute time vs. block time. The target time can be specified as an absolute time or a block height.

No guarantees. Delayed broadcast is best-effort, there is no delivery guarantee.

A new RPC ‘schedulerawtransaction’ is introduced, with syntax very similar to ‘sendrawtransaction’.

Scheduled transactions are kept in a separate memory structure, and are persisted to a separate file (for persistence accross restarts).

Note: I’ve put together a very rough but working prototype.

thx,
–optout

-------------------------

pyth | 2025-12-09 12:29:24 UTC | #2

 I think a particular attention must be pay about anti-snipping, as it can leak information about the time at which the transaction has been crafted.

-------------------------

bruno | 2025-12-09 20:15:20 UTC | #3

[quote="optout, post:1, topic:2148"]
hiding the real time of transaction creation

[/quote]

Hiding the real time of transaction creation could be done on client side, what would it be the real benefit of doing this on the node size?

Also, if you want to broadcast a transaction based on a precondition (e.g. occurrence of a transaction) I wonder how you would implement it on the RPC.

[quote="optout, post:1, topic:2148"]
A new RPC ‘schedulerawtransaction’ is introduced, with syntax very similar to ‘sendrawtransaction’.

[/quote]

I think you would have to implement more RPCs (e.g. to list the scheduled transactions, to cancel a schedule, etc).

-------------------------

optout | 2025-12-10 10:00:43 UTC | #4

Thanks for the remarks!

> Hiding the real time of transaction creation could be done on client side, what would it be the real benefit of doing this on the node size?

Yes, I did start my writeup mentioning that this could be solved on the client side as well. You could solve it by the user (get up in the middle of the night and broadcast manually), or by the client. This requires a client that keeps running. an I haven’t came across a wallet that offers this.
The node capable of scheduled broadcast is probably most useful for the power user scenario, when he has his own node. The node is up continuously, it’s a natural place to host the scheduling logic.

> Also, if you want to broadcast a transaction based on a precondition (e.g. occurrence of a transaction) I wonder how you would implement it on the RPC.

Maybe that was a bit misleading / too general statement. I imagine target time specified in absolute real time terms, or maybe block height. No other conditions. What I meant is a situation that you know that you expect an incoming payment at a certain data, and you want to make a payment after that (of course transactionwise these would be independent). In situations when it’s possible to express the time condition with a timelock on the transaction, then there is no need for scheduling.

[quote="bruno, post:3, topic:2148"]
I think you would have to implement more RPCs (e.g. to list the scheduled transactions, to cancel a schedule, etc).

[/quote]

Correct, I was thinking of that. However, as generally this is a fire-and-forget style usage, the ability to check the result is secondary. If I’m available at the time of broadcast, I can broadcast it myself. But cancelling prematurely could be useful.

-------------------------

arminsdev | 2026-01-22 17:00:01 UTC | #5

> hiding the real time of transaction creation

What is the intended use case here, and is there any privacy benefit? My understanding is that an adversary would still be able to observe the same P2P communication patterns, even if you delay broadcasting your own transactions.

-------------------------

optout | 2026-01-23 15:32:50 UTC | #6

The main motivation is privacy indeed.

There are several levels of onchain information sources: information readily available e.g. from a chain explorer, information available from a full node, information available from several scattered probing nodes. Each level requires more resources, is more difficult to achieve, but adds more information, such as first-seen times or triangulated origin.

A typical use case to think about is transfer of multiple UTXOs from a wallet to another, in separate transactions (in order to avoid linking them). If it’s done at almost the same time, the fact that those transactions happen very close in time, combined with some other heuristics (similar structure, similar propagation, potentially some already linked, etc). can be used to link them with high probability. A random delay can help a lot here. But doing that manually is quite cumbersome; this feature would alleviate that.

-------------------------

ArmchairCryptologist | 2026-01-24 09:09:58 UTC | #7

I think the biggest gotcha here is that nLockTime will leak the time of transaction creation time with high confidence. For Core in particular, there is a 10% chance that transactions have a nLockTime set to a random value within 100 blocks of the current height (see DiscourageFeeSniping in wallet/spend.cpp), but that still leaves a 90% chance for a transaction to reveal the true block height at the time it was created, and you will generally be able to tell it’s a delayed transaction if nLockTime is older than 17 hours or so. So I’m not sure this really is more useful than just doing `sleep 3600; ./bitcoin-cli sendrawtransaction…`

I suppose you could set nLockTime to 0 for any transactions you want to do this for, or set it to the block height you want to send the transaction at. Unfortunately, Bitcoin Core does not provide this functionality in most transaction creation functions, which makes this somewhat complicated - I believe you need to use `createrawtransaction` or `walletcreatefundedpsbt`to do that.

I could see this being useful if a) it was easier to create a transaction with a provided nLockTime, and b) the schedule function had the ability to send the transaction at the corresponding block height rather than (or in addition to) at a given time.

-------------------------

optout | 2026-01-26 08:27:33 UTC | #8

In the target use case here the transaction is not created by BitcoinCore, but a wallet application connected to BitcoinCore. If a wallet knows about the scheduling, and if it sets nLocktime, it can set it to around the scheduled time (not created time). So I don’t see a real concern here.

[quote="ArmchairCryptologist, post:7, topic:2148"]
So I’m not sure this really is more useful than just doing `sleep 3600; ./bitcoin-cli sendrawtransaction…`

[/quote]

Yes, that’s the purpose of this feature, to achieve that, but in a way usable when the user does not want to have a running process for the period, nor he has access to RPC directly.

The option to specify the target time in block height (not physical time) is certainly possible, though it doesn’t really make much difference (the two are quite interchangeable in the few-hours-to-few-days range).

-------------------------

