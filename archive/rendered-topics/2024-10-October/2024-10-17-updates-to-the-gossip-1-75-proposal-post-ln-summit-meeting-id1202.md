# Updates to the Gossip 1.75 proposal post LN summit meeting

ellemouton | 2024-10-17 15:00:21 UTC | #1

Hey everyone! After the various discussions that were had at the summit about
the gossip updates, it is clear that the current proposal needs quite a few
updates. So before I go ahead and start pushing updates to the proposal PR,
I thought I'd first make sure we are all on the same page and to make sure I didn't miss anything.

This post includes:
1. My main action item takeaways from the summit in regard to updating the
   existing proposal.
2. A proposal for rolling out the full update via 3 feature bits.
3. A thought on a potentially convenient side effect that could allow peers to
   upgrade a channel from unannounced to announced.

## A) Pure TLV for all new messages along with an unsigned range

The first update will be to change the current structure of all the new
messages (which has a fixed field for the signature followed by a TLV stream
that the signature completely covers) to instead follow the pattern used by
some of the Bolt 12 messages: make everything a TLV field (including the
signature) but have a designated TLV type range that the signature explicitly
will not cover. afaict, it makes sense just to use the same range that the
Bolt 12 messages use (where all signed fields are in the inclusive ranges:
0 to 159 and 1000000000 to 2999999999) (perhaps it also makes sense to define
this Pure TLV pattern separately from any particular bolt).

Why is this nice/useful?

1) With this messages structure update, it will allow peers to request
   additional information from their peers during gossip sync. For example, a
   light-client could request that its peers include SPV proofs in any
   `channel_announcement_2` messages that they forward. This proof can then be
   added to the message under a TLV type that is not signed by the announcement
   signature.
2) Since the signature no longer covers the complete serialisation of the
   message, a node that does not understand an optional field in the unsigned
   range can throw it away before persisting it if they so choose. Same goes for
   fields in the un-signed range that they do understand but don't need (such as
   an SPV proof).

## B) Using the new messages for P2WSH channel announcements and updates

Probably the biggest update to the initial proposal is to make the new messages
flexible enough such that they can be re-used for announcing P2WSH channels
(both announcing new P2WSH channels and re-announcing existing channels).

In terms of creating a new `channel_announcement_2` for a legacy channel, if we
want to use a Schnorr signature for these, then the channel peers will first
need to swap nonces for both their node and BTC keys via `channel_ready` (as is
done in the existing proposal for when creating `channel_announcement_2` for
simple taproot channels) and then will go on to use `announcement_sigs_2` as
normal. In terms of verifying these messages, any verifying node will first need
to obtain the pkscript of the funding transaction. From this, they can identify
the type of channel and will then know how the rest of the verification needs to
be done.

The biggest hurdle I see here (for LND at least) is that we really only expect
to do this announcement signing flow once in the channel lifecycle. So
implementations would need to be ok with handling this signing & announcing at
any time in the life cycle (I believe CLN already handles this?). Once this
refactor is complete then as long as both channel peers know how (see feature
bit 2 below) then either side can decide to initiate the process of signing a
`channel_announcement_2` for their P2WSH channel at any time (whether it be a
new channel or a long-standing one).

A few rapid-fire points on this:
- Initially, nodes will probably do both signing protocols for quite a while,
  and then they can decide to start switching off the legacy protocol when the
  time feels right (ie when most of the network seems to have upgraded).
- If nodes can handle this signing of a channel announcement at any time, then
  this could pave the way for channel peers to very easily decide to upgrade
  a long-standing unannounced channel to an announced one. This could be as
  simple as: If a node wants to upgrade the channel, they can send the
  `channel_ready` with the nonces as the signal to do so. If the peer responds
  with their own nonces, then they have agreed to the upgrade.
- For nodes receiving both old and new messages, the rules should be quite
  simple:
    - For validating, writing and broadcasting the gossip they receive, they
      should treat the two protocols as completely disjoint. For example, if a
      node has `chan_ann_1` for channel X, they should accept `chan_update_1` for
      the channel but should reject `chan_update_2` until they receive
      `chan_ann_2` for the channel. The rules for when/if to send out `node_ann_2`
      easily follow on from this.
    - When using these announcements and updates for their own pathfinding and
      graph view, nodes should favour the `v2` messages as the source of truth
      when it comes to the channel announcement. For channel updates, they should
      also try to favour the `v2` message where possible but should cater for
      switching back to the `v1` message if the `v2` one becomes very outdated for
      example.

