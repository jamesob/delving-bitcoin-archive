# Expiring HTLCs without free relay

josh | 2026-06-28 03:08:39 UTC | #1

## TLDR

It is possible to create expiring HTLCs without the possibility of free relay. This would enable preimage monitoring via the confirmed chainstate (rather than the local mempool), reducing liveness assumptions for forwarding lightning nodes and opening the door to secure routing in low-bandwidth or mempool-free environments (ex: a home router running Utreexo).

Achieving this requires one change to consensus:
1. Give consensus meaning to bit 21 of nSequence, so that the BIP68 inclusion height enforces a minimum nLockTime.

And two additional changes enable an optimized implementation:

2. Enforce bit 21 in `OP_CSV`.
3. `OP_LOCKTIME`: a tapscript-only opcode that pushes nLockTime onto the stack.

## Specification

#### `nSequence` enforced minimum `nLockTime`

If BIP68 is enforced and bit 21 of nSequence is set and bits 31 and 22 are unset:

1. Let $R$ be the height-based relative time lock.

2. Let $H$ be the lowest height BIP68 permits the input to be mined (coin-height + $R$).

3. Fail if $R < 100$, $\text{nLockTime} \geq 500,000,000$, or $\text{nLockTime} < H$.

#### `OP_CSV` enforcement (optional)

When executing `OP_CSV`, fail if bits 31 and 22 of the top stack element are unset, bit 21 of the top stack element is set, and bit 21 of nSequence is unset.

#### `OP_LOCKTIME` introspection (optional)

`OP_LOCKTIME` is a tapscript-only opcode that replaces an `OP_SUCCESS` and pushes a minimally encoded nLockTime onto the stack (max 5 bytes).

## Use case: HTLCs that expire without preimage publication

With only an `nSequence` enforced minimum `nLockTime`, we can create an HTLC that guarantees a refund if the preimage is not published at least 100 blocks before HTLC expiry. To achieve this, we make the preimage path a two-stage process:

1. Receiver publishes a preimage-gated presigned TRUC transaction (Tx1) that moves the HTLC funds to a staging output and has a receiver-keyed ephemeral anchor.

2. The staging output has two spending paths:

   a) A presigned TRUC transaction (Tx2) with a single output transferring the HTLC funds to the receiver, where `nSequence` enforces a 100 block relative timelock with bit 21 set and `nLockTime` is set to the HTLC expiry.

   b) A script-path allowing the offerer to spend the staging output $N$ blocks after HTLC expiry, enforced with `CLTV`, where $N$ is the window to broadcast Tx2.

This design is motivated by the [replacement cycling attack](https://gnusha.org/pi/bitcoindev/CALZpt+GdyfDotdhrrVkjTALg5DbxJyiS8ruO2S7Ggmi9Ra5B9g@mail.gmail.com/) faced by forwarding lightning nodes and is inspired by Peter Todd's [`OP_EXPIRE` proposal](https://gnusha.org/pi/bitcoindev/ZTMWrJ6DjxtslJBn@petertodd.org/). The primary advantage over `OP_EXPIRE` is that this approach is immune to free relay. Expiry is tied to the height of the parent transaction, which must have 100 confirmations, ensuring a valid transaction cannot later become invalid without a 100 block reorg.

Unlike `OP_EXPIRE`, this approach does *not* prevent replacement cycling in the refund path. Instead, it eliminates the economic incentive to attack by guaranteeing an eventual refund in the absence of timely preimage publication. Either a routing node has sufficient opportunity to claim the inbound funds, or they have a guaranteed claim to a refund.

This approach has four limitations:

1. HTLCs cannot be shorter than 100 blocks.
2. Receiver has 100 fewer blocks to publish the preimage.
3. Receiver must wait until HTLC expiry to claim funds after preimage publication.
4. Receiver must pay for multiple transactions, rather than one.

(1) and (2) are minor issues and can be mitigated by increasing the total CLTV budget by 100 or accepting a shorter maximum route for payments. In practice, the only affected payments would be those where the last HTLC expires in ~100 blocks or less, which are quite rare.

(3) poses a minor but real liquidity constraint, but only in the atypical event that an HTLC is settled on-chain in favor of the receiver, and the optimization below can provide some mitigation. (4) is a problem only for small HTLCs, which might not be able to justify fees in a high fee environment. Eliminating local-mempool preimage monitoring and improving the theoretical security of forwarding nodes may be worth these tradeoffs.

## Optimized approach

We can optimize the preimage path with `OP_CSV` enforcement and `OP_LOCKTIME` introspection:

1. Broadcast preimage-gated staging transaction (Tx1).

2. The staging output has two spending paths:

   a) Receiver can spend the staging output if $\text{BIP68 inclusion height} \leq \text{nLockTime} < \text{HTLC expiry}$, enforced using `CSV` (with bit 21 set), `LOCKTIME` and `LESSTHAN`.

   b) Offerer can spend the staging output after HTLC expiry, enforced with `CLTV`.

