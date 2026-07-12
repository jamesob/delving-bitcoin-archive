# Input-triggered transaction expiry

josh | 2026-06-29 02:25:47 UTC | #1

## TLDR

Input-triggered transaction expiry is an interesting primitive that warrants further research. It can be implemented with a minimalist consensus change:

> If `nSequence` enforces a height-based relative timelock $R$ and bit 21 is set, fail if $R < 100$, `nLockTime` is time-based, or the BIP68 min-height exceeds `nLockTime`.

This change is surprisingly powerful. It enables both mempool-free HTLC forwarding and (pseudo) contract-level relative timelocks for LN-Symmetry.

Both these applications would also be possible using classical expiry (see `OP_EXPIRE`), but unlike classical expiry, input-triggered expiry is immune to free relay.

## Background

Plain transaction expiry is a well-researched topic. The most fleshed out proposal appears to be Peter Todd's [`OP_EXPIRE`](https://gnusha.org/pi/bitcoindev/ZTMWrJ6DjxtslJBn@petertodd.org/), which invalidates a transaction after a certain block height.

The proposal requires several consensus changes:
1. An `nVersion` flag that enforces a 100-block maturity rule on all outputs.
2. A new `OP_EXPIRE` opcode.
3. A taproot annex that makes the expiry height explicit.

The primary motivation of `OP_EXPIRE` was the replacement cycling attack in HTLC forwarding. By expiring the preimage path at the HTLC-timeout, we stop the attack, allowing us to replace local-mempool preimage monitoring with chainstate monitoring.

The primary critique was the potential for free relay. If a valid transaction later becomes invalid, an attacker could spam the P2P network at little expected cost. Todd proposes a relay policy requiring a fee sufficient to be mined in the next block, but the community found this unsatisfactory (presumably because probabilistic free relay is still possible).

## Input-triggered transaction expiry

An input-triggered approach to transaction expiry is less powerful, but it is immune to free relay and fits naturally in the existing transaction context. The main idea is to expire a transaction if an input was mined too late.

This is enforceable with only the following consensus change:

> If `nSequence` enforces a height-based relative timelock $R$ and bit 21 is set, fail if $R < 100$, `nLockTime` is time-based, or the BIP68 min-height exceeds `nLockTime`.

For example, to enforce expiry by height $H$, a transaction would set bit 21 in `nSequence`, enforce a height-based relative timelock $R = 100$, and set $\text{nLockTime} = H$. Mining the input after $H-100$ will then trigger expiry.

The 100 block minimum relative timelock ensures a valid transaction cannot become invalid without a 100 block reorg, guaranteeing the same level of maturity as a coinbase output and preventing free relay.

