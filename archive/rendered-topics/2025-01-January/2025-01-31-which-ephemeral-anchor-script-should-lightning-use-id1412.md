# Which ephemeral anchor script should lightning use?

t-bast | 2025-01-31 15:49:30 UTC | #1

In this post, I'd like to explore the various options available for the ephemeral anchor output used in future lightning commitment transactions that use `nVersion = 3`. We could go in a few different directions, and every option has a different set of trade-offs.

Let's start with a brief summary of how this ephemeral anchor output will behave in the context of lightning channels.

## Ephemeral anchor in lightning commitment transactions

When there are no pending dust HTLCs, the amount of the anchor will be `0 sat`.

In order to spend it (to CPFP the commitment transaction), nodes will need to add additional inputs to pay fees.

When HTLCs are pending that are below the `dust_limit` defined by the channel participants, the amount of those HTLCs will be:

- subtracted from the sending node's main output

- added to the anchor output

Note that this means that the anchor output amount will contain funds that may come from both participants, depending on who sent dust HTLCs.

This can result in the anchor output amount being large enough to pay the commitment fees on its own, depending on how many pending dust HTLCs nodes allow on the channel. This can be a good thing because it may allow CPFP on the commitment transaction without adding external wallet inputs. But the drawback is that if the anchor output amount is larger than the on-chain fees that need to be paid, there is a race between everyone who can spend that output to collect the remaining amount.

For example, let's assume that the anchor output amount is `50 000 sat` but Alice only needs `20 000 sat` to pay enough on-chain fees for the commitment transaction to be confirmed. So Alice will create an `anchor-tx` that spends the anchor output and sends `30 000 sat` back to her address. But when Bob sees that transaction, he will most likely replace it with a different `anchor-tx` transaction that sends `25 000 sat` to his own address, and this will be a more interesting package for miners!

## Option 1: unkeyed anchor

Ephemeral anchors can be used with the new `P2A` output type (`OP_1 4e73`) which can be spent by anyone.

The benefits of this option are that:

- it can easily be spent by both channel participants

- since anyone can spend it, nodes can even delegate paying the on-chain fees to someone else

- this is the most economical way to spend an anchor output (no witness data)

The main drawback of this option is that when the anchor output amount is larger than the on-chain fees required for confirmation, miners will always claim the whole amount: when they see the transaction spending the anchor output, they always have an incentive to replace it with one that sends the funds to themselves. This can lead to a non-negligible overpaying of on-chain fees. However, note that this is already what happens in today's lightning channels, since pending dust HTLCs are currently directly added to mining fees (they're subtracted from the sender's main output but not added to any output).

I don't see other drawbacks to this solution: am I missing something?

## Option 2: single-participant keyed anchor

In this variant, we use a keyed script (either `p2wpkh` or `p2tr`) for the ephemeral anchor, paying to the node's `local_funding_pubkey`. This means that channel participants can only spend the anchor output of *their* commitment transaction. While this cannot be done with v2 commitments because of pinning, this is actually fine for v3 commitments: if the remote commitment has been published but doesn't confirm, you can replace it with your local commitment using v3 package RBF (but then you'll need to wait for the `to_self_delay` to spend your channel balance).

The benefits of this option are:

- it isn't too costly to spend (no script path)

- miners cannot steal the on-chain fee over-payment

- your channel counterparty may steal the on-chain fee over-payment, but only by getting *their* commitment transaction confirmed, thus locking their funds until `to_self_delay`

The drawbacks are that you cannot delegate paying the on-chain fees, and you may be forced to publish your local commitment if the remote commitment doesn't confirm, even though you weren't the node who decided to force-close.

## Option 3: shared key anchor

In this variant, we still use a keyed script (either `p2wpkh` or `p2tr`) for the ephemeral anchor, but it pays to a public key created by the channel opener, who shares the private key with the other node when opening the channel.

This option has the following benefits:

- it can be spent by both channel participants

- participants can CPFP the remote commitment (which lets them spend their balance immediately)

- by revealing the private key (which shouldn't be used for anything else), it becomes possible to delegate paying the on-chain fees

- miners cannot steal the on-chain fee over-payment

- but anyone you share that private key with can steal the on-chain fee over-payment

This option is very similar to option 1, but fixes the fact that miners can steal the over-payment, at the cost of a larger witness.

## Option 4: dual-keyed anchor

In this option, we use a taproot output for the ephemeral anchor where:

- the key path uses the `local_funding_pubkey`

- a script path is included that uses the `remote_funding_pubkey`

The benefits of this option are:

- it can be spent by both channel participants

- CPFP-ing the local commitment transaction is efficient (key-path spend)

- CPFP-ing the remote commitment transaction is less efficient (script-path spend) but you can spend your channel funds without a `to_self_delay`

- the on-chain fee over-payment can only be stolen by your channel peers

The drawback is that you cannot delegate paying the on-chain fees.

---

I haven't decided yet which option I like best: I think it mostly depends on how much we care about preventing on-chain fee over-payment.

Since nodes can decide how much dust they allow in the commitment, it is somewhat easy to limit this exposure, so maybe we don't care about it at all?

-------------------------

instagibbs | 2025-01-31 15:56:20 UTC | #2

Note that you aren't stuck with one format. You could reasonably consider p2a when the anchor is 0-value, and keyed otherwise.

re:miner "stealing" the funds, I'm not sure that's bad at all? The main new weirdness is counterparty may be tempted to ramp up the trimmed amount and take it themselves, even though in the end miners will probably do well to snipe that entire value.

-------------------------