This optimization has two notable benefits:
1. Receiver can claim funds before HTLC expiry.
2. Offerer can claim a refund at HTLC expiry without a delay.

(1) mitigates Limitation 3 in the unoptimized approach, and (2) ensures the offerer is not penalized by a late broadcast of the staging transaction.

## Rationale

#### `nSequence` enforced minimum `nLockTime`

The choice to use a minimum relative timelock of 100 is to prevent a mined transaction from becoming invalid through a deep chain reorganization. 100 was chosen to provide the same security guarantee as a mature coinbase output, and a similar delay is present in Peter Todd's `OP_EXPIRE` proposal.

Bit 21 was chosen for its proximity to bit 22 and its lack of current meaning in consensus. No applications (to my knowledge) currently use this bit for data availability on a BIP68-active input, so a consensus change is unlikely to be confiscatory.

The proposed change intentionally does not support time-based enforcement, for the reason given by Peter Todd in his `OP_EXPIRE` proposal.

Finally, a BIP68-enforced *minimum* nLockTime provides maximum flexibility, able to satisfy multiple inputs signaling enforcement with different inclusion heights while enforcing an absolute timelock at an even greater height.

#### `OP_CSV` enforcement

`OP_CSV` currently ignores bit 21, so a soft fork is needed to add enforcement. The most minimalist approach is to restrict enforcement to stack elements that leave bit 31 and 22 unset, as bit 21 has consensus meaning only in that context.

#### `OP_LOCKTIME` introspection

CLTV enforces a minimum `nLockTime`, but the described use case requires enforcement of a *maximum* value, so a new opcode is needed.

`OP_LOCKTIME` does not guarantee timelock enforcement, but it enables the required capability when combined with `CSV` bit 21 enforcement and `LESSTHAN`. This keeps the opcode minimalist, deferring responsibility to the user to ensure enforcement.

Note as well that a height-based timelock is always representable as a 32-bit signed integer, making it compatible with opcodes like `OP_ADD` and `OP_LESSTHAN`. This is not the case for time-based time locks after 2038, which would need [BIP441](https://github.com/bitcoin/bips/blob/master/bip-0441.mediawiki) for 64-bit integer support.

## Conclusion

This idea appears to offer a narrowly scoped alternative to `OP_EXPIRE` that removes the potential for free relay while eliminating local-mempool preimage monitoring. I'm curious to hear opinions about this approach and the appetite for mempool-free HTLC routing and transaction expiry in general.

Appreciate the feedback!

-------------------------

instagibbs | 2026-06-29 13:54:53 UTC | #2

Thanks for the interesting idea! Not having to worry about free relay is a nice benefit.

[quote="josh, post:1, topic:2663"]
Unlike `OP_EXPIRE`, this approach does *not* prevent replacement cycling in the refund path. Instead, it eliminates the economic incentive to attack by guaranteeing an eventual refund in the absence of timely preimage publication. Either a routing node has sufficient opportunity to claim the inbound funds, or they have a guaranteed claim to a refund.

[/quote]

Took me a while to get the claim, probably should have started with this. Many implementations do no mempool monitoring at all, so I was confused what this was accomplishing. What the claim is is that with proper expiry, replacement cycling is a just a more time insensitive grief vector, which makes the mempool scanning some may do as a partial mitigation, like LND , completely unnecessary. Wtxid grinding and rebroacasting also become unnecessary at least in theory (I think there are valid reasons to do it anyway)

-------------------------

josh | 2026-07-12 00:46:55 UTC | #3

[quote="instagibbs, post:2, topic:2663"]
Took me a while to get the claim, probably should have started with this. Many implementations do no mempool monitoring at all, so I was confused what this was accomplishing. What the claim is is that with proper expiry, replacement cycling is a just a more time insensitive grief vector, which makes the mempool scanning some may do as a partial mitigation, like LND , completely unnecessary. Wtxid grinding and rebroacasting also become unnecessary at least in theory (I think there are valid reasons to do it anyway)
[/quote]

Thanks for clarifying the idea! I was under the impression that most implementations did mempool monitoring, so to hear otherwise is a surprise.

As for grinding and rebroadcasting, I had hoped that this proposal would make those unnecessary, but my primary motivation was relay-safe expiry and the elimination of the time-sensitive replacement cycling attack.

Lastly, I would add that [per](https://delvingbitcoin.org/t/input-triggered-transaction-expiry/2667/3?u=josh) @ajtowns, it may be reasonable to eliminate the 100 block delay entirely. I included it here to be conservative and match the 100 block delay in `OP_EXPIRE`, but consensus and relay may be unaffected by a substantially lower number. I plan to explore this in a future post.

-------------------------

