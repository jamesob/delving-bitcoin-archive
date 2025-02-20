# Zero-fee commitments for mobile wallets

t-bast | 2025-02-18 13:06:24 UTC | #1

Work is currently ongoing to add support for [zero-fee commitments](https://github.com/lightning/bolts/pull/1228) to lightning channels.

In this post, I'd like to share my ideas on how this commitment format can be tweaked for mobile wallets, and gather feedback from the community.

This will eventually be translated into a [bLIP](https://github.com/lightning/blips) once we agree on a satisfying solution.

Let's first take a look at the threat model for mobile wallets: since mobile wallets don't relay payments, they have less attack surface than routing nodes.

However, most mobile wallets don't have on-chain utxos available: our challenge is to find ways to get force-close transactions confirmed using existing channel outputs only.

## Mobile wallet funds safety

### Revoked commitments

Mobile wallets must be able to publish penalty transactions if their peer broadcasts a revoked commitment.

This will work trivially with zero-fee commitments: channel outputs can be spent immediately (no CSV delay), so they can be used to pay the on-chain fees. We don't have to change anything here compared to the BOLTs.

### Received HTLCs

When receiving HTLCs, a mobile wallet is the final recipient. The only way the peer can steal funds is:

- the mobile wallet fulfills the HTLC (by sending `update_fulfill_htlc` with the preimage)

- the peer then forwards that preimage to its own peers, which ensures that it has been paid (the payer receives the preimage)

- they go silent and don't revoke their previous commitment (which contains the HTLC)

- they wait for the HTLC to timeout to try to claim it on-chain

Before the HTLC timeout, the mobile wallet must:

- broadcast their commitment transaction

- broadcast their HTLC-success transaction

- get those two transactions confirmed

### Sent HTLCs

When sending HTLCs, a mobile wallet is the payer: funds thus can never be stolen by the peer. If the payment succeeds, the peer must reveal the preimage to claim the funds, at which point the mobile wallet has a proof of payment.

An interesting thing to note is that mobile wallets are never in a rush to claim HTLCs when they timeout. They don't have funds at stake in an upstream channel since they are the payer, so they could potentially wait longer to see if their peer reveals the preimage.

Even after force-closing and publishing their HTLC-timeout transaction, if their peer publishes an HTLC-success transaction, the mobile wallet has not lost any funds: the payment can instead simply be considered fulfilled.

So the only thing the peer can do is griefing (not stealing):

- the mobile wallet sends an HTLC

- this HTLC times out

- the peer never fails the HTLC, which remains in the commitment transaction

- the funds used by that HTLC cannot be re-used until the HTLC is failed

At that point, to recover their funds, the mobile wallet must:

- broadcast their commitment transaction

- broadcast their HTLC-timeout transaction

- get those two transactions confirmed

Note that there is no deadline before which those transactions must confirm and the peer doesn't have anything to gain from this griefing.

### Unresponsive peer without HTLCs

When there are no pending HTLCs, no funds are at risk.

But the mobile wallet user can still be griefed if their peer becomes unresponsive or disappears.

To recover their funds, they must be able to:

- broadcast their commitment transaction

- claim their main output

Note that there is no deadline before which those transactions must confirm and the peer doesn't have anything to gain from this griefing.

## Additional signatures for HTLC transactions

With zero-fee commitments [as proposed in the BOLTs](https://github.com/lightning/bolts/pull/1228), mobile wallets will mostly rely on their peer publishing their commitment transaction. This way the mobile wallet can use their main output (or any HTLC output) to CPFP that commitment transaction and get it confirmed.

The main issue arises when the peer isn't cooperative and the mobile wallet has to publish their commitment transaction. That transaction usually has a `0 sat` anchor output and the main output has a `to_self_delay` CSV. So the only option to CPFP without additional inputs is to use HTLC transactions, but they are pre-signed and don't pay fees either.

A very simple proposal to fix that is to have the peer always sign two versions of HTLC transactions:

- the default one that doesn't pay any fee and needs additional on-chain inputs

- and another one in a custom TLV of `commitment_signed` at a high feerate that matches the currently observed feerate

- if the peer doesn't provide those signatures, the mobile wallet can force-close before revoking the commitment that doesn't contain the new HTLCs

This way, when HTLCs are pending, the mobile wallet can always publish their commitment transaction and CPFP it using one of the HTLC transactions. Since funds can only be stolen for received HTLCs, which should expire somewhat quickly after being received, fee estimation can be somewhat accurate.

With this simple addition, mobile wallets are able to unilaterally force-close when HTLCs are pending.

The only scenario that isn't fixed is the "Unresponsive peer without HTLCs" scenario. But this scenario can never be fixed by pre-signing transactions at various feerates, because we have no idea at which point in the future the peer will become unresponsive. On top of that, funds are not at risk in this scenario, which should only happen when an LSP completely disappears without closing channels. I think it's acceptable that when this happens, mobile wallet users will need *someone* (which could be an on-chain wallet they own) to spend the anchor output to CPFP the commitment transaction.

I like this proposal because it is trivial to implement and doesn't require a lot of changes compared to the default zero-fee commitment format. Please let me know what you think, and if you have other ideas on how we could make zero-fee commitments work seamlessly with mobile wallets.

-------------------------

harding | 2025-02-18 14:31:14 UTC | #2

[quote="t-bast, post:1, topic:1453"]
A very simple proposal to fix that is to have the peer always sign two versions of HTLC transactions:

* the default one that doesn’t pay any fee and needs additional on-chain inputs
* and another one in a custom TLV of `commitment_signed` at a high feerate that matches the currently observed feerate
[/quote]

This seems pretty clever.  Am I understanding correctly that the fees for the high feerate will be deducted from the balance of the channel opener in the current way?  Since we would always expect the mobile wallet to be the channel opener, that means (1) it costs the peer nothing to offer the mobile wallet a high feerate option and (2) the mobile wallet has an incentive to not use the high-feerate option unless it's actually necessary for safety or recovering liquidity from a stale channel.

-------------------------

t-bast | 2025-02-18 14:54:25 UTC | #3

[quote="harding, post:2, topic:1453"]
Am I understanding correctly that the fees for the high feerate will be deducted from the balance of the channel opener in the current way?
[/quote]

Not exactly the channel opener, but rather the owner of the HTLC transaction (ie the mobile wallet), which is even better (and as you highlight, doesn't cost anything to the peer): those fees are deducted from the output of the HTLC tx. Your conclusions correctly apply though!

The only thing to be careful about is that we shouldn't use an unreasonably high feerate, otherwise the following attack can be performed by the mobile wallet user:

- the mobile wallet receive a batch of HTLCs for which the fee-paying pre-signed transaction consumes almost all of the HTLC amount to fees
- the mobile wallet then fulfills those HTLCs
- later, they publish the revoked commitment that contains all of those HTLCs
- since most of the value goes to mining fees, it cannot be claimed by the peer as penalty transactions

This type of attack is why we had to make HTLC transactions pay 0 fees in anchor outputs to protect against miners attacking lightning peers. But in this specific case, the LSP can mitigate that risk by not overshooting the feerate and/or trying to detect that kind of behavior and refusing to relay "risky" HTLCs. Details need to be fleshed out for that, but I'm not too worried, I think we can come up with something that guarantees that the LSP doesn't take too much risk as long as they have some honest users.

-------------------------

t-bast | 2025-02-19 09:44:53 UTC | #4

Let me clarify my previous comment about revoked commitments, which was too vague. What a malicious wallet can do to its peer to abuse the additional pre-signed HTLC transactions is the following:

- open a channel
- wait for the mempool feerate to be high
- send its whole balance out (to another node it controls)
- once the HTLCs are fulfilled, it only has its channel reserve at stake (which may be `0 sat` if the wallet provider allows zero-reserve)
- broadcast the revoked commitment where the HTLCs were still pending
- broadcast the HTLC transactions that pay non-zero fees

The peer will be able to publish penalty transaction to claim the outputs of those HTLC transactions. But it won't be able to claim the HTLC transaction fees, which will go to miners. In that case, it's similar to the wallet peer paying the on-chain fees, whereas they should be paid by the wallet user.

This is the risk taken by the wallet peer when offering those pre-signed HTLC transactions. This risk is however offset by the following facts:

- the wallet peer should have earned fees for the channel creation
- the wallet peer earned routing fees for the outgoing HTLC
- the wallet peer decides the feerate, and could cap it to X% of the HTLC amount
- splice transactions make it impossible to publish revoked commitments that happened before the splice
- pathological scenarios like the one above can be detected and HTLCs can be failed instead of relayed when it looks risky
- the wallet peer can limit the risk they take until they've earned enough fees from the wallet user (from routing or liquidity purchases)

I think this is a risk that is worth taking by wallet providers (who are betting they will have enough honest users anyway) to provide good lightning trust trade-offs for their users.

-------------------------

harding | 2025-02-19 14:21:01 UTC | #5

[quote="t-bast, post:1, topic:1453"]
A very simple proposal to fix that is to have the peer always sign two versions of HTLC transactions:

* the default one that doesn’t pay any fee and needs additional on-chain inputs
* and another one in a custom TLV of `commitment_signed` at a high feerate that matches the currently observed feerate
[/quote]

Will the second (custom) variant also include a zero-fee P2A?  If so, doesn't the mobile wallet still need another UTXO to spend the P2A output to make the transaction relayable under the ephemeral dust policy?

-------------------------

t-bast | 2025-02-19 14:46:22 UTC | #6

I'm not sure I understand the issue: let me describe with more details that pre-signed transaction, that should help figure out if it works or not!

Let's assume we have one pending incoming HTLC of 20 000 sat (from the mobile wallet's point of view). Outgoing HTLCs will work exactly the same.

If we only follow the BOLTs, the HTLC-success transaction that is signed by the LSP will be:

- one input (the corresponding 20 000 sat HTLC output from the commitment transaction)
- one output with amount 20 000 sat (the transaction doesn't pay any fee)
- it is signed with `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY`

I proposed also signing a second version of that HTLC-success transaction:

- one input (the corresponding 20 000 sat HTLC output from the commitment transaction)
- one output with amount 17 500 sat (or a different value based on the feerate chosen)
- it is still signed with `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY`

If the mobile wallet needs to force-close and the LSP isn't cooperating, it will publish a package containing:

- its commitment transaction, which contains:
  - its main output, which has a long CSV delay
  - the LSP's main output
  - the shared anchor, which is likely 0 sat
  - the HTLC output
- the second pre-signed HTLC-success transaction, modified to also spend the shared anchor, with the mobile wallet adding a `SIGHASH_ALL` signature

This package pays 2500 sats of fees, and I believe this should work fine with ephemeral dust policy as it spends the commitment's dust P2A?

-------------------------

harding | 2025-02-19 15:14:08 UTC | #7

[quote="t-bast, post:6, topic:1453"]
the second pre-signed HTLC-success transaction, modified to also spend the shared anchor, with the mobile wallet adding a `SIGHASH_ALL` signature
[/quote]

That LGTM.  Thanks for writing it out in detail!

-------------------------

morehouse | 2025-02-19 16:11:27 UTC | #8

[quote="t-bast, post:6, topic:1453"]
I proposed also signing a second version of that HTLC-success transaction:

* one input (the corresponding 20 000 sat HTLC output from the commitment transaction)
* one output with amount 17 500 sat (or a different value based on the feerate chosen)
* it is still signed with `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY`
[/quote]

We would want to sign this new variant with `SIGHASH_ALL`.

The only reason to do `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY` is so the mobile user can add their own input and output to pay fees, which we're trying to avoid here.  And allowing the user to add their own outputs means that they can claim any excess mining fees for *themselves* when broadcasting a revoked commitment+HTLC package.

In the above example, the wallet user could broadcast the revoked package and claim some portion of the 2500 sat intended for fees.  Depending on the size of the channel and whether the LSP allows zero-reserve, it shouldn't be difficult for the user to profit from this.

-------------------------

t-bast | 2025-02-19 16:18:32 UTC | #9

Right, good catch @morehouse! The additional pre-signed transaction should indeed then include the P2A output from the commitment transaction and use `SIGHASH_ALL`. If the mobile wallet can actually leverage `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY` by adding inputs to pay fees, they should use the default 0-fee HTLC transaction.

-------------------------

morehouse | 2025-02-19 18:28:29 UTC | #10

[quote="t-bast, post:1, topic:1453"]
A very simple proposal to fix that is to have the peer always sign two versions of HTLC transactions:

* the default one that doesn’t pay any fee and needs additional on-chain inputs
* and another one in a custom TLV of `commitment_signed` at a high feerate that matches the currently observed feerate
* if the peer doesn’t provide those signatures, the mobile wallet can force-close before revoking the commitment that doesn’t contain the new HTLCs
[/quote]

How will feerates be negotiated between the user and the LSP?  I think this might add some complexity...

### User decides, LSP accepts/rejects

If we just use what exists today, the user needs to periodically send `update_fee` to the LSP, so that the LSP uses the correct feerate when forwarding HTLCs to the user.  But if the user is offline for a while (as mobile users often are), the LSP may end up forwarding an HTLC with too low of a feerate.  In that case, the user would need to NACK the HTLC somehow.  Currently there's three ways to (kind of) do this:

1. Accept the HTLC, sign updated commitments, and then fail it.
2. Force close.
3. Disconnect and reconnect with the peer.

Option 1 might work if we're careful.  A state is created where the HTLC is accepted and the user's commitment+HTLC package won't confirm in a timely manner.  But the current HTLC is safe from theft because the user will fail it without revealing the preimage.  The only concern is theft of any *previous* HTLCs on the commitment for which the user has already revealed the preimage.  It seems like a rare corner case, but we'd need to think carefully about how to avoid or get out of that situation.  Option 1 also has bad UX since the entire payment gets failed back to the sender even though the payment path was fine.

Option 2 protects from theft, but also sucks from a UX perspective.  Especially if the the previous commitment has no HTLCs on it -- then the user *can't* force close and would need to wait for the HTLC to expire and for the LSP to force close.

In theory, Option 3 could work.  But in practice today's implementations simply resend the same commitment updates on reconnection, so I don't think the user would be able to do `update_fee` before the LSP resends the same commitment and HTLC signatures.

### LSP decides, user accepts/rejects

The LSP could just choose the feerate on its own and use it when forwarding HTLCs to the user.  If the user thinks the feerate is too high/low, they (once again) need some way to NACK the HTLC.  In that case we have the same options and problems described above.

To be fair, this feerate disagreement should only be expected during times of feerate spikes, which current lightning channels already have issues with.  But zero-fee commitments were supposed to fix this issue, so it's unfortunate that this approach brings it back.

### Add a new NACK mechanism

I think there was discussion about adding a NACK mechanism to `option_simplified_update` at some point, but the [current spec proposal](https://github.com/lightning/bolts/pull/867) doesn't have it.  We could either wait for that, or implement something special purpose for this case.  Either one is probably more work than we want.

-------------------------

t-bast | 2025-02-20 08:43:54 UTC | #11

I was leaning towards LSP decides and user accepts or rejects through option 1. I think this is the simplest way to implement this feature that doesn't lead to unwanted force-closes: also, since the LSP is always online and runs a bitcoin node, they have a more precise view of the feerate than the mobile wallet user will ever have.

There is a risk that the LSP uses a feerate that is too low during feerate spikes that aren't detected by mobile wallet users in time: since by design mobile wallet users will never have a perfectly reliable view of the mempool/feerate (and nobody can predict the future), I don't think this can be completely fixed, so we'll have to live with the fact that it's not a perfect solution, but is still an improvement over the current anchor output channels!

Mobile wallet users should also use longer `min_final_expiry_delta` than what they use today (which is between 24 and 36 blocks for Phoenix) to have more time to potentially get their HTLC-success transaction confirmed.

I don't think we can ever have a 100% secure solution for mobile wallet users unless they use some form of watch-tower (which most people won't do): my goal is to raise the bar to make it as hard as possible for LSPs to cheat, without making the protocol more complex than it needs to be :man_shrugging: 

[quote="morehouse, post:10, topic:1453"]
Add a new NACK mechanism
[/quote]

I think this hasn't been added to the existing PR simply because it hasn't been updated in a while: it will probably be added once someone is ready to work on implementing this feature.

-------------------------

