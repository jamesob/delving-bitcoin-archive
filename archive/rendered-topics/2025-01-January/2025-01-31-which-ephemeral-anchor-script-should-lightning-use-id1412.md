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

instagibbs | 2025-01-31 15:58:28 UTC | #2

Note that you aren't stuck with one format. You could reasonably consider p2a when the anchor is 0-value, and keyed otherwise.

re:miner "stealing" the funds, I'm not sure that's bad at all? The main new weirdness is counterparty may be tempted to ramp up the trimmed amount and take it themselves, even though in the end miners will probably do well to snipe that entire value.

Alternatively, as soon as the output gets large enough to not be considered dust, you can go right back to inserting the remaining trimmed value into the commitment transaction fee, since it no longer has to be 0-fee.

-------------------------

ariard | 2025-01-31 18:00:52 UTC | #3

I strongly recommend not to do the 1st option: unkeyed anchors to avoid txn hijack by miners:
https://groups.google.com/g/bitcoindev/c/ZspZzO4sBys
(see the section 6.4 “transaction traffic hijack and policy exposure surface”)

Option 2 sounds reasonable: this let each counterparty deal with its own commitment tx + cpfp, without letting the other one tampering with it, rather than a pure replacement at the root of the utxo. Though other keyed options sound okay-ish too.

For keeping fee-bumping delegation, I don’t see why pre-signed txn with `ANYONECANPAY` cannot be used between the delegate and the delegatee (in that model, you exclude that the delegate pin or do other things).

-------------------------

harding | 2025-02-04 14:11:45 UTC | #4

Notwithstanding @instagibbs's comments about a dynamic approach, option 1 (P2A) seems much safer to me:

- _Theft of trimmed HTLC value:_ options 2, 3, and 4 allow a counterparty to receive trimmed HTLC value if their force close is uncontested while it is in the mempool.  Average time in the mempool is easily reduced to 10 minutes by paying a next-block fee, and it may be possible for your counterparty to make good guesses about when your node will be offline for ~10 minutes.

  For example, every Saturday night, Bob automatically takes his node offline for 30 minutes to perform a full filesystem backup and software update.  His counterparty, Mallory, learns about his predictable downtime from her logs and causes the maximum amount of trimmed HTLC value to be forwarded through his node to her node shortly before his next downtime.  She then force closes the channel with the keyed anchor paying for next-block confirmation, which at that time is less than its full value, allowing her to profit.

- _Reduction in reorg safety:_ a counterparty that works with a miner can pay the full value of a keyed anchor to their wallet, leaving both the commitment transaction and the CPFP transaction as zero fee in a block.  If there's a natural reorg, no rational miner will include those zero-fee transactions in their reorg blocks, reopening a channel that previously seemed closed.  If the reorg is long enough, it may become possible to steal value from regular HTLCs in the reopened channel.

  Long reorgs create all sorts of other problems, so I'm not claiming that this is a strong criticism, but it does seem better to me that out-of-band fee payment can't reduce the chance that a commitment transaction will re-mined in a reorg.

-------------------------

t-bast | 2025-02-04 14:15:28 UTC | #5

[quote="instagibbs, post:2, topic:1412"]
Note that you aren’t stuck with one format. You could reasonably consider p2a when the anchor is 0-value, and keyed otherwise.
[/quote]

Good point, we should definitely do something like this! When the anchor output is 0, there's no value to steal so I think P2A makes a lot of sense since it's the cheapest way to bump fees. @ariard I don't think the issue you're describing applies in that case, does it?

So we really just need to choose which option we use when the anchor amount isn't 0. Let's keep debating that case!

[quote="instagibbs, post:2, topic:1412"]
Alternatively, as soon as the output gets large enough to not be considered dust, you can go right back to inserting the remaining trimmed value into the commitment transaction fee, since it no longer has to be 0-fee.
[/quote]

