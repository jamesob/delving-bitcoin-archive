# Disclosure: Crashing CLN with a flood of pings

erickcestari | 2026-08-27 19:07:59 UTC | #1

*The following disclosure is copied verbatim from a [blog post](https://erickcestari.dev/blog/ping-flood-oom/) on [erickcestari.dev](https://erickcestari.dev/), reproduced here to facilitate discussion.*

I found a critical DoS vulnerability in [Core Lightning (CLN)](https://github.com/ElementsProject/lightning) while writing my own BOLT8 implementation. If I flood the node with `ping` messages asking for the largest possible `pong` and then never read the TCP buffer, the node keeps queueing the `pong` messages until it runs out of memory and gets OOM-killed.

## Background

Every Lightning node speaks an encrypted peer-to-peer protocol defined in [BOLT 8](https://github.com/lightning/bolts/blob/152897261850d93c4f4597f39cf22d7d22d6ede6/08-transport.md): messages are framed and encrypted with the [Noise protocol](https://noiseprotocol.org/noise.html), and each frame is at most 65,535 bytes. In [Core Lightning (CLN)](https://github.com/ElementsProject/lightning) this transport lives in a dedicated daemon, `connectd`, which multiplexes one TCP connection per peer and shuffles messages between the peer and the per-channel subdaemons.

Most messages are just forwarded: `connectd` decrypts them and hands them to the subdaemon that owns that channel. But a few never reach a subdaemon at all. `connectd` builds the reply itself, right there on the connection, and sends it back. Those are the ones I'll call locally handled:

* **`ping`** ([BOLT 1](https://github.com/lightning/bolts/blob/152897261850d93c4f4597f39cf22d7d22d6ede6/01-messaging.md)). A `ping` carries a `num_pong_bytes` field. The receiver must reply with a `pong` message padded to exactly that many bytes, up to a maximum of 65,531. It is a liveness probe, and the reply is generated entirely by the receiver.
* **`query_channel_range`** ([BOLT 7](https://github.com/lightning/bolts/blob/152897261850d93c4f4597f39cf22d7d22d6ede6/07-routing-gossip.md)). A gossip query asking "give me the channels in this block range." The receiver builds and streams back a reply covering `first_blocknum` through `first_blocknum + number_of_blocks`. (`query_short_channel_ids` is handled the same way, though a 65,535-byte frame caps how much you can ask for in one request.)

Both are handled inside `connectd`, and in both the *sender* picks how large the *reply* is.

## The Unbounded Output Queue

`connectd` is not supposed to read as fast as a peer can send. The read loop reads **one** message and then parks itself on a wake token, `&peer->peer_in`, instead of immediately reading the next one.

```c
/* Wait for them to wake us */
peer->peer_in_lastmsg = type;
peer->peer_in_lasttime = time_mono();

return io_wait(peer_conn, &peer->peer_in, next_read, peer);
```

It stays asleep until something wakes that token. In the vulnerable code that wake came from the subdaemon side: once `write_to_subd` had drained a subdaemon's queue, it woke the read loop again.

```c
/* Nothing to send? */
if (!msg) {
        ...
        /* Tell them to read again. */
        io_wake(&subd->peer->peer_in);
        ...
}
```

This is textbook backpressure, but the resource it protects is the subdaemon, not the socket. `connectd` reads no faster than the subdaemons consume, which is enough as long as every message the read loop parks on is on its way to one. The locally handled ones are not: `connectd` answers them itself, onto the peer's own outgoing queue, and after answering one the read loop did not park at all.

```c
/* If we swallow this, just try again. */
if (handle_message_locally(peer, decrypted))
        return next_read(peer_conn, peer);   /* read the NEXT message immediately */
```

`next_read` reads again right away. It never touches `&peer->peer_in`, so nothing at all limits how fast a peer can make `connectd` generate replies.

The messages where the peer sizes that reply all sit behind this shortcut:

```c
/* We handle pings and gossip messages. */
static bool handle_message_locally(struct peer *peer, const u8 *msg)
{
        ...
        } else if (type == WIRE_PING) {
                handle_ping_in(peer, msg);
                return true;
        ...
        } else if (type == WIRE_QUERY_CHANNEL_RANGE) {
                handle_query_channel_range(peer, msg);
                return true;
        } else if (type == WIRE_QUERY_SHORT_CHANNEL_IDS) {
                handle_query_short_channel_ids(peer, msg);
                return true;
        ...
}
```

And each `ping` produces a reply that the sender sizes, allocated and enqueued on the outgoing queue:

```c
if (num_pong_bytes < 65532) {
        ignored = tal_arrz(ctx, u8, num_pong_bytes);   /* up to 65,531 bytes */
        *pong = towire_pong(ctx, ignored);
        ...
}
```

Put together, the loop against a non-reading peer becomes:

1. Read a `ping` asking for a 65,531-byte `pong`.
2. Build the `pong`, enqueue it on `peer_outq`.
3. `return next_read(...)`, immediately read the next `ping`. **No wait on the write side.**
4. The peer never reads, so the write side never drains `peer_outq`. But the read loop never checks; it keeps looping as fast as it can decrypt.

The outgoing queue grows without bound. `query_channel_range` with `number_of_blocks` set to `U32::MAX` does the same thing and worse, piling a whole chain's worth of gossip replies onto the same queue from one small request.

## Crashing the Node

The exploit needs no funded channel and no valid gossip. It only needs to complete the Noise handshake and then refuse to read:

1. Alice, the attacker, completes the BOLT 8 handshake with Bob, the victim.
2. Alice sends a flood of `ping` messages, each with `num_pong_bytes = 65531` (or a flood of `query_channel_range` with `number_of_blocks = U32::MAX`).
3. Alice **never reads** from the socket.
4. Bob's `connectd` answers every message, appending a large `pong` (or gossip reply) to the per-peer outgoing queue, and immediately reads the next request because local handling skips the read gate.
5. The queue grows until `connectd` exhausts RAM and swap, and the node is killed by the OOM killer.

In testing on a 2-core VM with 2 GB RAM and 2 GB swap, a single connection was enough to take the node down; a hundred simulated peers did it faster. The victim needs no open channel with the attacker, so the attack surface is every reachable node on the network.

## The Fix

The fix has two parts. The first routes locally handled messages back through the gate: instead of reading the next message immediately, `connectd` nudges the write side and then parks on `&peer->peer_in`, exactly like a subdaemon-bound message.

```c
/* If we swallow this, just try again. */
if (handle_message_locally(peer, decrypted)) {
        /* Make sure to update peer->peer_in_lastmsg so we blame correct msg! */
        io_wake(peer->peer_outq);
        goto out;
}
...
/* Wait for them to wake us */
peer->peer_in_lastmsg = type;
out:
peer->peer_in_lasttime = time_mono();

return io_wait(peer_conn, &peer->peer_in, next_read, peer);
```

That alone would deadlock the connection. The only thing that ever woke `&peer->peer_in` was `write_to_subd`, and a locally handled message never reaches a subdaemon, so the read loop would park and never be woken again. The second part gives the gate a socket-side waker:

```c
if (!msg) {
        /* Tell them to read again, */
        io_wake(&peer->subds);
        io_wake(&peer->peer_in);

        /* Wait for them to wake us */
        return msg_queue_wait(peer_conn, peer->peer_outq, write_to_peer, peer);
}
```

Now the read loop sleeps after queuing a reply and does not read again until the writer has drained `peer_outq` and woken it. Against a peer that never reads, the writer never drains, so reading stalls after a single queued reply. `peer_outq` can no longer grow past roughly one message, and the flood can no longer drive the node to OOM.

It landed as [PR #8525](https://github.com/ElementsProject/lightning/pull/8525).

## Discovery

I was learning BOLT 8 by implementing it, writing my own library for the Noise handshake and transport. Once it could talk to a real node, the obvious next step was to point it at one and see how the P2P layer held up under traffic no well-behaved peer would ever send. Running a CLN node in regtest on a small VM, I scripted 100 peers that completed the handshake and then spammed `ping` messages with `num_pong_bytes = 65531` while deliberately reading nothing back. The node consumed all RAM and swap and was OOM-killed. The same crash reproduced with `query_channel_range` set to the maximum block range, and even with a single peer, which pointed at the real cause: the outgoing per-peer queue was never being bounded when the peer refused to drain it.

## Lessons Learned

Backpressure only counts if every path goes through it. CLN had the gate and the gate worked, but it was coupled to subdaemon delivery, and the handful of messages `connectd` answers by itself never touch a subdaemon. One shortcut around one gate, on the messages where the sender picks the reply size, was enough to OOM the node once the peer stopped draining its socket.

## Timeline

* **2025-08-25:** Vulnerability reported privately to Rusty Russell.
* **2025-09-02:** Fix merged as [PR #8525](https://github.com/ElementsProject/lightning/pull/8525) and released as the last change in Core Lightning `v25.09`.
* **2026-08-25:** Public disclosure.

-------------------------

