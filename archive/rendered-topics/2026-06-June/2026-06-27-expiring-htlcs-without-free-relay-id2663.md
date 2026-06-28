# Expiring HTLCs without free relay

josh | 2026-06-27 22:32:49 UTC | #1

## TLDR

It is possible to create expiring HTLCs without the possibility of free relay. This would enable preimage monitoring via the confirmed chainstate (rather than the local mempool), reducing liveness assumptions for forwarding lightning nodes and opening the door to secure routing in low-bandwidth or mempool-free environments (ex: a home router running Utreexo).

Achieving this involves three changes to consensus:
1. Give consensus meaning to bit 21 of nSequence, so that the BIP68 inclusion height enforces a minimum nLockTime.
2. Enforce bit 21 in `OP_CSV`.
3. `OP_LOCKTIME`: a tapscript-only opcode that pushes nLockTime onto the stack.

## Specification

If BIP68 is enforced and bit 21 of nSequence is set and bits 31 and 22 are unset:

1. Let $R$ be the height-based relative time lock.

2. Let $H$ be the lowest height BIP68 permits the input to be mined (coin-height + $R$).

3. Fail if $R < 100$, $\text{nLockTime} \geq 500,000,000$, or $\text{nLockTime} < H$.

When executing `OP_CSV`, fail if bits 31 and 22 of the top stack element are unset, bit 21 of the top stack element is set, and bit 21 of nSequence is unset.

`OP_LOCKTIME` is a tapscript-only opcode that replaces an `OP_SUCCESS` and pushes a minimally encoded nLockTime onto the stack (max 5 bytes).

## Use Case: HTLCs that expire without preimage publication

We can use these changes to create an HTLC that guarantees a refund if the preimage is not published at least 100 blocks before HTLC expiry. To achieve this, we make the preimage path a two-stage process:

1. Receiver publishes a preimage-gated presigned TRUC transaction that moves the HTLC funds to a staging output and has a receiver-keyed ephemeral anchor.

2. The staging output has a spending path for each user:

   a) Receiver can spend the staging output if $\text{BIP68 inclusion height} \leq \text{nLockTime} < \text{HTLC expiry}$, enforced using `CSV` (with bit 21 set), `LOCKTIME` and `LESSTHAN`.

   b) Offerer can spend the staging output after HTLC expiry, enforced with `CLTV`.

This design is motivated by the [replacement cycling attack](https://gnusha.org/pi/bitcoindev/CALZpt+GdyfDotdhrrVkjTALg5DbxJyiS8ruO2S7Ggmi9Ra5B9g@mail.gmail.com/) faced by forwarding lightning nodes and is inspired by Peter Todd's [`OP_EXPIRE` proposal](https://gnusha.org/pi/bitcoindev/ZTMWrJ6DjxtslJBn@petertodd.org/). The primary advantage over `OP_EXPIRE` is that this approach is immune to free relay. Expiry is tied to the height of the parent transaction, which must have 100 confirmations, ensuring a valid transaction cannot later become invalid without a 100 block reorg.

Unlike `OP_EXPIRE`, this approach does *not* prevent replacement cycling in the refund path. Instead, it eliminates the economic incentive to attack by guaranteeing an eventual refund in the absence of timely preimage publication. Either a routing node has sufficient opportunity to claim the inbound funds, or they have a guaranteed claim to a refund.

This approach has four limitations:

1. HTLCs cannot be shorter than 100 blocks.
2. Receiver has 100 fewer blocks to publish the preimage.
3. Receiver must wait 100 blocks to claim funds after preimage publication.
4. Receiver must pay for three transactions, rather than one.

(1) and (2) are minor issues and can be mitigated by increasing the total CLTV budget by 100 or accepting a shorter maximum route for payments. In practice, the only affected payments would be those where the last HTLC expires in ~100 blocks or less, which are quite rare.

(3) poses a minor but real liquidity constraint, but only in the atypical event that an HTLC is settled on-chain in favor of the receiver. Eliminating local-mempool preimage monitoring and improving the theoretical security of forwarding nodes may be worth tradeoffs (3) and (4).

## Rationale

The choice to use a minimum relative timelock of 100 is to prevent a mined transaction from becoming invalid through a deep chain reorganization. 100 was chosen to provide the same security guarantee as a mature coinbase output, and a similar delay is present in Peter Todd's `OP_EXPIRE` proposal.

Bit 21 was chosen for its proximity to bit 22 and its lack of current meaning in consensus. No applications (to my knowledge) currently use this bit for data availability on a BIP68-active input, so a consensus change is unlikely to be confiscatory.

The proposed changes intentionally do not support time-based enforcement, for the reason given by Peter Todd in his `OP_EXPIRE` proposal. Furthermore, a height-based time lock is always representable as a 32-bit signed integer, making it compatible with opcodes like `OP_ADD` and `OP_LESSTHAN`. This is not the case for time-based time locks after 2038, which would need [BIP441](https://github.com/bitcoin/bips/blob/764f75e33886fb05cb21fb3ebc230e7f8cb8e526/bip-0441.mediawiki) for 64-bit integer support.

It is also worth pointing out that `OP_LOCKTIME` does not guarantee timelock enforcement. This keeps the opcode minimalist, deferring responsibility to the user to ensure enforcement.

Finally, a BIP68-enforced *minimum* nLockTime provides maximum flexibility, able to satisfy multiple inputs signaling enforcement with different inclusion heights while enforcing an absolute timelock at an even greater height.

## Conclusion

This idea appears to offer a narrowly scoped alternative to `OP_EXPIRE` that removes the potential for free relay while eliminating local-mempool preimage monitoring. I'm curious to hear opinions about this approach and the appetite for mempool-free HTLC routing and transaction expiry in general.

Appreciate the feedback!

-------------------------

