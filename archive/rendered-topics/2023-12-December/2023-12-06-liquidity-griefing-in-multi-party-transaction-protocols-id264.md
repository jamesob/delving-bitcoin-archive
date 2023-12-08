# Liquidity griefing in multi-party transaction protocols

morehouse | 2023-12-07 19:12:18 UTC | #1

Multi-party transaction protocols (e.g., dual funding, splicing) are generally vulnerable to liquidity griefing attacks that lock up the victim's UTXOs for an extended period of time.  Some forms of these attacks are difficult to prevent and can require the victim to pay a high ransom (inflated transaction fee) to unlock their funds.

What follows is my best summary and commentary on the various liquidity griefing vectors and potential defenses proposed by `@t-bast`, `@rustyrussell`, `@niftynei`, @roasbeef, @MattCorallo, `@ariard`, myself, and others.  The goal is to spark some discussion about all the potential forms of these attacks (including ones I've missed), potential defenses and tradeoffs.

# Tl;dr

| Griefing Method | Initiator signs first | Delay UTXO locking until broadcast | Require confirmed inputs |  Mempool monitoring | Require p2wpkh inputs | Anchor outputs | v3 transactions |
|-----------------------|-------------------------|-------------------------------------------------|-----------------------------------|-----------------------------------|-----------------------------|----------------------|----------------------|
| Withholding signatures | partial fix | fix | | | | | |
| Nonexistant inputs | | fix | fix | partial fix | | | |
| Low feerate ancestors | | | fix | partial fix | | | |
| Replacing ancestors | | | fix | partial fix | | | |
| Pinned conflicting tx | | | | | | | |
| Witness inflation | partial fix | | | | fix | partial fix | partial fix |

# Liquidity griefing vectors

There are two main vectors for liquidity griefing:
1. preventing successful broadcast of the joint transaction
2. preventing or delaying confirmation of the joint transaction

## Preventing transaction broadcast

For single-party transactions, wallets typically lock UTXOs while constructing the transaction to ensure that concurrent sessions do not double spend UTXOs.  For multi-party transactions, such a policy is vulnerable to liquidity griefing from an attacker who can prevent the transaction from being successfully broadcast.

### Withholding signatures

Once Bob shares signatures for his transaction inputs with Mallory, he can no longer safely back out of the protocol and forget about the joint transaction -- he must be prepared for Mallory to broadcast the joint transaction at any point in the future.  If Mallory never broadcasts the transaction and never shares her input signatures with Bob, Bob's UTXOs will remain locked until he pays transaction fees to double spend one of his inputs.

### Nonexistant inputs

Mallory can use inputs for the joint transaction that do not exist on-chain or in mempools, thus preventing the joint transaction from being successfully broadcast.  Bob doesn't know whether Mallory's inputs will eventually exist on-chain, so he cannot safely forget about the transaction.  So Bob's UTXOs will remain locked until he pays transaction fees to double spend one of his inputs.

## Preventing transaction confirmation

Even if a participant in a multi-party protocol chooses not to lock UTXOs during transaction construction, they generally cannot afford to leave UTXOs unlocked until transaction confirmation because that would cause lots of accidental double spends. The participant will usually need to lock UTXOs soon after the joint transaction is successfully broadcast.  As a consequence, the participant is vulnerable to liquidity griefing from an attacker who can prevent or delay the joint transaction from confirming after broadcast.

### Low feerate ancestors

If Mallory uses unconfirmed inputs in the joint transaction, she can delay confirmation of the joint transaction at least until her input transactions confirm.  By making the feerate on those input transactions very low, Mallory can lock Bob's liquidity for an extended period.

### Replacing ancestors

After Bob has successfully broadcast the joint transaction, Mallory can double-spend one of her unconfirmed input transactions, permanently preventing the joint transaction from confirming.  If Bob fails to detect this, his UTXOs will remain locked indefinitely.

### Pinned conflicting transaction

After sending her signatures to Bob, Mallory can pin a low-feerate transaction that conflicts with the joint transaction in mempools.  The pinning can be done by disabling RBF for the transaction or by attaching many low-feerate descendants.  If she broadcasts to the right nodes and has the good timing, Mallory can partition the network such that Bob's mempool contains the joint transaction while the conflicting transaction is pinned in most other mempools.  The joint transaction will not propagate to miners, and Bob's liquidity will be locked up until Mallory's conflicting transaction confirms.

### Witness inflation

Mallory can inflate the size of her witnesses to lower the effective feerate on the joint transaction and delay confirmation.  Mallory may also attach many low-feerate descendants to the joint transaction in order to prevent Bob from fee bumping via CPFP or double-spending via RBF.


# Potential Defenses

There are several proposed defenses against the above griefing vectors, though none are magic bullets and all have tradeoffs.

## Initiator signs first

*Partially fixes: withholding signatures, witness inflation*

In some multi-party protocols (e.g., dual funding), the non-initiating party is more exposed to repeated griefing than the initiating party.  For these protocols, a policy requiring the initiator to send their input signatures first can help prevent griefing against the non-initiator.

If Mallory is the initiator, this policy trivially prevents her from griefing Bob by withholding signatures (if she does, Bob can safely back out of the protocol).

Additionally, Bob is able to inspect Mallory's witnesses prior to broadcast and ensure they are not inflated.  Bob can also broadcast the joint transaction widely before sending Mallory his signatures, making it difficult for Mallory to construct and broadcast a competing inflated transaction successfully.

### Downsides

Batching multiple channel opens and/or splices into the same transaction becomes impossible for the initiator.

Does not prevent griefing against the initiator.

## Delay UTXO locking until broadcast

*Fixes: withholding signatures, nonexistant inputs*

If Bob keeps his UTXOs unlocked until he is able to successfully broadcast the joint transaction, it is impossible to lock up his liquidity by preventing transaction broadcast.  Bob will simply use the same UTXOs for another purpose once he has a productive opportunity to do so, negating the goal of the attack.

### Downsides

Accidental double spending against honest counterparties can occur.  Concurrent multi-party sessions could end up spending the same UTXOs, causing one of the sessions to end in failure.

When concurrent sessions are expected to be rare and conclude quickly, this approach can work well in practice.  This approach is currently used for dual funding and splicing.

## Require confirmed inputs

*Fixes: nonexistant inputs, low feerate ancestors, replacing ancestors*

By rejecting unconfirmed inputs, Bob can trivally prevent any attack vector that relies on unconfirmed inputs.

### Downsides

Transactions in the mempool must now confirm before they can be used in the joint transaction. This causes a minimum delay of 1 block (~10 minutes) before the multi-party protocol can begin.

## Mempool monitoring

*Partially fixes: nonexistant inputs, low feerate ancestors, replacing ancestors*

If Bob wants to allow unconfirmed inputs, he can inspect all unconfirmed ancestors in his mempool prior to sending his input signatures.  If any ancestors are not present or have a low feerate, Bob can safely cancel the negotiation.

Once he broadcasts the joint transaction, Bob can watch the blockchain for transactions that conflict with any unconfirmed ancestor.  If such a transaction is found, Bob can safely unlock all of his input UTXOs and use them for other purposes.

### Downsides

Bob's mempool doesn't necessarily match other mempools on the network, so it is possible that Mallory has pinned a low-feerate conflicting ancestor in other mempools.  In that case mempool monitoring doesn't help Bob, and Mallory can still lock up his liquidity until the conflicting ancestor confirms.

Mempool monitoring also requires access to a full node's mempool, which may not be possible for parties with resource constraints (e.g., mobile devices).

## Require p2wpkh inputs

*Fixes: witness inflation*

If Bob requires all inputs to be p2wpkh, there is no way for Mallory to inflate her witnesses for those inputs.

#### What about p2tr?
There was also some discussion at the last Lightning spec meeting about p2tr being a viable input type if a proof is provided that only key-path spends are possible.  Unfortunately, I think key-path spends can still have their witnesses inflated [via the annex](https://github.com/bitcoin/bips/blob/e918b50731397872ad2922a1b08a5a4cd1d6d546/bip-0341.mediawiki#script-validation-rules).

### Downsides

Restricting inputs to p2wpkh significantly reduces usability of the multi-party protocol.  For example, this restriction would make it impossible to have a single transaction splice funds from one lightning channel to another.

## Anchor outputs

*Partially fixes: witness inflation*

If the protocol requires all but 1 output per party to be encumbered by a CSV of at least 1, [CPFP carve out](https://bitcoinops.org/en/topics/cpfp-carve-out/) can be used to confirm the joint transaction when needed, regardless of pinning attempts.

### Downsides

CPFP carve-out only works for 2-party protocols.  Therefore, batching multiple channel opens and/or splices into the same transaction becomes impossible in most cases.

Bob also must pay extra fees to perform the CPFP.  If Mallory repeatedly engages in witness inflation attacks against Bob, she can force him to slowly burn his funds as transaction fees.

In addition, encumbering outputs with CSV 1 reduces on-chain efficiency and requires extra complexity to verify the encumberance.

## v3 transactions

*Partially fixes: witness inflation*

[v3 transactions](https://github.com/bitcoin/bitcoin/pull/25038) severely limit the number and size of descendants they can have.  Thus if joint transactions are marked v3, pinning doesn't work, and either party can unlock their funds by double spending one of their inputs.

### Downsides

Even with v3 transactions, double spending can incur a limited RBF penalty fee.  If Mallory engages in repeated witness inflation attacks against Bob, she can force him to consistently pay higher-than-normal fees to double-spend his griefed UTXOs.

And, of course, v3 transactions don't exist yet.

# Conclusion

We have potential defenses against some liquidity griefing vectors, but they all have tradeoffs.  It seems that all known defenses against witness inflation have substantial downsides, and we have no known defenses  against a pinned conflicting transaction that is missing from the victim's mempool.

Hopefully we can come up with some more ideas to address this fundamental problem in multi-party transaction protocols.

-------------------------

t-bast | 2023-12-07 11:21:18 UTC | #2

Thanks for putting this together!

I have a few comments on some of the sections.

## Withholding signatures

> So Bob’s UTXOs will remain locked until he pays transaction fees to double spend one of his inputs.

There is one important subtlety that you don't mention here: the non-initiator will only contribute some funds if the initiator is paying them (via liquidity ads). In the liquidity ads protocol, there is a mechanism to ensure that it is the buyer who pays for the on-chain fees of the seller's inputs and outputs.

Let's suppose that Mallory buys liquidity from Bob and then withholds signatures (or does something else to prevent confirmation). When Alice wants to buy liquidity from Bob, it will be Alice who pays for double-spending Bob's inputs from that previous transaction.

The way I'm currently implementing this for eclair is that node operators will use a dedicated bitcoin wallet for the funds they want to lease. This wallet will be used exclusively for liquidity ads, and only the funds in that wallet will be exposed to such griefing issues. Any double-spend on that wallet will be made by other liquidity ads transactions, which ensures the seller doesn't pay those on-chain fees.

## Preventing transaction confirmation

> they generally cannot afford to leave UTXOs unlocked until transaction confirmation because that would cause lots of accidental double spends. The participant will usually need to lock UTXOs soon after the joint transaction is successfully broadcast.

I agree that this should be the default behavior, to avoid double-spending honest attempts that just take a bit of time to confirm. But that doesn't prevent the non-initiator from unlocking utxos after broadcast, if they detect something weird. If the transaction looks like it has the right feerate but is still not confirming after some blocks, something is fishy and the utxos should probably be unlocked.

That's not a very satisfying solution, but it's something ¯\\__(ツ)_/¯

> Replacing ancestors

This one is really hard to detect (you can never be entirely sure that this is what is happening), but if you unlock utxos the double-spend will then be free, since the original transaction is gone anyway.

## Require confirmed inputs

> Downsides

Another downside is that this is very inefficient from a liquidity perspective. Every unconfirmed change output is effectively "locked liquidity" until it confirms. Ideally we'd really like to be able to use them.

> Hopefully we can come up with some more ideas to address this fundamental problem in multi-party transaction protocols.

I think we've explored mitigations a lot, and I don't think we'll find new solutions at the lightning layer that wouldn't have downsides.

The only real way to fix this is at the mempool layer: fortunately package relay, v3 transactions, ephemeral anchors and cluster mempool are seeing a lot of active research! Sure, it will still take years before that is widely available, but I'm confident that it will eventually be there.

Until then, there is additional risk when selling your liquidity: it's hard to find the right balance between what should be put into the protocol to offer mitigations, and what should be in implementations' heuristics/policies, but IMHO the current state of the proposals strikes a good balance between those.

-------------------------

instagibbs | 2023-12-07 14:35:49 UTC | #3

[quote="morehouse, post:1, topic:264"]
There was also some discussion at the last Lightning spec meeting about p2tr being a viable input type if a proof is provided that only key-path spends are possible. Unfortunately, I think key-path spends can still have their witnesses inflated [via the annex ](https://github.com/bitcoin/bips/blob/e918b50731397872ad2922a1b08a5a4cd1d6d546/bip-0341.mediawiki#script-validation-rules)
[/quote]

There is no annex relay that is considered standard, by any implementation of bitcoin I know of, for exactly this reason. If it becomes standard we need to ensure it cannot be used for witness inflation.

[quote="t-bast, post:2, topic:264"]
fortunately package relay, v3 transactions, ephemeral anchors and cluster mempool are seeing a lot of active research!
[/quote]

For the first iteration it's going to target the commitment transaction style use-case exclusively. I can see two futures:

1) v3 sees a big uptake. We can't think of anything better than ah-hoc rules with bits, and we add another bit that specifically helps in scenarios such as dual funding/splicing/hltlcs
2) Post-cluster mempool we figure out a more general opt-in method, and "relax" v3 into this. Could relax tx topology, make it safe for usage in dual-funding/splicing, and previous users of the bit continue on as usual.

-------------------------

morehouse | 2023-12-07 19:12:45 UTC | #4


[quote="morehouse, post:1, topic:264"]
## v3 transactions
...
### Downsides

Even with v3 transactions, double spending can incur a limited RBF penalty fee. If Mallory engages in repeated witness inflation attacks against Bob, she can force him to consistently pay higher-than-normal fees to double-spend his griefed UTXOs.
[/quote]

From a [GitHub discusssion](https://github.com/lightning/bolts/pull/851#issuecomment-1845650132) about liquidity griefing against v3 transactions:

[quote="morehouse"]
I did a little math to see what a typical griefing might cost an LSP.

Some recent mempool feerates:

- Minimum relay: 10 sat/vB
- 1 week confirmation: 20 sat/vB
- 6 hour confirmation: 80 sat/vB

Let's say we use 80 sat/vB for a funding transaction and the LSP's portion of the transaction is 250 vB, so they allocate `80 * 250 = 20,000 sat` for the transaction fee.  Let's say the total transaction is 500 vB, so the total fee for the transaction is 40,000 sat.

The attacker inflates their witnesses to reduce the feerate to 10 sat/vB and also chains a 1,000 vB descendant transaction (the max allowed by v3 policy) with a fee of 10,000 sat (10 sat/vB).  This package is unlikely to confirm for a very long time.

If the LSP wishes to double-spend their griefed UTXOs, they must now spend `40,000 + 10,000 = 50,000 sat`.  If they use another 500 vB dual-funded transaction to double spend, the required feerate for that transaction would be `50,000 sat / 500 vB = 100 sat/vB`  This is 25% more than the expected current feerate.

Suppose further that the attacker is doing repeated griefing and this new dual-funded transaction is actually with another node controlled by the attacker.  The attacker can grief this transaction as well, causing the double-spend fee to increase to `50,000 + 10,000 = 60,000 sat` (120 sat/vB).  And so on.
[/quote]

[quote="t-bast"]
Nice work! But this pinning attack does not reliably work though, so I believe it's not the end of the world. To be able to effectively lower the feerate of the funding transaction, the attacker needs to have low-feerate ancestors. But since those ancestors are low-feerate, they will be dropped from a fraction of the network's mempools, which will allow an honest transaction at the normal feerate to go through (because in those mempools, that honest transaction isn't a double-spend, the conflicting transactions have been evicted). That's what [@TheBlueMatt](https://github.com/TheBlueMatt) mentioned a few times, that in practice today, based on his experiments, even low-ish feerate transactions are still able to propagate and confirm.
[/quote]

-------------------------

morehouse | 2023-12-07 19:14:57 UTC | #5

[quote="t-bast, post:4, topic:264"]
Nice work! But this pinning attack does not reliably work though, so I believe it’s not the end of the world. To be able to effectively lower the feerate of the funding transaction, the attacker needs to have low-feerate ancestors. But since those ancestors are low-feerate, they will be dropped from a fraction of the network’s mempools, which will allow an honest transaction at the normal feerate to go through (because in those mempools, that honest transaction isn’t a double-spend, the conflicting transactions have been evicted). That’s what @TheBlueMatt mentioned a few times, that in practice today, based on his experiments, even low-ish feerate transactions are still able to propagate and confirm.
[/quote]

Witness inflation is done on the joint transaction itself.  Unconfirmed ancestors are not needed for this attack vector.

v3 does prevent pinning, but witness inflation can still be used with a single 1000 vB descendant to inflate the RBF cost for the victim.  Doing this repeatedly can burn the victim's funds to fees, with no cost to the attacker.  Whether this can be done reliably, I'm not sure -- the only way to find out is to try such an attack.

-------------------------

t-bast | 2023-12-08 08:11:22 UTC | #6

> Witness inflation is done on the joint transaction itself. Unconfirmed ancestors are not needed for this attack vector.

It has the same effect though: the inflated transaction is thus paying a lower feerate than expected, which makes it easy to replace with a transaction that pays the expected feerate (when not coupled with descendant pinning).

It is also hard to reliably pull out: if you also pin with descendants, that means the descendants will be even lower feerate and will thus be evicted from some nodes' mempool, allowing a replacement to propagate. It's impossible to quantify that though, since it really depends on the topology of the whole network's mempools...

-------------------------

instagibbs | 2023-12-08 15:11:17 UTC | #7

[quote="morehouse, post:4, topic:264"]
This is 25% more than the expected current feerate.
[/quote]

Honestly, that doesn't seem very bad, considering the variability of "going feerates" and the practical realities of block confirmations. 

[quote="t-bast, post:6, topic:264"]
It has the same effect though: the inflated transaction is thus paying a lower feerate than expected, which makes it easy to replace with a transaction that pays the expected feerate (when not coupled with descendant pinning).
[/quote]

anti-DoS rule#3 is the pain point here. If it gets evicted from the mempool, then you're in the clear(future bytes will have to pay higher minfee, protecting the node), otherwise you have to overcome the total value.

[quote="morehouse, post:5, topic:264"]
v3 does prevent pinning, but witness inflation can still be used with a single 1000 vB descendant to inflate the RBF cost for the victim. Doing this repeatedly can burn the victim’s funds to fees, with no cost to the attacker. Whether this can be done reliably, I’m not sure – the only way to find out is to try such an attack.
[/quote]

The future post-cluster mempool scheme I have in mind is pretty simple: v3 would be relaxed to "must be in top block to enter mempool". This makes pinning essentially impossible unless the attacker somehow knows the top-rate backlog is going to get bad shortly in the future, and should protect against witness inflation and the like. Would allow for pin-resistant batch CPFP, would mean you don't necessarily need no unconfirmed ancestors, should make ANYONECANPAY usage more safe(like HTLC second stage txns) etc.

Just a sketch at this point, but something to consider if you're already thinking about pins and mitigations.

-------------------------

