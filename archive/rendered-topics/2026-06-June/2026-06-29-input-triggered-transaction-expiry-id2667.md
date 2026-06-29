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

