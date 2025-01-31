# Minimal signing flow changes for TRUC channels

morehouse | 2025-01-31 19:19:09 UTC | #1

To truly mitigate pinning against lightning channels, we need to force all commitment transactions and all preimage-claim transactions to use TRUC (see discussions [here](https://github.com/lightning/bolts/issues/1221#issuecomment-2621162542) and [here](https://delvingbitcoin.org/t/lightning-transactions-with-v3-and-ephemeral-anchors/418/12)).  @t-bast has outlined how we can do this for [commitment transactions](https://delvingbitcoin.org/t/lightning-transactions-with-v3-and-ephemeral-anchors/418), but some extra work is needed for preimage claims.

# Local commitment preimage claims

We already use presigned HTLC-Success transactions for this case.  The only change needed is to presign these transactions as v3.

# Remote commitment preimage claims

Currently these claims are not presigned, which means we currently can't force them to use TRUC.  We need to start presigning these claims as v3.

In order to presign these transactions, we need to exchange signatures for them during the commitment update flow.

## Adding a half round-trip

@t-bast has [suggested](https://github.com/lightning/bolts/issues/1221#issuecomment-2626983064) that we need to add half a round-trip to the protocol for this, similar to his draft protocol for [PTLCs](https://github.com/t-bast/lightning-docs/blob/398a1b78250f564f7c86a414810f7e87e5af23ba/taproot-updates.md#point-time-locked-contracts).  That would look something like this:

```
 Alice                      Bob
   |   commitment_proposed   |
   |------------------------>|
   |    commitment_signed    |
   |<------------------------|
   |     revoke_and_ack      |
   |------------------------>|
```

1. When Alice is ready to update her commitment transaction, she sends `commitment_proposed` with signatures for the HTLC-Remote-Success transactions Bob needs to claim HTLCs via preimage from Alice's new commitment transaction.
2. Bob sends the usual `commitment_signed` message with the HTLC-Success, HTLC-Timeout, and commitment signatures Alice needs to spend from her new commitment transaction.
3. Alice sends `revoke_and_ack` to revoke her previous commitment.

### Signing flow race

This change increases complexity due to a new race in the signing flow:

- Alice must now initiate the signing flow for her own commitment transaction by sending `commitment_proposed` with whatever HTLCs she has pending on her commitment.
- Bob may still be sending updates when he receives Alice's `commitment_proposed`.
- Thus when Bob receives Alice's `commitment_proposed` he doesn't immediately know which HTLCs need to be included in Alice's new commitment. Bob must check Alice's HTLC-Remote-Success signatures against every possible intermediate commitment transaction to figure out which one he should use for his `commitment_signed` message.

## Minimal alternative

AFAICT, we don't need to add a half round-trip or introduce a new message at all.  We can simply attach HTLC-Remote-Success signatures to the `revoke_and_ack` message:

```
 Alice                      Bob
   |    commitment_signed    |
   |<------------------------|
   |     revoke_and_ack      |
   |------------------------>|
```

1. When Bob is ready to update Alice's commitment transaction, he sends `commitment_signed` as usual.
2. Alice sends `revoke_and_ack` to revoke her previous commitment, while also sending signatures for Bob's HTLC-Remote-Success transactions.

At no point during this exchange are either Alice's nor Bob's funds at risk.

- **Before `commitment_signed`**: Alice and Bob both hold a single valid commitment transaction and all necessary HTLC transactions.
- **After `commitment_signed` but before `revoke_and_ack`**: Alice now holds a second valid commitment transaction, from which Bob cannot claim any HTLCs via preimage.  This is fine because the funds locked up in that HTLC are *Alice's*, not Bob's, and Alice can still recover those funds via HTLC-Timeout once the CLTV expires.  Since Bob wouldn't forward the HTLC anyway until he receives Alice's `revoke_and_ack` and subsequent `commitment_signed`, he has no risk of losing funds.
- **After `revoke_and_ack`**: Alice and Bob both hold a single valid commitment transaction and all necessary HTLC transactions.  Note that if Alice sends invalid HTLC-Remote-Success signatures, Bob can simply send an error and close the channel.

-------------------------

instagibbs | 2025-01-31 20:24:18 UTC | #2

So let's assume this new pattern is used, and now Bob confidently forwards that added HTLC.

After the next `commitment_signed` message from Bob to Alice, if Alice withholds the subsequent `revoke_and_ack`, cannot Alice just take that to chain and "take back" the committed HTLC Bob has forwarded?

I get turned around quite a bit in asymmetric channel states, but I feel like this is the same reasoning I wrote about PTLCs like two years ago? https://gist.github.com/instagibbs/1d02d0251640c250ceea1c66665ec163#rationale

-------------------------

