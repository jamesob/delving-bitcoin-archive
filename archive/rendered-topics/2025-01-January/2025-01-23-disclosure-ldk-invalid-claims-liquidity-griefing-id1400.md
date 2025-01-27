# Disclosure: LDK Invalid Claims Liquidity Griefing

morehouse | 2025-01-23 16:08:59 UTC | #1

*The following disclosure is copied verbatim from a [blog post](https://morehouse.github.io/lightning/ldk-invalid-claims-liquidity-griefing/) on morehouse.github.io, reproduced here to facilitate discussion.*

LDK 0.0.125 and below are vulnerable to a liquidity griefing attack against anchor channels.
The attack locks up funds such that they can only be recovered by manually constructing and broadcasting a valid claim transaction.
Affected users can unlock their funds by upgrading to [LDK 0.1](https://github.com/lightningdevkit/rust-lightning/releases/tag/v0.1) and replaying the sequence of commitment and HTLC transactions that led to the lock up.

# Background

When a channel is force closed, LDK creates and broadcasts transactions to claim any HTLCs it can from the commitment transaction that confirmed on chain.
To save on fees, some HTLC claims are aggregated and broadcast together in the same transaction.

If the channel counterparty is able to get a competing HTLC claim confirmed first, it can cause one of LDK's aggregated transactions to become invalid, since the corresponding HTLC input has already been spent by the counterparty's claim.
LDK contains logic to detect this scenario and remove the already-claimed input from its aggregated claim transaction.
When everything works correctly, the aggregated transaction becomes valid again and LDK is able to claim the remaining HTLCs.

# The Invalid Claims Bug

Prior to LDK 0.1, the logic to detect conflicting claims works like this:

```python
for confirmed_transaction in confirmed_block:
  for input in confirmed_transaction:
    if claimable_outpoints.contains(input.prevout):
      agg_tx = get_aggregated_transaction_from_outpoint(input.prevout)
      agg_tx.remove_matching_inputs(confirmed_transaction)
      break  # This is the bug.
```

Note that this logic stops processing a confirmed transaction after finding the first aggregated transaction that conflicts with it.
If the confirmed transaction conflicts with *multiple* aggregated transactions, conflicting inputs are only removed from the *first* matching aggregated transaction, and any other conflicting aggregated transactions are left invalid.

Any HTLCs claimed by invalid aggregated transactions get locked up and can only be recovered by manually constructing and broadcasting valid claim transactions.

# Liquidity Griefing

Prior to LDK 0.1, there are only two types of HTLC claims that are aggregated:

- HTLC preimage claims
- revoked commitment HTLC claims

For HTLC preimage claims, LDK takes care to confirm them before their HTLCs time out, so there's no reliable way for an attacker to confirm a conflicting timeout claim and trigger the invalid claims bug.

For revoked commitment transactions, however, an attacker can immediately spend any incoming HTLC outputs via HTLC-Success transactions.
Although LDK is then able to claim the HTLC-Success outputs via the revocation key, the attacker can exploit the invalid claims bug to lock up any remaining HTLCs on the revoked commitment transaction.

## Setup

The attacker opens an anchor channel with the victim, creating a network topology as follows:
```
A -- B -- M
```
In this case `B` is the victim LDK node and `M` is the node controlled by the attacker.
The attacker must use an anchor channel so that they can spend multiple HTLC claims in the same transaction and trigger the invalid claims bug.

The attacker then routes HTLCs along the path `A->B->M` as follows:

1. 1 small HTLC with CLTV of `X`
2. 1 small HTLC with CLTV of `X+1`
3. 1 large HTLC with CLTV of `X+1`  (this is the one the attacker will lock up)

The attacker knows preimages for all HTLCs but withholds them for now.

To complete the setup, the attacker routes some other HTLC through the channel, causing the commitment transaction with the above HTLCs to be revoked.

## Forcing Multiple Aggregations

Next the attacker waits until block `X-13` and force closes the `B-M` channel using their revoked commitment transaction, being sure to get it confirmed in block `X-12`.
By confirming in this specific block, the attacker can exploit LDK's buggy aggregation logic prior to v0.1 (see below), causing LDK to aggregate HTLC justice claims as follows:

- **Transaction 1:**  HTLC 1
- **Transaction 2:**  HTLCs 2 and 3

### Buggy Aggregation Logic

Prior to v0.1, LDK only aggregates HTLC claims if their timeouts are more than 12 blocks in the future.
Presumably 12 blocks was deemed "too soon" to guarantee that LDK can confirm preimage claims before the HTLCs time out, and once one HTLC times out the counterparty can pin a competing timeout claim in mempools, thereby preventing confirmation of *all* the aggregated preimage claims.
In other words, by claiming HTLCs separately in this scenario, LDK limits the damage the counterparty could do if one of those HTLCs expires before LDK successfully claims it.

Unfortunately, this aggregation strategy makes no sense when LDK is trying to group justice claims that the counterparty can spend immediately via HTLC-Success, since the timeout on those HTLCs does not apply to the counterparty.
Nevertheless, prior to LDK 0.1, the same 12 block aggregation check applies equally to all justice claims, regardless of whether the counterparty can spend them immediately or must wait to spend via HTLC-Timeout.

An attacker can exploit this buggy aggregation logic to make LDK create multiple claim transactions, as described above.

## Locking Up Funds

Finally, the attacker broadcasts and confirms a transaction spending HTLCs 1 and 2 via HTLC-Success.
The attacker's transaction conflicts with both Transaction 1 and Transaction 2, but due to the invalid claims bug, LDK only notices the conflict with Transaction 1.
LDK continues to fee bump and rebroadcast Transaction 2 indefinitely, even though it can never be mined.

As a result, the funds in HTLC 3 remain inaccessible until a valid claim transaction is manually constructed and broadcast.

Note that if the attacker ever tries to claim HTLC 3 via HTLC-Success, LDK is able to immediately recover it via the revocation key.
So while the attacker can lock up HTLC 3, they cannot actually steal it once the upstream HTLC times out.

## Attack Cost

When the attacker's revoked commitment transaction confirms, LDK is able to immediately claim the attacker's channel balance.
LDK is also able to claim HTLCs 1 and 2 via the revocation key on the `B-M` channel, while also claiming them via the preimage on the upstream `A-B` channel.

Thus a smart attacker would minimize costs by spending their channel balance down to the 1% reserve before carrying out the attack and would then set the amounts of HTLCs 1 and 2 to just above the dust threshold.
The attacker would also maximize the pain inflicted on the victim by setting HTLC 3 to the maximum allowed amount.

# Stealing HTLCs in 0.1-beta

Beginning in [v0.1-beta](https://github.com/lightningdevkit/rust-lightning/releases/tag/v0.1.0-beta1), LDK [started](https://github.com/lightningdevkit/rust-lightning/pull/3340) aggregating HTLC timeout claims that have compatible locktimes.
As a result, the beta release is vulnerable to a variant of the liquidity griefing attack that enables the attacker to steal funds.
Thankfully the invalid claims bug was fixed between the 0.1-beta and 0.1 releases, so the final LDK 0.1 release is not vulnerable to this attack.

The fund-stealing variant for LDK 0.1-beta works as follows.

## Setup

The attack setup is identical to the liquidity griefing attack, except that the attacker does not cause its commitment transaction to be revoked.

## Forcing Multiple Aggregations

The attacker then force closes the `B-M` channel.
Due to differing locktimes, LDK creates HTLC timeout claims as follows:

- **Transaction 1:**  HTLC 1  (locktime `X`)
- **Transaction 2:**  HTLCs 2 and 3  (locktime `X+1`)

Once height `X` is reached, LDK broadcasts Transaction 1.
At height `X+1`, LDK broadcasts Transaction 2.

At this point, if Transaction 1 confirmed immediately in block `X+1`, the attack fails since the attacker can no longer spend HTLCs 1 and 2 together in the same transaction.
But if Transaction 1 did not confirm immediately (which is more likely), the attack can continue.

## Stealing Funds

The attacker broadcasts and confirms a transaction spending HTLCs 1 and 2 via HTLC-Success.
This transaction conflicts with both Transaction 1 and Transaction 2, but due to the invalid claims bug, LDK only notices the conflict with Transaction 1.
LDK continues to fee bump and rebroadcast Transaction 2 indefinitely, even though it can never be mined.

Once HTLC 3's upstream timeout expires, node `A` force closes and claims a refund, leaving the coast clear for the attacker to claim the downstream HTLC via preimage.

# The Fix

The invalid claims bug was fixed by a [one-line patch](https://github.com/lightningdevkit/rust-lightning/pull/3538) just prior to the LDK 0.1 release.

# Discovery

This vulnerability was discovered during an audit of LDK's chain module.

## Timeline

- **2024-12-23:** Vulnerability reported to the LDK security mailing list.
- **2025-01-15:** Fix [merged](https://github.com/lightningdevkit/rust-lightning/pull/3538).
- **2025-01-16:** LDK 0.1 released containing the fix, with public disclosure in release notes.
- **2025-01-23:** Detailed description of vulnerability published.

# Prevention

The invalid claims bug is fundamentally a problem of incorrect control flow -- a `break` statement was inserted into a loop where it shouldn't have been.
Why wasn't it caught during initial code review, and why wasn't it noticed for years after that?

The `break` statement was [introduced](https://github.com/lightningdevkit/rust-lightning/commit/feb472dc9ef971b926b19d27e1ad05a79423778f) back in 2019, long before LDK supported anchor channels.
The code was actually correct back then, because before anchor channels there was no way for the counterparty to construct a transaction that conflicted with two of LDK's aggregated transactions.
But even after [LDK 0.0.116](https://github.com/lightningdevkit/rust-lightning/releases/tag/v0.0.116) added support for anchor channels, the bug went unnoticed for over two years, despite multiple changes being made to the surrounding code in that time frame.

It's impossible to say exactly what kept the bug hidden, but I think the complexity and unreadability of the surrounding code was a likely contributor.
Here's the for-loop containing the [buggy code](https://github.com/lightningdevkit/rust-lightning/blob/ad462bd9c8237e505f463c227c9ac98ebd3fbb16/lightning/src/chain/onchaintx.rs#L896-L984):

```rust
let mut bump_candidates = new_hash_map();
if !txn_matched.is_empty() { maybe_log_intro(); }
for tx in txn_matched {
    // Scan all input to verify is one of the outpoint spent is of interest for us
    let mut claimed_outputs_material = Vec::new();
    for inp in &tx.input {
        if let Some((claim_id, _)) = self.claimable_outpoints.get(&inp.previous_output) {
            // If outpoint has claim request pending on it...
            if let Some(request) = self.pending_claim_requests.get_mut(claim_id) {
                //... we need to check if the pending claim was for a subset of the outputs
                // spent by the confirmed transaction. If so, we can drop the pending claim
                // after ANTI_REORG_DELAY blocks, otherwise we need to split it and retry
                // claiming the remaining outputs.
                let mut is_claim_subset_of_tx = true;
                let mut tx_inputs = tx.input.iter().map(|input| &input.previous_output).collect::<Vec<_>>();
                tx_inputs.sort_unstable();
                for request_input in request.outpoints() {
                    if tx_inputs.binary_search(&request_input).is_err() {
                        is_claim_subset_of_tx = false;
                        break;
                    }
                }

                macro_rules! clean_claim_request_after_safety_delay {
                    () => {
                        let entry = OnchainEventEntry {
                            txid: tx.compute_txid(),
                            height: conf_height,
                            block_hash: Some(conf_hash),
                            event: OnchainEvent::Claim { claim_id: *claim_id }
                        };
                        if !self.onchain_events_awaiting_threshold_conf.contains(&entry) {
                            self.onchain_events_awaiting_threshold_conf.push(entry);
                        }
                    }
                }

                // If this is our transaction (or our counterparty spent all the outputs
                // before we could anyway with same inputs order than us), wait for
                // ANTI_REORG_DELAY and clean the RBF tracking map.
                if is_claim_subset_of_tx {
                    clean_claim_request_after_safety_delay!();
                } else { // If false, generate new claim request with update outpoint set
                    let mut at_least_one_drop = false;
                    for input in tx.input.iter() {
                        if let Some(package) = request.split_package(&input.previous_output) {
                            claimed_outputs_material.push(package);
                            at_least_one_drop = true;
                        }
                        // If there are no outpoints left to claim in this request, drop it entirely after ANTI_REORG_DELAY.
                        if request.outpoints().is_empty() {
                            clean_claim_request_after_safety_delay!();
                        }
                    }
                    //TODO: recompute soonest_timelock to avoid wasting a bit on fees
                    if at_least_one_drop {
                        bump_candidates.insert(*claim_id, request.clone());
                        // If we have any pending claim events for the request being updated
                        // that have yet to be consumed, we'll remove them since they will
                        // end up producing an invalid transaction by double spending
                        // input(s) that already have a confirmed spend. If such spend is
                        // reorged out of the chain, then we'll attempt to re-spend the
                        // inputs once we see it.
                        #[cfg(debug_assertions)] {
                            let existing = self.pending_claim_events.iter()
                                .filter(|entry| entry.0 == *claim_id).count();
                            assert!(existing == 0 || existing == 1);
                        }
                        self.pending_claim_events.retain(|entry| entry.0 != *claim_id);
                    }
                }
                break; //No need to iterate further, either tx is our or their
            } else {
                panic!("Inconsistencies between pending_claim_requests map and claimable_outpoints map");
            }
        }
    }
    for package in claimed_outputs_material.drain(..) {
        let entry = OnchainEventEntry {
            txid: tx.compute_txid(),
            height: conf_height,
            block_hash: Some(conf_hash),
            event: OnchainEvent::ContentiousOutpoint { package },
        };
        if !self.onchain_events_awaiting_threshold_conf.contains(&entry) {
            self.onchain_events_awaiting_threshold_conf.push(entry);
        }
    }
}
```

<br/><br/>

Perhaps others have a better mental parser than me, but I find this code quite difficult to read and understand.
The loop is so long, with so much nesting and so many low-level implementation details that by the time I get to the buggy `break` statement, I've completely forgotten what loop it applies to.
And since the comment attached to the break statement gives a believable explanation, it's easy to gloss right over it.

Perhaps the buggy control flow would be easier to spot if the loop was simpler and more compact.
By hand-waving some helper functions into existence and refactoring, the same code could be written as follows:

```rust
maybe_log_intro();

let mut bump_candidates = new_hash_map();
for tx in txn_matched {
    for inp in &tx.input {
        if let Some(claim_request) = self.get_mut_claim_request_from_outpoint(inp.previous_output) {
            let split_requests = claim_request.split_off_matching_inputs(&tx.input);
            debug_assert!(!split_requests.is_empty());

            if claim_request.outpoints().is_empty() {
                // Request has been fully claimed.
                self.mark_request_claimed(claim_request, tx, conf_height, conf_hash);
                break;
            }

            // After removing conflicting inputs, there's still more to claim.  Add the modified
            // request to bump_candidates so it gets fee bumped and rebroadcast.
            self.remove_pending_claim_events(claim_request);
            bump_candidates.insert(claim_request.clone());

            self.mark_requests_contentious(split_requests, tx, conf_height, conf_hash);
            break;
        }
    }
```

The control flow in this version is much more apparent to the reader.
And although there's no guarantee that the buggy `break` statements would have been discovered sooner if the code was written this way, I do think the odds would have been much better.

# Takeaways

- Code readability matters for preventing bugs.
- Update to LDK 0.1 for the vulnerability fix.

-------------------------

