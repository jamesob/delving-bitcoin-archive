# CLN: Self-Referential Channel Announcement DoS disclosure

NishantBansal2003 | 2026-10-24 10:00:27 UTC | #1

> The issue discussed below was originally disclosed at [nishantbansal2003.github.io](https://nishantbansal2003.github.io/posts/CLN-Self-Referential-Channel-Announcement-DoS/) and is reproduced here for discussion.

Core Lightning _v25.02_ through _v26.04_ crash on a `channel_announcement` whose `node_id_1` and `node_id_2` are the same node.
CLN never checks that the two endpoints differ, so `gossipd` adds the same node to its routing graph twice, trips a no-duplicates assertion, and aborts, taking the whole node down.
Because the announcement is written to disk **before** it is loaded, `gossipd` reloads it and aborts again on **every restart**, so the node stays offline until the operator removes the record by hand.
The attacker needs only an unspent 2-of-2 output it controls, so the attack costs nothing beyond on-chain fees.

Upgrade to [Core Lightning v26.04.1](https://github.com/ElementsProject/lightning/releases/tag/v26.04.1) or later.

## Background

Nodes learn the channel graph from [BOLT 7](https://github.com/lightning/bolts/blob/191462825ae341f0a27098765c075f01ca9bc6c2/07-routing-gossip.md) gossip.
A `channel_announcement` identifies a channel by its `short_channel_id` and names its two endpoints, `node_id_1` and `node_id_2`.
It carries four signatures: one from each node, and one from each of the two keys, `bitcoin_key_1` and `bitcoin_key_2`, in the channel's 2-of-2 funding output.
BOLT 7 [requires](https://github.com/lightning/bolts/blob/a3772650d8ebc06acf457fcadf97968ebfc4dfff/07-routing-gossip.md?plain=1#L189-L191) the sender to order the node IDs so that `node_id_1` is the lexicographically lesser, which implies they differ, but until this report it placed no matching requirement on the receiver.

CLN's `gossipd` validates each announcement, appends it to an on-disk `gossip_store`, and then loads it into the `gossmap`, an in-memory routing graph that records each node once, keyed by its node ID.

## The vulnerability

When CLN v26.04 receives a `channel_announcement`, [`gossmap_manage_channel_announcement()`](https://github.com/ElementsProject/lightning/blob/6f9356fb9369d34811640f292c75cc3b7e43e9d1/gossipd/gossmap_manage.c#L620-L732) parses it and verifies its signatures, but never checks that the two node IDs differ.
A signed announcement with equal node IDs, backed by an unspent 2-of-2 output, therefore passes validation and is [written to the `gossip_store`](https://github.com/ElementsProject/lightning/blob/6f9356fb9369d34811640f292c75cc3b7e43e9d1/gossipd/gossmap_manage.c#L829).

`gossipd` then loads it into the `gossmap` with [`add_channel()`](https://github.com/ElementsProject/lightning/blob/6f9356fb9369d34811640f292c75cc3b7e43e9d1/common/gossmap.c#L451-L520), which assumes the two node IDs differ:

```c
static struct gossmap_chan *add_channel(struct gossmap *map, u64 cannounce_off,
                                        size_t msglen, bool *redundant)
{
    /* ... read the two node IDs. */
    n[0] = gossmap_find_node(map, &node_id[0]);
    if (n[0])
        nidx[0] = gossmap_node_idx(map, n[0]);
    else
        nidx[0] = new_node(map);

    n[1] = gossmap_find_node(map, &node_id[1]);
    if (n[1])
        nidx[1] = gossmap_node_idx(map, n[1]);
    else
        nidx[1] = new_node(map);

    /* ... create the channel between nidx[0] and nidx[1]. */
    if (!n[0])
        nodeidx_htable_add(map->nodes,
                           node2ptrint(map->node_arr + nidx[0]));
    if (!n[1])
        nodeidx_htable_add(map->nodes,
                           node2ptrint(map->node_arr + nidx[1]));
    return chan;
}
```

For each endpoint, the function looks up its node ID in the node table and creates a new node if it isn't found.
New nodes are added to the table only at the end, after the channel is created.
So when both endpoints are the same unknown node, the second lookup can't see the node the first one just created, and creates it again.
Adding the second copy then puts a node ID into the table that is already there, tripping the table's [no-duplicates assertion](https://github.com/ElementsProject/lightning/blob/6f9356fb9369d34811640f292c75cc3b7e43e9d1/common/gossmap.c#L52-L53).
The failed assertion aborts `gossipd`, and because `lightningd` [treats that as fatal](https://github.com/ElementsProject/lightning/blob/6f9356fb9369d34811640f292c75cc3b7e43e9d1/lightningd/subd.c#L663-L669), the whole node goes down.

Releases before v25.02 also accept the self-referential announcement but don't crash on it, because their node table allowed duplicates and the second copy was simply added.
The table was [switched](https://github.com/ElementsProject/lightning/commit/b6c1ffa359936166162a59a7d16c4f5c7534454b) to reject duplicates in v25.02.

### The crash survives restarts

The announcement is already in the `gossip_store` when `gossipd` aborts.
On the next start, `gossipd` [reads the store back](https://github.com/ElementsProject/lightning/blob/6f9356fb9369d34811640f292c75cc3b7e43e9d1/common/gossmap.c#L939), loads the same announcement, and aborts again.
The node can't start until the operator removes the record from the `gossip_store` by hand, and upgrading doesn't remove a record that is already stored.

## The attack

An attacker completes the BOLT 8 handshake and `init` exchange with the victim as an ordinary peer.
It then does the following:

1. Fund a 2-of-2 P2WSH output whose two keys the attacker controls, so it looks like any Lightning funding output.
2. Craft a `channel_announcement` for the output's `short_channel_id` with a fresh node ID, not yet in the victim's graph, as both endpoints, then sign it with that node's key and the two funding keys.
3. Once the output is 6 blocks deep, send the announcement to the victim.

All four signatures are valid and the output is unspent, so CLN accepts the announcement, writes it to the `gossip_store`, and aborts while loading it.
The funds stay in an output the attacker controls, so the attack costs only on-chain fees.

The attacker also doesn't need to connect to every victim. Nodes that relay such announcements instead of rejecting them, such as CLN releases before v25.02, can carry them to vulnerable nodes the attacker can't connect to directly.

### Observed impact

The node aborts on every restart until the operator removes the record.
While it is down, it can't watch the chain, so it can miss a peer broadcasting a revoked commitment or an HTLC it must claim before it expires, putting funds at risk.
After the operator removes the record, the attacker can resend the same announcement at no further cost for as long as the output stays unspent.

## The fix

[PR #9082](https://github.com/ElementsProject/lightning/pull/9082), merged 2026-04-24 and released in v26.04.1, adds a check, right after parsing, that rejects any `channel_announcement` whose `node_id_1` isn't lexicographically less than `node_id_2`:

```c
const char *gossmap_manage_channel_announcement(
    const tal_t *ctx, struct gossmap_manage *gm,
    const u8 *announce TAKES, const struct node_id *source_peer TAKES,
    const struct amount_sat *known_amount)
{
    /* ... parse the announcement. */
    if (!(node_id_cmp(&node_id_1, &node_id_2) < 0)) {
        return tal_fmt(ctx, "node_id_1 must be the lesser node id! 1=%s, 2=%s",
                       fmt_node_id(tmpctx, &node_id_1),
                       fmt_node_id(tmpctx, &node_id_2));
    }
    /* ... verify the signatures and queue the funding output lookup. */
    return NULL;
}
```

Requiring `node_id_1` to be strictly less rejects equal node IDs as well as misordered ones, so the announcement is never stored and never reaches the routing graph.
[BOLT 7 PR #1333](https://github.com/lightning/bolts/pull/1333), prompted by this report, adds the same requirement for the receiver to the spec.

## Discovery

This bug was found with a gossip state-machine fuzz target for Core Lightning, later proposed in [PR #9115](https://github.com/ElementsProject/lightning/pull/9115).
The target feeds `gossipd` both valid and malformed messages and simulates the chain lookup, so announcements that pass validation are stored and loaded into the routing graph as on a real node.
A self-referential `channel_announcement` generated by the fuzzer tripped the no-duplicates assertion, and reproducing it on a regtest node showed that the crash repeats on every restart.

### Timeline

- **2026-04-23:** Vulnerability reported privately to Core Lightning.
- **2026-04-24:** Fix merged as [PR #9082](https://github.com/ElementsProject/lightning/pull/9082).
- **2026-04-25:** Rusty confirmed the bug was reproduced and fixed in the next release.
- **2026-04-25:** [Core Lightning v26.04.1](https://github.com/ElementsProject/lightning/releases/tag/v26.04.1) released with the fix.
- **2026-05-04:** [BOLT 7 PR #1333](https://github.com/lightning/bolts/pull/1333) merged, adding the receiver requirement to the spec.
- **2026-10-24:** Public disclosure.

## Takeaways

- **Upgrade to the latest Core Lightning release, currently v26.06.9.**
  Vulnerabilities are being found faster than ever, and older releases quickly fall behind on fixes.
- **Validate untrusted input before persisting it.**
  A message that crashes the node after it is stored is reloaded, and crashes it again, on every restart.
- **Check sender requirements on the receiver too.**
  A malicious peer can ignore them, so a receiver that relies on them must check them.

-------------------------

