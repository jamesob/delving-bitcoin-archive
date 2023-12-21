# Denial-of-service bugs in LND's channel update gossip handling

dergoegge | 2023-12-21 16:39:35 UTC | #1

This is a write-up of two denial-of-service bugs I found in [LND](https://github.com/lightningnetwork/lnd) at the end of 2021. Users do not need to take any action at this time, as the initial fixed versions are themselves superseded due to other [exploited](https://github.com/lightningnetwork/lnd/security/advisories/GHSA-hc82-w9v8-83pr) and [disclosed](https://morehouse.github.io/lightning/fake-channel-dos/) vulnerabilities.

# (1) Premature `channel_update` denial-of-service

LND prior to [v0.14.0](https://github.com/lightningnetwork/lnd/releases/tag/v0.14.0-beta) was vulnerable to denial-of-service by running out of memory from unconditionally storing premature channel updates.

## Details

When a channel is first established between two parties, three gossip messages will be broadcast to the network: a `channel_announcement` and two `channel_update`s (one for each channel direction). During the propagation of these messages, a race may occur in which the channel updates are received prior to the channel announcement. The receiving node can not validate these premature channel updates, as the channel announcement contains the public keys that correspond to the signatures in the updates.

To address this race, LND used a buffer for temporarily storing premature channel updates until the channel announcement is received. Unfortunately the size of this buffer was unbounded and an attacker could fill it with entirely invalid updates, leading the victim to run out of memory. As with any crash bug in lightning node software, this comes with a **risk of loosing funds**.

The issue was fixed in [PR #5902](https://github.com/lightningnetwork/lnd/pull/5902) by replacing the unbounded buffer with a limited cache that can hold up to 100 premature updates.

Fun fact: The issue was [fixed](https://github.com/lightningnetwork/lnd/pull/4895) earlier in Jan. 2021 by removing the buffer entirely but it is unclear whether or not that was intentional, as the fix was later [rolled back](https://github.com/lightningnetwork/lnd/pull/5003) due to flaky tests.

# (2) `channel_update` signature validation after rate limiting

LND prior to [v0.15.0](https://github.com/lightningnetwork/lnd/releases/tag/v0.15.0-beta) was vulnerable to channel update gossip censorship.

## Details

In the lightning protocol, there is no inherent cost to creating and broadcasting `channel_update` messages besides having to sign them. To address this, all lightning implementations rate limit how many channel updates they will relay within a given time frame (to prevent gossip spam on the network).

LND used to check signatures of channel updates after applying its relay rate limits. As consequence, an attacker could censor channel updates for any channel from propagating by creating invalid channel updates and spamming them to LND nodes (causing any new valid updates from the victim to be ignored).

The issue was fixed in [PR #6278](https://github.com/lightningnetwork/lnd/pull/6278) by ensuring that signature validation of channel updates happens prior to applying the rate limits.

# Timeline

Disclosure timeline for both bugs:

- 19-10-2021: Initial disclosure to Lightning Labs
- 05-11-2021: (1) fixed: [https://github.com/lightningnetwork/lnd/pull/5902](https://github.com/lightningnetwork/lnd/pull/5902)
- 17-11-2021: v0.14.0 released
- 16-03-2022: (2) fixed: [https://github.com/lightningnetwork/lnd/pull/6278](https://github.com/lightningnetwork/lnd/pull/6278)
- 24-06-2022: v0.15.0 released
- 21-12-2023: Public disclosure

---

Support security focused Bitcoin research and development by [donating to Brink](https://brink.dev/donate).

-------------------------