The most obvious drawback is that this approach imposes an absolute timelock of $H$, preventing early publication if the input is mined early. While not ideal, this is desirable in some use cases (such as UC2) and can be entirely avoided using introspection (see [this post](https://delvingbitcoin.org/t/expiring-htlcs-without-free-relay/2663)).

## Use case 1: mempool-free HTLC forwarding

Like `OP_EXPIRE`, input-triggered transaction expiry enables HTLC forwarding without local-mempool preimage monitoring. A detailed construction can be found in [this post](https://delvingbitcoin.org/t/expiring-htlcs-without-free-relay/2663).

## Use case 2: (pseudo) contract-level relative timelocks

### Background
In classical LN-Symmetry, each update transaction resets the relative timelock. This delays HTLC-timeout by 2x in a 2-party channel and by a factor of $N$ in an $N$-party channel, locking up liquidity.

The most efficient solution is a [contract-level relative timelock](https://delvingbitcoin.org/t/contract-level-relative-timelocks-or-lets-talk-about-ancestry-proofs-and-singletons/1353), which enforces a relative timelock on settlement based on the height of a kickoff transaction. Unfortunately, CLRTs appear to require a succinct ancestry proof or a way to propagate the kickoff height. Neither is possible today, and adding either capability would require a significant change to consensus.

### Summary
A (pseudo) contract-level relative timelock can be constructed using input-triggered expiry, at the cost of increased interactivity, storage, and periodic refresh. With vector commitments, these additional costs can be eliminated with off-chain computation.

### Construction

Consider the following force-closure path in a modified LN-Symmetry (assume `CSFS` and `TEMPLATEHASH`):

1. A kickoff TRUC transaction (Tx1) moves the channel funds to a kickoff output (committed to at setup using `TEMPLATEHASH`), paired with an ephemeral anchor.

2. A presigned window TRUC transaction (Tx2) moves the funds to an update output controlled by a window-unique public key $P_{H}$, with no ephemeral anchor. The transaction sets `nLockTime` to $H$ and enforces bit 21 of `nSequence` with a 100 block relative locktime.

3. One or more update transactions are published, followed by the latest settlement transaction. 

4. The settlement transaction commits to a non-final `nSequence` and sets `nLockTime` to $H + C$, where $C$ is the desired challenge window.

> **Key idea:** Due to bit 21 `nSequence` enforcement, Tx2 expires if Tx1 is not mined by $H-100$. This expires all update transactions signed with $P_{H}$, enforcing a pseudo contract-level relative locktime on the settlement transaction.

This functionality comes at the cost of increased interactivity and storage. With each state update, channel parties sign $N$ update transactions for windows beginning over the next $N$ blocks. If there is no presigned Tx2 for height $H$, that transaction is signed as well.

Periodically, channel parties also refresh the current state by signing new update transactions committing to the same state outputs. This ensures a force-closure path is always available.

### Optimizations

An opcode that enables vector commits (i.e., `PAIRCOMMIT`, `CAT`, etc.) enables merkle proof verification, which eliminates the added interactivity and storage requirements and the need to refresh. This involves four changes:
1. All update outputs are controlled by the same public key $P$.
2. The window height $H$ is committed to in the update script.
3. Parties sign a merkle root approving $N$ windows, where each leaf commits to an update template hash and window height $H$.
4. The update script verifies the signed template hash and window via a merkle proof.

With this optimization, channels can trivially support millions of windows, removing the need to refresh in practice.

### Force-closure walkthrough

Consider a two-party channel between Alice and Bob, where Alice wishes to force close.

1. First, Alice publishes the kickoff transaction (Tx1), notifying Bob of the force closure.

2. Suppose Tx1 is mined at block $K$. This invalidates every window transaction (Tx2) where $\text{nLockTime} < K + 100$, invalidating in turn every update transaction committing to a settlement transaction where $\text{nLockTime} < K + 100 + C$, where $C$ is the agreed-upon challenge window.

3. After 100 blocks, Alice may choose any Tx2 to publish, provided `nLockTime` is less than or equal to the current block height. 

4. If Alice is honest, she will publish the Tx2 with the lowest valid locktime. If Alice is malicious:
  
   a) She will withhold Tx2 and force Bob to publish instead. (Bob must publish a Tx2 before the earliest valid window ends so that he has the opportunity to stop a stale commitment).

   b) She will publish the Tx2 with the highest valid locktime, delaying settlement.

5. Alice publishes an update transaction, committing to a settlement transaction.

6. If the update transaction is out-of-date, Bob publishes an update transaction with the latest state.

7. Alice publishes the settlement transaction after the absolute settlement timelock expires.

### Security

This construction inherits most of the security properties of LN-Symmetry with an ideal CLRT, but it is perhaps most interesting with vector commitments, which remove the added refresh and interactivity requirements.

The most significant limitation is the added 100-block delay in the force closure path. This raises the minimum HTLC-timeout, which reduces the maximum payment length. In practice, most payments would be unaffected, as the total CLTV budget is sufficiently high, but the total CLTV budget can be increased by 100 if needed.

## Final thoughts

Relay-safe transaction expiry seems like a genuinely useful primitive, and I find it surprising that it can be implemented via a consensus change with such a small surface area, by shifting the focus to input-triggered expiry.

The use cases described above are primarily illustrative, but they seem interesting enough to warrant further research. With respect to CLRTs specifically, there are many potential constructions, but the construction described above was an honest surprise. `CSFS` + `TEMPLATEHASH` + `nSequence` bit 21 + a vector-commit opcode is a fascinating combination.

I'm interested to hear the community's thoughts on this idea. Is transaction expiry worth revisiting if it can be implemented with strong no-free-relay guarantees?

-------------------------

josh | 2026-06-29 02:19:27 UTC | #2

## A quick follow up

"Input expiry" may be the best description of this primitive:

* `nLockTime` specifies the height the transaction can be mined.
* `nSequence` specifies how many blocks earlier the input expires.

Framing the primitive this way makes it much more intuitive.

It also makes it clear that it is orthogonal to pure transaction expiry. It enables similar applications, but the mechanism is clearly different.

-------------------------

ajtowns | 2026-06-29 03:47:18 UTC | #3

I think the difference here is that normal "expiry" proposals say "this transaction is invalid after X happens", whereas this approach says "this spending path is invalid if this transaction was confirmed after X".

I think it's worth comparing this to a generic ability to introspect the confirmation height of the coin being spent; I proposed that in the [OP_TX thread on bitcoindev](https://gnusha.org/pi/bitcoindev/aOOq-Pw0ht_R0OAK@erisian.com.au/) 8 months or so ago.

Introspecting commit height gives you roughly the same functionality in a straightforward way, instead of (nSequence R + bit21, nLockTime H) write the script (`<TXSEL_HEIGHT> OP_TX <H-100> LESSTHANOREQUAL VERIFY`).

You can somewhat do parent height introspection with this feature, in that `100 CSV 900100 CLTV` will verify that the parent was mined prior to block 900000, if nSequence bit 21 is also set.

I don't see a strong argument for why the consensus-enforced minimum delay here should be 100 blocks rather than 50, 20, 10, 6 or even 1 -- for coinbase outputs that makes some sense; coinbases pay directly to miners who are ultimately responsible for whether we see extensive reorgs or not, but for individual transactions, I don't think this creates a significantly bigger risk in regards to reorgs than regular double-spends.

-------------------------

cmp_ancp | 2026-06-29 11:54:45 UTC | #4

An use case i find interesting is to cut steps on contracts and swaps.

If a contract uses HTLCs and end up with a privkey handover, then the other party already have access to the musig, however, today we need a final tx in order to be certain that no HTLC will be published onchain in the future. With HTLC expiration, we can make swaps 1 tx long, something that can be useful in a high fee environment.

-------------------------

instagibbs | 2026-06-29 17:08:21 UTC | #6

@ademan

[quote="josh, post:5, topic:2667"]
`CLTV` enforces a minimum `nLockTime`, so I don't believe this works. We need to enforce a maximum `nLockTime`. This is probably best accomplished with `nLockTime` introspection (ex: `OP_LOCKTIME`) and `OP_LESSTHAN`.

[/quote]

I believe he meant in conjunction with `OP_TX` and math on the stack

-------------------------

ajtowns | 2026-06-29 17:12:49 UTC | #8

No, I just meant if this proposal were implemented, it would allow partial height introspection (in that you could end up with 900,000 on the stack when spending a coin that was confirmed at height 900k or earlier).

-------------------------

instagibbs | 2026-06-29 18:11:18 UTC | #10

[quote="josh, post:7, topic:2667"]
In [this post](https://gnusha.org/pi/bitcoindev/aOOq-Pw0ht_R0OAK@erisian.com.au/?__goaway_challenge=meta-refresh&__goaway_id=9f2bb273b5b3e13e94f6a563ef49b7a1), @ajtowns suggested that coin-height introspection could be used to solve the delay problem in LN-Symmetry. Below is what may be the absolute cleanest possible construction:

[/quote]

The double-delay problem is when Alice, say, doesn't have the latest update tx fully signed (she sent her psig, waiting to hear back), and she needs to go on-chain. After U-1 blocks Bob then sends the final update tx, which in classical relative timelock ln-symmetry resets the timeout, resulting in a wait of `U*2` before settlement can occur. Does your proposed change allow Alice to immediately settle or otherwise mitigate? Am I thinking on the wrong layer?

(I am genuinely very bad at reasoning about timelock semantics in bitcoin validation so bear with me)

edit: good thread linked on the CSV->CLTV transformation, think I missed that one

-------------------------

instagibbs | 2026-06-29 20:04:05 UTC | #12


[quote="josh, post:11, topic:2667"]
There are two Symmetry constructions that I find interesting, which require transaction expiry:

[/quote]

Ok I was referring to your now-deleted idea, which makes me think it was invalid.

I still have to mull over the other options you state here.

-------------------------

josh | 2026-07-12 02:25:58 UTC | #14

[quote="ajtowns, post:3, topic:2667"]
I think the difference here is that normal “expiry” proposals say “this transaction is invalid after X happens”, whereas this approach says “this spending path is invalid if this transaction was confirmed after X”.
[/quote]

I might suggest an alternative framing. The difference here is that this approach establishes an invariant on `nLockTime`, such that **all timelocks expire by `nLockTime`** if every BIP68 input is height-based and enforces bit 21.

This implies what you are suggesting. The transaction becomes invalid if the input transaction was not confirmed before some height.

[quote="ajtowns, post:3, topic:2667"]
I think it’s worth comparing this to a generic ability to introspect the confirmation height of the coin being spent; I proposed that in the [OP_TX thread on bitcoindev](https://gnusha.org/pi/bitcoindev/aOOq-Pw0ht_R0OAK@erisian.com.au/) 8 months or so ago.
[/quote]

I agree, and thank you for sharing. I was not previously aware of your post.

As you allude to, script-based coin-height introspection is a legitimate alternative, but I would argue that this approach would be like using a hammer to push down a pin. Two drawbacks come straight to mind:

1. **No scriptless expiry:** With the proposed approach, a scriptless spend can enforce input-triggered expiry. This is useful in presigned transactions, key-path spends, and (potential) bare output scripts enforcing a `TEMPLATEHASH` or `CTV` commitment.
2. **More powerful than needed:** The fundamental capability we care about is an absolute cap on an input's confirmation height. Coin-height introspection enables other behavior, like a minimum or exact confirmation height, which may prove unwanted.

Other less pressing but legitimate reasons to prefer the described approach include:

3. **Transaction readability:** With bit 21, all time-related information remains visible in the txid-commitment. This is useful when displaying transactions to users to sign, creating PSBTs, or writing relay policies that require knowledge of expiry. Script-based enforcement would lack these properties by default.
4. **Separation of concerns:** With the proposed approach, the existing separation of concerns between context validation and script validation would remain unchanged. Adding coin-height introspection to script would violate the existing separation of concerns, introduce new code paths, and demand a change in downstream projects like the kernel.

[quote="ajtowns, post:3, topic:2667"]
You can somewhat do parent height introspection with this feature, in that `100 CSV 900100 CLTV` will verify that the parent was mined prior to block 900000, if nSequence bit 21 is also set.
[/quote]

I am uncertain if the script you are suggesting works as you describe. `CLTV` enforces a minimum `nLockTime`, and the true `nLockTime` could be anywhere between that value and the current height. To get the behavior you desire, I think you would need a `NONE|ACP` presignature which commits to the `nSequence` and `nLockTime` values.

For this reason, if script-based max-coin-height enforcement is a goal, I would probably suggest an explicit `OP_SEQUENCE` and `OP_LOCKTIME` introspection capability. You could then enforce bit 21 in script and set an explicit maximum value on `nLockTime` using `LESSTHAN`.

[quote="ajtowns, post:3, topic:2667"]
I don’t see a strong argument for why the consensus-enforced minimum delay here should be 100 blocks rather than 50, 20, 10, 6 or even 1 – for coinbase outputs that makes some sense; coinbases pay directly to miners who are ultimately responsible for whether we see extensive reorgs or not, but for individual transactions, I don’t think this creates a significantly bigger risk in regards to reorgs than regular double-spends.
[/quote]

I think you may be right. I felt it best to be conservative and stick with the same 100-block delay as the `OP_EXPIRE` proposal, but the more I think about it the more I come to the conclusion that the delay is unnecessary.

I plan to write a dedicated follow-up on this issue.

[quote="cmp_ancp, post:4, topic:2667, full:true"]
An use case i find interesting is to cut steps on contracts and swaps.

If a contract uses HTLCs and end up with a privkey handover, then the other party already have access to the musig, however, today we need a final tx in order to be certain that no HTLC will be published onchain in the future. With HTLC expiration, we can make swaps 1 tx long, something that can be useful in a high fee environment.
[/quote]

Short-duration HTLCs is definitely an interesting primitive, and it is good motivation for a shorter delay than the current 100-block minimum. See above.

-------------------------