I'm not a fan of this option (but we should still keep it on the table if others think this is desirable). I'd rather always keep the commitment 0-fee, so that all of the channel's value is split among existing outputs and nodes don't have to lose all of the dust HTLCs to mining fees in case of a force-close. It seems to be an improvement to materialize these dust HTLCs so that they may be only partially allocated to mining fees (especially in option 2, where your peer could overbid you but would need to get their commitment confirmed in that case, which means they'll need to wait for the `to_self_delay`).

-------------------------

t-bast | 2025-02-04 14:24:16 UTC | #6

[quote="harding, post:4, topic:1412"]
*Reduction in reorg safety:* a counterparty that works with a miner can pay the full value of a keyed anchor to their wallet, leaving both the commitment transaction and the CPFP transaction as zero fee in a block.
[/quote]

I hadn't considered this case, thanks! I think this is fine though, because lightning nodes will only consider transactions irrevocably confirmed after a long enough delay to protect against reorgs. In this case, the honest node will keep rebroadcasting his anchor-spend (and the associated commitment), which honest miners should pick up?

HTLC delays should always be large enough to ensure that reorgs aren't an issue (e.g. `cltv_expiry_delta = 144 blocks`) so this shouldn't put funds at risk?

[quote="harding, post:4, topic:1412"]
it may be possible for your counterparty to make good guesses about when your node will be offline for ~10 minutes.
[/quote]

While this may apply to small nodes (who probably aren't a meaningful target since they're small), I don't think this applies to serious nodes (our node never has a 10 minutes downtime), so I think it could still be valuable to have the option to claim back some of the dust HTLCs instead of having them go entirely to fees? Even if you're not able to claim it back 100% of the time, it's still valuable if you're able to claim it back sometimes?

-------------------------

harding | 2025-02-04 15:12:03 UTC | #7

[quote="t-bast, post:6, topic:1412"]
HTLC delays should always be large enough to ensure that reorgs aren’t an issue (e.g. `cltv_expiry_delta = 144 blocks`) so this shouldn’t put funds at risk?
[/quote]

Sure, but if you think all of your channels are closed, you may take your node offline after (say) just 6 confirmations.  Again, I'm not claiming this is a strong criticism; it just seems more conservative to prevent your counterparty from being able to use money you paid to create transactions that are less likely to persist through a reorg.

[quote="t-bast, post:6, topic:1412"]
While this may apply to small nodes (who probably aren’t a meaningful target since they’re small), I don’t think this applies to serious nodes (our node never has a 10 minutes downtime)
[/quote]

I don't like safety tradeoffs that optimize for "serious nodes".  I'd prefer a network that's reasonably safe even for non-experts.

In addition, the risk is probabilistic.  If an attacker can guess when your node will be offline for 1 minute, they have a ~10% chance of being able to get their transaction mined uncontested.  The cost is the onchain fee they paid to open the channel (if any) and the cost to unilaterally close.  Opening a single-funded channel at 1 s/vb costs ~150 sats and closing with just a keyed anchor is (I'm guessing) ~300 sats at 1 s/vb; so with Eclair's default dust tolerance of 50,000 sat, the attack reaches breakeven profitability if the attacker can predict when your node will be offline for at least 6 seconds (`600 seconds / (50_000 sats dust tolerance / 450 sats costs)`).

Six seconds is probably below propagation time to your node and back to miners if your counterparty is able to initially relay the transaction directly to large miners, meaning a clever attacker doesn't even need to wait for your node to go offline.

-------------------------

morehouse | 2025-02-05 22:13:12 UTC | #8

A major benefit of TRUC channels is that nodes don't have to monitor mempools anymore, since they can rely on their commitment package replacing the counterparty's once they bump to a high enough fee rate.  This simplifies a lot of code and is especially good for mobile.

I think we should lean into this simplification and therefore dismiss Options 3 and 4 entirely.  Their main benefit is the ability to CPFP the remote commitment, which many/most nodes won't implement anyway for TRUC channels.

In deciding between the remaining options, there's at least 2 issues to consider:

- [fee griefing](https://petertodd.org/2023/v3-txs-pinning-vulnerability)
- dust theft
-----------------------

**Tl;dr:** We can avoid dust theft and fee griefing as follows:
- Do Option 2 with the same anchor script we already use.
- Make @instagibbs [suggested changes](https://delvingbitcoin.org/t/which-ephemeral-anchor-script-should-lightning-use/1412/2?u=morehouse) to dust HTLC handling.

--------------------------

## Option 1: unkeyed anchor

#### Fee griefing

Anyone on the network can broadcast a conflicting anchor spend, potentially evicting the commitment package from the mempool (replacement cycling) or forcing the victim to spend more fees than normal (limited RBF rule #3 pinning).

#### Dust theft

In the medium-long term, unkeyed anchors are safe from dust theft attacks, since miners will claim the full anchor output as fees, preventing the channel counterparty from ever profiting from such an attack.  As a result, the counterparty has no extra motivation to inflate the anchor output with dust HTLCs in the first place, beyond getting the other party to pay for the force close (which exists with current channels already).

## Option 2: single-participant keyed anchor

#### Fee griefing

The channel counterparty can do fee griefing and replacement cycling, but no one else can.

#### Dust theft

The counterparty can steal dust HTLCs, as @t-bast described.

## Alternative option: single-participant keyed anchor without anchor value inflation

This is Option 2, along with @instagibbs idea:

- Dust HTLCs increase the value of the keyed anchor up to the dust limit.
- Once the anchor reaches the dust limit, further dust HTLCs increase the commitment fees.

#### Fee griefing

The channel counterparty can do fee griefing and replacement cycling, but no one else can.

#### Dust theft

The channel counterparty cannot steal dust HTLCs, since the excess always goes to miners as commitment fees.

## Thoughts

Current mitigations against replacement cycling (rebroadcasting and aggressive fee bumping) are enough to make such attacks very expensive for the channel counterparty or anyone else on the network, so I think we shouldn't be too worried about additional exposure due to unkeyed anchors in that regard.

However, unkeyed anchors *do* make it possible for anyone to grief lightning force closes they see in the mempool, forcing each victim to spend up to ~50% more in fees (see Peter Todd's math [here](https://petertodd.org/2023/v3-txs-pinning-vulnerability)).

We can limit our exposure to such griefing by using keyed anchors.  But then to prevent dust HTLC theft we need to add the excess dust HTLCs to commitment fees.  And to prevent UTXO bloat in that case we should probably add an anyone-can-spend path to the non-ephemeral anchor with a CSV delay (like with current anchor channels).

So most of the time we could use a simple P2TR anchor script and the anchor would be considered ephemeral dust, so it would need to be spent immediately.  But once the anchor output accumulates a value of 330 sats, we'd need to switch to the same P2WSH anchor script we currently have for anchor channels:

```
<local_funding_pubkey> OP_CHECKSIG OP_IFDUP
OP_NOTIF
    OP_16 OP_CHECKSEQUENCEVERIFY
OP_ENDIF
```

Or, for simplicity, we could just always use the current anchor script and address the slight P2TR optimization later when we do taproot TRUC channels.

-------------------------

ajtowns | 2025-02-06 06:19:44 UTC | #9

[quote="morehouse, post:8, topic:1412"]
The channel counterparty can do fee griefing and replacement cycling, but no one else can.
[/quote]

Isn't the channel counterparty doing fee griefing more of a concern? If a random person does it, they risk losing a bunch of fees, and get no benefit; if a miner does it, either they mine your original tx (delaying it slightly because other miners didn't see it) and everything's still fine, or they risk losing funds and get no benefit; but if your counterparty does it, they may get to claim a htlc payment back via the timeout path, and actually make a profit.

-------------------------

morehouse | 2025-02-06 18:31:50 UTC | #10

[quote="ajtowns, post:9, topic:1412"]
Isn’t the channel counterparty doing fee griefing more of a concern? If a random person does it, they risk losing a bunch of fees, and get no benefit; if a miner does it, either they mine your original tx (delaying it slightly because other miners didn’t see it) and everything’s still fine, or they risk losing funds and get no benefit; but if your counterparty does it, they may get to claim a htlc payment back via the timeout path, and actually make a profit.
[/quote]

When the channel counterparty does it, yes they might be able to steal HTLCs if the victim isn't willing to fee bump high enough.

A random person indeed can't steal funds via this attack, but they *can* grief the victim.  They don't need to risk much in fees either if they already have a low-priority ~1000vB transaction they want to get mined anyway.  In fact we've already seen a similar kind of [accidental pinning](https://x.com/peterktodd/status/1793380279018803208) in the wild with expired anchor outputs.  Are you sure that *no one* on the network will ever want to grief in this way? 

A mining pool may stand to profit nicely if the victim pays 50% more in fees than they normally would.  The EV depends on:

- the probability of the pinned transaction confirming before the victim fee bumps enough to escape the pin, and
- the percentage of hash rate controlled by the pool.


By using a keyed anchor, we can limit our exposure to such attacks to *just* the channel counterparty, instead of allowing anyone on the network to do it.

-------------------------

instagibbs | 2025-02-06 19:45:31 UTC | #11

[quote="morehouse, post:10, topic:1412"]
Are you sure that *no one* on the network will ever want to grief in this way?
[/quote]

It's a question of probabilities and assumptions, there are no absolutes here. Every time a P2TR keyspend is done in lieu of a P2A spend, that's ~46vB additionally used.

The actual % of pin relies on many factors, the absolute worst case being a very flat, deep mempool, a tiny commitment transaction and tiny honest CPFP.

To reverse the framing, every time you require a signature where none was needed, with small packages you are deterministically "pinning yourself" by ~10% to avoid being pinned by X% due to a completely unrelated third party attacker. As packages get larger, the relative pin in both senses drops, of course.

> A mining pool may stand to profit nicely if the victim pays 50% more in fees than they normally would

I think this is isomorphic with miner censorship with one exception: the fact that there would also have "victim miners" who are essentially being robbed of the "honest" CPFP that would have been on the table initially for the next block. If miners in general think they'll get more out of you by collectively waiting, that cannot be helped and you need to set longer HTLC expiries to make defection more palatable.

All that said, I think this specific choice is a fairly minor decision in the overall security story.

-------------------------

ajtowns | 2025-02-07 03:43:05 UTC | #12

[quote="morehouse, post:10, topic:1412"]
In fact we’ve already seen a similar kind of [accidental pinning](https://x.com/peterktodd/status/1793380279018803208) in the wild with expired anchor outputs.
[/quote]

I don't think that's an interesting example: anchor outputs for txs that have already confirmed are just "here's a small amount of free sats for anyone"; if you're structuring your tx in such a way that you're making moving real funds contingent on also claiming those free sats, that's a bad bug.

[quote="morehouse, post:10, topic:1412"]
Are you sure that *no one* on the network will ever want to grief in this way?
[/quote]

Even with a keyed anchor, your counterparty can grief you in this way, and they are much more likely to do so than a random person on the network, because they have a much greater chance of seeing a profit from doing so (and even if it's just due to buggy software, they're the ones paying attention to the tx prior to it being mined). You can't make this scenario impossible (or even vanishingly unlikely) all you can do is make the expected costs cheaper. If the 1000vb ancestor limit provided by TRUC doesn't reduce those costs enough, that's an argument for providing a way to have [a smaller limit](https://github.com/bitcoin/bips/blob/ea7aae8d1ff275f0d5cff4b3c0efaff66b8fa55f/bip-0431.mediawiki#cite_note-13) (the limit was already reduced from [4000vb to 1000vb](https://github.com/bitcoin/bitcoin/pull/25038#issuecomment-1198946822)).

[quote="morehouse, post:10, topic:1412"]
A mining pool may stand to profit nicely if the victim pays 50% more in fees than they normally would.
[/quote]

If they run the attack, then their win scenarios are:
 * every other miner that wins a block before them sees their tx (at high fee but low feerate) instead of the user's tx (at low fee but high feerate) and doesn't mine it, slightly driving down the revenue of other mining pools
 * people routinely overpay channel close txs, driving up average close fees, of which they get the regular percentage

Their loss scenario is that the mempool clears out a bit too much and a different pool mines their high fee / low feerate tx, meaning they pay extra fees to another miner, and the user gets their tx mined without paying any fees. I don't think that adds up to making a nice profit.

If this griefing approach does somehow become common anyway, it can also be worked around by some nodes/pools adopting a policy of "replace by feerate", at least for txs that would be at the top of the mempool. (And cluster mempool would make measuring "top of the mempool" substantially more efficient/straightforward)

-------------------------

jsarenik | 2025-02-07 07:39:38 UTC | #13

Seeing word "miners" mentioned a lot I would like to shed some light to the background of a recent [mainnet P2A all-fee tx](https://mempool.space/tx/ad1507f186ccdd448cc58a5ed68367249de6ad55d7f4b70bb3fcc5c88a85c626?mode=details).

I kept watching P2A recently but came too late to notice the last-recently added sats and found the transaction on mempool.space in its 3rd replacement that was about to send 330 sats to one particular keyed address I did not make note of and which is no longer in replacements history on mempool.space. That tx also had a child transaction and even combining all the available P2A unspent inputs in an all-fee no-text transaction didn't get me to replace.

So I decided to experiment and prioritise my tx in my mempools of 3 pruned nodes running at home on one IPv4 address, just different ports.

And it worked.

See [my handrafted tx as a commit in git repo](https://github.com/carnhofdaki/anchor/commit/86a4983b4f0947e2858795e90faa3c71d09fa290)

I think miners are totally clean in this. MARA just may happen to use Peter Todd's LibreRelay where replacements need merely 1/4 sats extra compared to Bitcoin Core.

That's my two sats on the topic of P2A.

`in fees we rust`

-------------------------

ariard | 2025-02-08 04:02:33 UTC | #14

> Note that you aren’t stuck with one format. You could reasonably consider p2a when the anchor is 0-value, and keyed otherwise.

As LN pre-signed transactions are always valid, option A should not allow you to break the tx-relay safety of option B, whatever is picked up for option A and option B.

> "re:miner “stealing” the funds, I’m not sure that’s bad at all? The main new weirdness is counterparty may be tempted to ramp up the trimmed amount and take it themselves, even though in the end miners will probably do well to snipe that entire value.”

Max HTLC cltv_expiry_time is 2016, so a LN node operator is trusting that a miner with 1/2016 is not stealing the funds.

> Theft of trimmed HTLC value: options 2, 3, and 4 allow a counterparty to receive trimmed HTLC value if their force close is uncontested while it is in the mempool. Average time in the mempool is easily reduced to 10 minutes by paying a next-block fee, and it may be possible for your counterparty to make good guesses about when your node will be offline for ~10 minutes.”

Mining is an anonymous open set by design, so I think we can never assume that lightning counterparty != miner. So as far as i can understand of the options proposed, "theft of trimmed HTLC value" is a risk we always have.

> "Good point, we should definitely do something like this! When the anchor output is 0, there’s no value to steal so I think P2A makes a lot of sense since it’s the cheapest way to bump fees. @ariard I don’t think the issue you’re describing applies in that case, does it?”

No it doesn't fix it, the jamming attacker can evict a honest parent transaction from the mempool with a RCA, while the CPFP is bringing to-be-captured feerate paying for the surface of said parent.

To be fair, I don't think that something that LN can really fixes at this level, apart of avoiding to increase the set of can-be hijacked txn with unkeyed anchors and making the problem worst.

Package malleability bad.

> "HTLC delays should always be large enough to ensure that reorgs aren’t an issue (e.g. cltv_expiry_delta = 144 blocks) so this shouldn’t put funds at risk?”

No, this is not the cltv delta for a carried HTLC, it's the on-chain reorg-safety delay (rust-lightning’s `ANTI_REORG_DELAY`), after which a HTLC is considered irrevocably resolved.

> I don’t like safety tradeoffs that optimize for “serious nodes”. I’d prefer a network that’s reasonably safe even for non-experts.

I agree here with Dave - Lightning protocol safety tradeoffs are already circumvoluted enough that even serious nodes configured by experts are not reasonably safe.

> In addition, the risk is probabilistic

And even if a node max downtime is 10 min, in bitcoin block issuance is probabilistic, so you might have a spike of blocks in few lapse of time. That is something a counterparty can try to take opportunity off, as your node will have to catchup the downtime (i.e block validation), and unless you over-provision the downtime resumption, that computational overhead won't be amortize before a longer period of blocks. Not a tx propagation issue only.

> Six seconds is probably below propagation time to your node and back to miners if your counterparty is able to initially relay the transaction directly to large miners, meaning a clever attacker doesn’t even need to wait for your node to go offline.

Privacy-enhancing propagation delays are already 2 seconds for outbound links (`OUTBOUND_INVENTORY_BROADCAST_INTERVAL`).

""A major benefit of TRUC channels is that nodes don’t have to monitor mempools anymore, since they can rely on their commitment package replacing the counterparty’s once they bump to a high enough fee rate. This simplifies a lot of code and is especially good for mobile.

No, the benefit to not have to monitor mempools anymore is not coming with TRUC itself per se, which is a mempool-level topological restriction policy only. The benefit of propagating one's commitment transaction and a CPFP, and getting a guarantee this can replace a current spend of the same UTXO is what normally should be achieved with ancestor package relay, atomicity of the replacement try matters. Actually, this is still good to monitor mempool, and a lightning node can see what should be the most effective feerate for a claim to be included, and not some hardcoded bumps at each X block ticks.

> I think we should lean into this simplification and therefore dismiss Options 3 and 4 entirely. Their main benefit is the ability to CPFP the remote commitment, which many/most nodes won’t implement anyway for TRUC channels.

True, with current dual anchors format, last time I checked most lightning implementations, at the exception of Eclair, didn't CPFP correctly the remote commitment.

> Anyone on the network can broadcast a conflicting anchor spend, potentially evicting the commitment package from the mempool (replacement cycling) or forcing the victim to spend more fees than normal (limited RBF rule #3 pinning).

If the 0-fee commitment transaction is under mempool min fee, yes it is evicted from network mempools when they're at their max size.

To enhance the analysis, there should be a dissociation of a fee griefing (the LN counterparty forces to spend more fees than the average) from HTLC stealing (the LN counterparty "double-spend" a routed HTLC).

> In the medium-long term, unkeyed anchors are safe from dust theft attacks, since miners will claim the full anchor output as fees, preventing the channel counterparty from ever profiting from such an attack. As a result, the counterparty has no extra motivation to inflate the anchor output with dust HTLCs in the first place, beyond getting the other party to pay for the force close (which exists with current channels already).

This forgets the edge situation where the counterparty is a low-hashrate miner with a direct motivation to inflate the anchor output with dust HTLCs.

> Current mitigations against replacement cycling (rebroadcasting and aggressive fee bumping) are enough to make such attacks very expensive for the channel counterparty or anyone else on the network, so I think we shouldn’t be too worried about additional exposure due to unkeyed anchors in that regard.

In the world of today where you can exploit the expiration time of mempools txn to slash the cost of a replacement cycling attack, not really:

https://groups.google.com/g/bitcoindev/c/OWxX-o4FffU

> We can limit our exposure to such griefing by using keyed anchors. But then to prevent dust HTLC theft we need to add the excess dust HTLCs to commitment fees. And to prevent UTXO bloat in that case we should probably add an anyone-can-spend path to the non-ephemeral anchor with a CSV delay (like with current anchor channels).

I think it's good idea to do keyed anchors to reduce malleability of the package to the counterparty only, and not the whole world. We can never prevent dust HTLC theft in the edge situation where the counterparty is a miner.

> A random person indeed can’t steal funds via this attack, but they can grief the victim. They don’t need to risk much in fees either if they already have a low-priority ~1000vB transaction they want to get mined anyway. In fact we’ve already seen a similar kind of accidental pinning in the wild with expired anchor outputs. Are you sure that no one on the network will ever want to grief in this way?

A LSP fee griefing another LSP's splice-out transaction cannot be excluded, as there might be a way to capture more off-chain routing fees from doing that.

A LSP fee griefing another LSP's splice-out transaction cannot be excluded, as there might be a way to capture more off-chain routing fees from doing that.

> the probability of the pinned transaction confirming before the victim fee bumps enough to escape the pin, and

> the percentage of hash rate controlled by the pool.

And devil is in the details - the pinning transaction can in theory be used to pin multiple channels in parallel to maximize the EV.

No guarantee that all the victims try to fee-bump at the same time point in the network.

> every other miner that wins a block before them sees their tx (at high fee but low feerate) instead of the user’s tx (at low fee but high feerate) and doesn’t mine it, slightly driving down the revenue of other mining pools
> people routinely overpay channel close txs, driving up average close fees, of which they get the regular percentage
> Their loss scenario is that the mempool clears out a bit too much and a different pool mines their high fee / low feerate tx, meaning they pay extra fees to another miner, and the user gets their tx mined without paying any fees. I don’t think that adds up to making a nice profit.

In the world of today where you can exploit the expiration time of mempools txn to kickout the low fee but high feerate of a user's transaction, that's easy to make profit:

https://groups.google.com/g/bitcoindev/c/OWxX-o4FffU

Though of course coming up with a quantitative analysis of a "nice profit" in real-world mempool flows, that is the question.

## Gathering Thoughts

I believe we have 4 distinct risks (at least) to consider for ephemeral anchors:
- a) miner fee griefing: where a miner seeks to optimize its block template average feerate e.g RCA a commitment's CPFP
- b) counterparty fee griefing: where a counterparty seeks to DoS the package propagation for an external gain
- c) dust theft: miner or counterparty stealing the trimmed HTLC output
- d) counterparty HTLC stealing: where a counterparty seeks to steal a HTLC e.g RCA the 2nd-stage HTLC-timeout

I think one of the main open question is if we assume that a counterparty can be a miner, even before considering keyed vs unkeyed anchor, and what to do with the trimmed HTLC value.

E.g for a dust theft as a counterparty inflating the dust value, and as a miner mining this dust value out-of-band.

If I remember correctly, a HTLC max locktime is 2016 blocks LN-wide, so it’s a bar as low as like a miner with 1/2016 of the hashrate to consider as a potential adversary targeting a LN node (though even a miner with 1/144 of the hashrate, I think it’s a low bar).

-------------------------

