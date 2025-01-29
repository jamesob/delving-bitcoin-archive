# Disclosure: LDK Duplicate HTLC Force Close Griefing

morehouse | 2025-01-29 18:02:41 UTC | #1

*The following disclosure is copied verbatim from a [blog post](https://morehouse.github.io/lightning/ldk-duplicate-htlc-force-close-griefing/) on [morehouse.github.io](http://morehouse.github.io), reproduced here to facilitate discussion.*


LDK 0.1 and below are vulnerable to a griefing attack that causes all of the victim's channels to be force closed.
Update to [LDK 0.1.1](https://github.com/lightningdevkit/rust-lightning/releases/tag/v0.1.1) to protect your channels.

# Background

Whenever a new payment is routed through a lightning channel, or whenever an existing payment is settled on the channel, the parties in that channel need to update their commitment transactions to match the new set of active HTLCs.
During the course of these regular commitment updates, there is always a brief moment where one of the parties holds two valid commitment transactions.
Normally that party immediately revokes the older commitment transaction after it receives a signature for the new one, bringing their number of valid commitment transactions back down to one.
But for that brief moment, the other party in the channel must be able to handle the case where *either* of the valid commitments confirms on chain.

For this reason, LDK contains logic to detect when there's a difference between the counterparty's confirmed commitment transaction and the set of currently outstanding HTLCs.
Any HTLCs missing from the confirmed commitment transaction are considered unrecoverable and are immediately failed backward on the upstream channel, while all other HTLCs  are left active until the resolution of the downstream HTLC on chain.

Because the same payment hash and amount can be used for multiple HTLCs (e.g., [multi-part payments](https://github.com/lightning/bolts/blob/e92a7fae44bef8271d07377e95cee82e00acff62/04-onion-routing.md#basic-multi-part-payments)), some extra data is stored to match HTLCs on commitment transactions against the set of outstanding HTLCs.
LDK calls this extra data the "HTLC source" data, and LDK maintains this data for both of the counterparty's valid commitment transactions.

# The Duplicate HTLC Failback Bug

Once a counterparty commitment transaction has been revoked, however, LDK forgets the HTLC source data for that commitment transaction to save memory.
As a result, if a revoked commitment transaction later confirms, LDK must attempt to match commitment transaction HTLCs up to outstanding HTLCs using only payment hashes and amounts.
LDK's [logic](https://github.com/lightningdevkit/rust-lightning/blob/020be440b6d2dfea41820a137c7b26f43b289290/lightning/src/chain/channelmonitor.rs#L2624-L2684) to do this matching works as follows:

```python
for htlc, htlc_source in outstanding_htlcs:
  if !confirmed_commitment_tx.is_revoked() &&
      confirmed_commitment_tx.contains_source(htlc_source):
    continue
  if confirmed_commitment_tx.is_revoked() &&
      confirmed_commitment_tx.contains_htlc(htlc.payment_hash, htlc.amount):
    continue

  failback_upstream_htlc(htlc_source)
```

Note that this logic short-circuits whenever an outstanding HTLC matches the payment hash and amount of an HTLC on the revoked commitment transaction.
Thus if there are multiple outstanding HTLCs with the same payment hash and amount, a single HTLC on the revoked commitment transaction can prevent all of the duplicate outstanding HTLCs from being failed back immediately.

Those duplicate HTLCs remain outstanding until corresponding downstream HTLCs are resolved on chain.
Except, in this case there's only one downstream HTLC to resolve on chain, and its resolution only [triggers](https://github.com/lightningdevkit/rust-lightning/blob/020be440b6d2dfea41820a137c7b26f43b289290/lightning/src/chain/channelmonitor.rs#L4460-L4468) *one* of the duplicate HTLCs to be failed upstream.
**All the other duplicate HTLCs are left outstanding indefinitely**.

# Force Close Griefing

Consider the following topology, where `B` is the victim and the `A_[1..N]` nodes are all the nodes that `B` has channels with.
`M_1` and `M_2` are controlled by the attacker.

```
     -- A_1 --
    /         \
M_1 --  ...  -- B -- M_2
    \         /
     -- A_N --
```

The attacker routes `N` HTLCs from `M_1` to `M_2` using the same payment hash and amount for each, with each payment going through a different `A` node.
`M_2` then confirms a revoked commitment that contains only one of the `N` HTLCs.
Due to the duplicate HTLC failback bug, only one of the routed HTLCs gets failed backwards, while the remaining `N-1` HTLCs get stuck.

Finally, after upstream HTLCs expire, all the `A` nodes with stuck HTLCs force close their channels with `B` to reclaim the stuck HTLCs.

## Attack Cost

The attacker must broadcast a revoked commitment transaction, thereby forfeiting their channel balance.
But the size of the channel can be minimal, and the attacker can spend their balance down to the 1% reserve before executing the attack.
As a result, the cost of the attack can be negligible compared to the damage caused.

# The Fix

Starting in v0.1.1, LDK [preemptively fails back](https://github.com/lightningdevkit/rust-lightning/pull/3556) HTLCs when their deadlines approach if the downstream channel has been force closed or is in the process of force closing.
While the main purpose of this behavior is to prevent cascading force closures when mempool fee rates spike, it also has a nice side effect of ensuring that duplicate HTLCs always get failed back eventually after a revoked commitment transaction confirms.
As a result, the duplicate HTLCs are never stuck long enough that the upstream nodes need to force close to reclaim them.

# Discovery

This vulnerability was discovered during an audit of LDK's chain module.

## Timeline

- **2024-12-07:** Vulnerability reported to the LDK security mailing list.
- **2025-01-27:** Fix [merged](https://github.com/lightningdevkit/rust-lightning/pull/3556).
- **2025-01-28:** LDK 0.1.1 released containing the fix, with public disclosure in [release notes](https://github.com/lightningdevkit/rust-lightning/releases/tag/v0.1.1).
- **2025-01-29:** Detailed description of vulnerability published.

# Prevention

Prior to the [introduction](https://github.com/lightningdevkit/rust-lightning/commit/70ae45fea030ed1d2064918c7b023aa142387bc8) of the duplicate HTLC failback bug in 2022, LDK would immediately fail back *all* outstanding HTLCs once a revoked commitment reached 6 confirmations.
This was the safe and conservative thing to do -- HTLC source information was missing, so proper matching of HTLCs could not be done.
And since all outputs on the revoked commitment and HTLC transactions could be claimed via revocation key, there was no concern about losing funds if the downstream counterparty confirmed an HTLC claim before LDK could.

## Better Documentation

Considering that LDK previously had a [test](https://github.com/lightningdevkit/rust-lightning/commit/70ae45fea030ed1d2064918c7b023aa142387bc8#diff-b30410f22a759d5e664e05938af7ef2edd244c8a7872e7ada376055ff130088bL7296-L7314) explicitly checking for the original (conservative) failback behavior, it does appear that the original behavior was understood and intentional.
Unfortunately the original author did not document the *reason* for the original behavior anywhere in the code or test.

A single comment in the code would likely have been enough to prevent later contributors from introducing the buggy behavior:

```c
// We fail back *all* outstanding HTLCs when a revoked commitment
// confirms because we don't have HTLC source information for revoked
// commitments, and attempting to match up HTLCs based on payment hashes
// and amounts is inherently unreliable.
//
// Failing back all HTLCs after a 6 block delay is safe in this case
// since we can use the revocation key to reliably claim all funds in the
// downstream channel and therefore won't lose funds overall.
```

# Takeaways

- Code documentation matters for preventing bugs.
- Update to LDK 0.1.1 for the vulnerability fix.

-------------------------

