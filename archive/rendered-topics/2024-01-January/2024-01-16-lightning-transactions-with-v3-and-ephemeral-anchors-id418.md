# Lightning transactions with v3 and ephemeral anchors

t-bast | 2024-01-16 17:28:44 UTC | #1

# Lightning transactions with v3 and ephemeral anchors

In this post, I'll detail the changes I believe we should make to lightning transactions to leverage v3 transactions and ephemeral anchors.
See https://delvingbitcoin.org/t/v3-transaction-policy-for-anti-pinning for details on those policy changes at the bitcoin layer.

## Commitment transaction

The commitment transaction represents the current state of a lightning channel's funds, by splitting the funding amount into the various outputs of the commitment transaction.
Whenever a change happens in the channel (HTLCs are added or removed or the feerate changes), a new commitment transaction is signed.
The previous commitment transaction is revoked by revealing a revocation key that allows the other peer to spend every output of that transaction.

It has the following format:

```ascii
+------------+
| funding tx |
+------------+
      |
      |        +-------------------+
      +------->|    commit tx B    |
               +-------------------+
                  |  |  |  |  |  |  
                  |  |  |  |  |  | A's main output
                  |  |  |  |  |  +-----------------> to A after a 1-block relative delay
                  |  |  |  |  |
                  |  |  |  |  |                 +---> to B after a relative delay
                  |  |  |  |  | B's main output |
                  |  |  |  |  +-----------------+
                  |  |  |  |                    |
                  |  |  |  |                    +---> to A immediately with revocation key
                  |  |  |  |
                  |  |  |  |                   +---> to A with its channel funding key
                  |  |  |  | A's anchor output |
                  |  |  |  +-------------------+
                  |  |  |                      |
                  |  |  |                      +---> to anyone after a 16-blocks relative delay
                  |  |  |
                  |  |  |                   +---> to B with its channel funding key
                  |  |  | B's anchor output |
                  |  |  +-------------------+
                  |  |                      |
                  |  |                      +---> to anyone after a 16-blocks relative delay
                  |  |
                  |  |                                              +---> to B after relative delay
                  |  |                        +-----------------+   |
                  |  |                   +--->| HTLC-timeout tx |---+
                  |  | HTLC offered by B |    +-----------------+   |
                  |  +-------------------+      (after timeout)     +---> to A immediately with revocation key
                  |                      |
                  |                      +---> to A with payment preimage after a 1-block relative delay
                  |                      |
                  |                      +---> to A immediately with revocation key
                  |
                  |                                                     +---> to B after relative delay
                  |                            +-----------------+      |
                  |                    +------>| HTLC-success tx |------+
                  | HTLC received by B |       +-----------------+      |
                  +--------------------+     (with payment preimage)    +---> to A immediately with revocation key
                                       |
                                       +---> to A after timeout (absolute delay) and a 1-block relative delay
                                       |
                                       +---> to A immediately with revocation key
```

Some interesting details to highlight are:

- the anchor outputs are the only outputs that don't have a CSV (except when spending a revoked commitment)
- the anchor outputs have a fixed amount (330 sat)

The fees of the commitment transaction are deduced from the channel initiator's main output.
Both peers can agree to update the feerate and thus deduce more (or less) fees from the channel initiator's output.
Note that this creates a strong coupling between the current on-chain fees and the funds that are available to spend off-chain, which is undesirable.

There is another mechanism that increases the fees paid by the commitment transaction: pending trimmed HTLCs.
When HTLCs are added that aren't economical to spend on-chain, we don't create an HTLC-offered or HTLC-received output in the commitment transaction.
The amount of that HTLC is instead simply deduced from the sender's main output, and temporarily allocated to mining fees.
When that HTLC is resolved, its amount is either added back to the sender's main output (if failed) or added to the recipient's main output (if fulfilled).

With v3 and ephemeral anchors, we could make the following changes:

