# Disclosure: Memory-exhaustion DoS in Eclair 0.14.0

rorp | 2026-11-09 03:20:57 UTC | #1

# Memory-exhaustion DoS via `start_batch` in Eclair 0.14.0

**Affected:** Eclair 0.14.0

**Fixed in:** Eclair 0.14.1 (\[e4e1a19\](https://github.com/ACINQ/eclair/commit/e4e1a19d913279b3cdea1feefee1ec8372735172))

Any peer that can connect to an Eclair 0.14.0 node can make it buffer an unlimited number of `commitment_signed` messages until it runs out of memory. The attacker doesn't need a channel with the node. Users should upgrade to 0.14.1 or later.

## What `start_batch` is and why it's needed

A channel normally has one funding transaction, so each commitment update needs one `commitment_signed`. During a splice there are also unconfirmed splice transactions, and neither side knows which one will confirm. Every update must be signed for each candidate, so with `N` pending splices the sender sends `N + 1` `commitment_signed` messages.

The receiver has to treat those messages as one logical update. [`start_batch`](https://github.com/lightning/bolts/blob/191462825ae341f0a27098765c075f01ca9bc6c2/02-peer-protocol.md?plain=1#L3043) (type 127) announces them: it carries the `channel_id`, a `u16` `batch_size`, and a `message_type` of 132 (`commitment_signed`). Eclair holds the messages in `PeerConnection` until the batch is complete, then forwards them to the channel as one `CommitSigBatch`.

Because the receiver has to queue these messages, BOLT 2 limits `batch_size` to at most 20, "to protect against excessive queuing that could be abused to DoS receiving nodes". It also [sets a lower bound](https://github.com/lightning/bolts/blob/191462825ae341f0a27098765c075f01ca9bc6c2/02-peer-protocol.md?plain=1#L3071):

> A receiving node:
> - If `batch_size` is not strictly greater than 1:
>   - MUST ignore the `start_batch` message.
>   - SHOULD send a `warning`.

## The bug

`PeerConnection` enforces only the upper bound ([source](https://github.com/ACINQ/eclair/blob/ec8c63b215b98148887237e3f050342a6ca61f4c/eclair-core/src/main/scala/fr/acinq/eclair/io/PeerConnection.scala#L361-L381), condensed):

```scala
case msg: StartBatch =>
  if (!msg.messageType_opt.contains(132)) {
    // ...
  } else if (msg.batchSize > 20) {
    log.debug("ignoring start_batch with batch_size = {} > 20", msg.batchSize)
    d.transport ! Warning(msg.channelId, "invalid start_batch message: batch_size must not be greater than 20")
    stay()
  } else {
    // ...
    stay() using d.copy(commitSigBatch_opt = Some(PendingCommitSigBatch(msg.channelId, msg.batchSize, Nil)))
  }
```

A `batch_size` of 0 is therefore accepted. Each `commitment_signed` that follows for the same `channel_id` is appended to the pending batch ([source](https://github.com/ACINQ/eclair/blob/ec8c63b215b98148887237e3f050342a6ca61f4c/eclair-core/src/main/scala/fr/acinq/eclair/io/PeerConnection.scala#L386-L395)):

```scala
case msg: CommitSig if msg.channelId == pending.channelId =>
  val received1 = pending.received :+ msg
  if (received1.size == pending.batchSize) {
    log.debug("received last commit_sig in batch for channel_id={}", msg.channelId)
    d.peer ! CommitSigBatch(received1)
    stay() using d.copy(commitSigBatch_opt = None)
  } else {
    log.debug("received commit_sig {}/{} in batch for channel_id={}", received1.size, pending.batchSize, msg.channelId)
    stay() using d.copy(commitSigBatch_opt = Some(pending.copy(received = received1)))
  }
```
`received1.size` is always at least 1 and `pending.batchSize` is 0, so the batch never completes and the list keeps growing. Only 0 triggers this, because a `batch_size` of 1 completes on the first message.

Several details make this easy to exploit:

- **No channel is needed:** `PeerConnection` never checks that `channel_id` belongs to a real channel, so the attacker can use any value.
- **Messages can be large:** nothing validates the signatures at this layer, so each message can approach the 65,535-byte limit.
- **There is no backpressure:** each message is [acknowledged to the transport](https://github.com/ACINQ/eclair/blob/ec8c63b215b98148887237e3f050342a6ca61f4c/eclair-core/src/main/scala/fr/acinq/eclair/io/PeerConnection.scala#L357) before it is buffered, so memory grows as fast as the attacker can send data.
- **The buffer lasts as long as the connection:** the attacker only has to answer pings and avoid sending anything that flushes the batch, such as another `start_batch` or a different channel message.
- **Connection limits don't help:** `max-no-channels` caps channel-less inbound connections at 64, but not the memory each one can use.

## Impact

A single connection can exhaust the heap and take the node down. The bug gives an attacker no direct access to funds. The risk is indirect: a Lightning node must be online to react to revoked commitments and to resolve HTLCs before they expire, so repeatedly crashing a node can put its channel funds at risk.

## Fix

Commit [e4e1a19](https://github.com/ACINQ/eclair/commit/e4e1a19d913279b3cdea1feefee1ec8372735172) (#3333), released in [0.14.1](https://github.com/ACINQ/eclair/blob/d769db070424ae0ab6c05975fd7c86dac8b3e1f4/docs/release-notes/eclair-v0.14.1.md), rejects `batch_size <= 1` as the spec requires:

```diff
-            } else if (msg.batchSize > 20) {
-              log.debug("ignoring start_batch with batch_size = {} > 20", msg.batchSize)
-              d.transport ! Warning(msg.channelId, "invalid start_batch message: batch_size must not be greater than 20")
+            } else if (msg.batchSize <= 1 || msg.batchSize > 20) {
+              log.debug("ignoring start_batch with invalid batch_size = {} (max = 20)", msg.batchSize)
+              d.transport ! Warning(msg.channelId, "invalid start_batch message: batch_size must be greater than 1 and not be greater than 20")
               stay()
```

-------------------------

