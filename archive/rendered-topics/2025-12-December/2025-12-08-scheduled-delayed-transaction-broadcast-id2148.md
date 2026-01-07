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