- the commit tx uses v3
- we replace the two anchor outputs by a single ephemeral anchor output
- the amount of the ephemeral anchor output will be the sum of all pending trimmed HTLCs (thus potentially 0 sat)
- we remove the 1-block relative delay on the main outputs and HTLC outputs
- we remove the `update_fee` message (since commitment transactions don't pay fees anymore)

This effectively removes the coupling between the current on-chain fees and the funds available to spend off-chain.
The maximum amounts that can be sent and received on a given channel are now fully deterministic.
We don't need hacks such as the [fee spike buffer](https://github.com/lightning/bolts/pull/740) anymore and similar edge cases disappear.
Channel participants only need to care about on-chain fees whenever they publish a commitment transaction.

Notes and caveats:

- the commitment transaction can only be published as a package, with a child transaction spending the ephemeral anchor
- only packages of 1-parent 1-child will be allowed initially, which prevents batching
- the ephemeral anchor transaction may need additional wallet inputs when broadcasting our local commitment, if the ephemeral anchor value is too low
  - those wallet inputs must be confirmed
- but it doesn't need any wallet input when we detect a revoked commitment or the remote commitment: we can instead use one of the channel's outputs to pay the fees

An interesting thing to note is that if the ephemeral anchor output has an amount that is greater than the current on-chain fees, miners will always have an incentive to claim it all for themselves.
For example Alice may publish a commitment transaction with a 50 000 sats ephemeral anchor, along with a child transaction spending the ephemeral anchor and sending 20 000 sats back to herself.
This package pays 30 000 sats of mining fees. But since the ephemeral anchor can be spent by anyone, miners should replace the child transaction by one that sends the funds to themselves, thus earning 50 000 sats of fees.
This isn't an issue, since this is already the case with lightning today: whenever a commitment transaction with pending trimmed HTLCs is broadcast, those trimmed HTLCs directly go to miners.
Node operators decide what their [maximum dust exposure](https://github.com/lightning/bolts/pull/919) is, and it will constrain the amount of the ephemeral anchor output.

The commitment transaction will then have the following format:

```ascii
+------------+
| funding tx |
+------------+
      |
      |        +-----------------+
      +------->|   commit tx B   |
               +-----------------+
                  |  |  |  |  |  
                  |  |  |  |  | A's main output
                  |  |  |  |  +-----------------> to A
                  |  |  |  |
                  |  |  |  |                 +---> to B after a relative delay
                  |  |  |  | B's main output |
                  |  |  |  +-----------------+
                  |  |  |                    |
                  |  |  |                    +---> to A with revocation key
                  |  |  |
                  |  |  |
                  |  |  | ephemeral anchor output
                  |  |  +-------------------------> to anyone
                  |  |  
                  |  |                                              +---> to B after relative delay
                  |  |                        +-----------------+   |
                  |  |                   +--->| HTLC-timeout tx |---+
                  |  | HTLC offered by B |    +-----------------+   |
                  |  +-------------------+      (after timeout)     +---> to A with revocation key
                  |                      |
                  |                      +---> to A with payment preimage
                  |                      |
                  |                      +---> to A with revocation key
                  |
                  |                                                     +---> to B after relative delay
                  |                            +-----------------+      |
                  |                    +------>| HTLC-success tx |------+
                  | HTLC received by B |       +-----------------+      |
                  +--------------------+     (with payment preimage)    +---> to A with revocation key
                                       |
                                       +---> to A after timeout (absolute delay)
                                       |
                                       +---> to A with revocation key
```

## HTLC transactions

When using `option_anchors_zero_fee_htlc_tx`, HTLC transactions don't pay any mining fees: they are signed with `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY` to allow the broadcaster to add inputs.
We don't need to change this behavior, it will keep working exactly the same after changing the format of the commitment transaction.
I don't think we should change anything for HTLC transactions, but maybe I'm missing an interesting opportunity here?

## Upgrade path

There are various proposals to upgrade channels without closing them:

- https://github.com/lightning/bolts/pull/868
- https://github.com/lightning/bolts/pull/1117

They have nice interactions with [splicing](https://github.com/lightning/bolts/pull/863) (which is a feature that most implementations are already working on).
Depending on the timeline and perceived urgency of upgrading, we may wait for one of these proposals to be finalized into the spec, or create a simpler upgrade path for that specific scenario.

-------------------------

instagibbs | 2024-01-16 18:06:54 UTC | #2

[quote="t-bast, post:1, topic:418"]
we remove the 1-block relative delay on the main outputs and HTLC outputs
[/quote]

Note that this is easily done by having the feature not be `option_anchors`, since that's the prior format for all the output scripts.

The other discussion that came up is that of deployment strategy. There's a bit of a timeline problem:
1) We need to roll out updates to mempool/relay to make the transition to v3 style transactions safe, so LN implementations can adopt it
2) We need LN implementations to update before we can deploy cluster mempool upgrade, since CPFP carveout as it stands is inapplicable to cluster mempool. 

A suggestion on the call was essentially "what if Bitcoin Core implicitly opts commitment transactions that rely on CPFP carveout into a new regime that doesn't rely on the carveout per se"?

This could lead to a revised project roadmap:

1) V3, 2-cluster package rbf, 1p1c relay + mild orphanage churn protection
2) Imbue LN commit txs with two anchors with “implicit signaling” of V3  
  a)  Anything with *two 330-sat outputs*?  
      i) Or get more specific, e.g., one input only, nversion==2, a couple of bytes here or there...  
  b) Allow 1p2c “cpfp carveout”  
    i) “One more” child, same size limitations as normal v3 child tx limit  
    ii) This, along with 2-cluster package RBF, means if you stop spending the remote anchor, you can now efficiently do package rbf against remote tx, and the counter-party cannot pin local commit tx. Child will have to be “small” following V3 rules. Exact implementation TBD  
3) Further rollouts, with no inherent inter-dependencies
  a) Cluster mempool
  b) Further make orphan handling robust
  c) Ephemeral anchors
  d) LN spec update to do 0-fee commit txns
4) ... Remove the 1p2c CPFP carveout

It doesn't interfere with the current work, and removes critical paths from deployment, so I think it's worth considering.

With the revised timeline, we may be able to swap out CPFP carveout without doing any harm, allow implementations to improve their security via limited package rbf, and allow spec updates to happen on their own time for further improvements.

---

# V3 Child tx size

We would also like feedback on what the maximal V3 child size should be. It's an inherent tradeoff between practical CPFP tx sizes and potential pin vectors, so it'd be nice to know what expectations are around that from any wallet project.

-------------------------