## C) Outpoint and optional inclusion of SPV proofs in `channel_announcement_2`

There are three types of nodes we want to consider:

1) A node with a full chain backend without `txindex` enabled.
2) A node with a full chain backend with `txindex` enabled.
3) A light-client node (has easy access to block-headers).

In today's `channel_announcement(1)`, only the SCID of the funding transaction
is communicated. This makes verification of the announcement difficult because
all node types need to fetch the entire block that the transaction is in and
nodes that do have `txindex` cannot make use of it for faster transaction
retrieval. Just as a reminder, nodes want to fetch the transaction that the SCID
points to so that they can extract the pk script from it and then prove that the
channel_announcement indeed proves ownership over the output. As a bonus nodes
generally also check that the output is unspent.

- Including the outpoint of the funding transaction in the channel_announcement
  should solve the issue for nodes that do have `txindex` enabled to make use
  of their index to fetch the funding transaction.
- Having the option of including an SPV proof in the new channel announcement
  (in the un-signed range) covers the light-client case if we include the raw
  transaction in the proof as well. The proof will thus contain the raw
  transaction along with the remainder of the merkle proof hashes (the first can
  be derived from the raw tx). The light-client can then: 1) extract the PK
  script from the raw transaction (it uses the SCID to know which output to
  use) 2) verify that the transaction is in the block specified by the SCID
  given the merkle proof. Nodes without `txindex` may also choose to use this
  verification process.

Some questions and thoughts on the above:
1. Adding the outpoint as a signed TLV technically adds 2 sources of truth to
   the channel announcement. But I assume this is ok since the SCID still is
   useful in its own right since it provides nodes with an easy way to link an
   announcement to an update without having to derive the SCID. The other option
   is to just leave the SCID and have the outpoint as an unsigned optional
   field that nodes can specifically ask for via gossip sync feature bits.
2. We could require that the channel peers create a hash of the SPV proof and
   include that hash in the signed range of the message. That way nodes can do
   a quick check to ensure that the proof they receive matches the hash before
   making use of the proof. But I'm not sure that this is actually necessary
   since the proof will in any case contain the raw tx with the pk script which
   they can use to check the signature of the message against. So I think the
   signature already commits to part of the proof in this way. Another
   argument against including this hash is that both channel peers will need to
   derive the proof when they are constructing the announcement, which may not 
   be feasible for some peers

## D) Gossip 1.75 upgrade path via 3 feature bits

Since the proposal is quite large and is a network wide upgrade, I propose that
for the sake of piece-meal implementation, we use 3 new feature bits for the
full upgrade. The feature bits will have the following meanings:

### Feature bit 1:

This feature bit is the main "gossip_v2" feature bit that signals:
- The node can understand the new gossip messages (meaning it also understands
  what a taproot channel is and knows how to verify a taproot channel as well as
  a P2WSH channel advertised via the new messages).
- When used in combination with the "simple_taproot_channels" feature bit, this
  also means that the node can create an advertised taproot channel (ie, it
  knows how to construct a `channel_announcement_2` message with its channel
  peer).

### Feature bit 2:

- Depends on bit 1
- Signals to its channel peers that a node is able to and willing to re-announce
  P2WSH channels using the new protocol.

### Feature bit 3:

- Depends on bit 1
- Signals that a node is able and willing to provide SPV proofs of a funding
  transaction along with `channel_announcement_2` messages if asked.

## E) Upgrade channels from un-announced to announced (Feature bit 4?)

As mentioned in section B of this post, after the implementation of feature bit
2 as defined above, a node should theoretically be set-up to be able to upgrade
existing channels from un-announced to announced. So potentially we could have
a fourth feature bit here that signals the ability to do this upgrade.


Cool - that's it I think!
Looking forward to any thoughts.

Elle

-------------------------

