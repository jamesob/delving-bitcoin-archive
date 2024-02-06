# OP_VAULT comments

harding | 2024-02-05 23:45:12 UTC | #1

I'm (slowly) reviewing BIP345 `OP_VAULT` and I had some questions/concerns that seemed to be appropriate for here.

Is it problematic that the expected fee cost to create a trigger + withdrawal transaction may be less than the expected fee cost to initiate a recovery?

Example 1: Bob has a vault that's guarded by a watchtower than only looks for confirmed transactions (no mempool monitoring).  Mallory obtains his trigger authorization key and creates a relatively low feerate transaction that steals a relatively small amount of Bob's funds, revaulting the remaining amount.  For Bob to ensure he recovers the funds, he'll have to create a fairly large recovery transaction that pays a relatively high feerate to get confirmed in time.  That may cost him more sats than the amount Mallory stole is worth, so it might be rational for him to write off the lost amount and allow Mallory to claim it.

Can that above attack be extended using revaulting?

Example 2: Bob again has a vault; in this example, his watchtower even scans the mempool.  Mallory obtains his trigger authorization key and creates (but does not broadcast) a zero-fee transaction that steals a relatively small amount of Bob's funds, revaulting the remaining amount.  Then Mallory creates another offchain spend from the revaulted output in another zero-fee transaction that steals a relatively small additional amount of Bob's funds.  Mallory repeats this process until she has roughly a block full of small-value spends from Bob's vault.  Mallory hasn't yet leaked any evidence that she knows Bob's trigger authorization key, so she patiently privately attempts to mine the block full of spends from Bob's vault during low feerate periods (she switches to mining a default Bitcoin Core template during high feerate periods).  When she finds a block of her spends from Bob's vault, she broadcasts it.

Bob's watchtower is now alerted to the theft attempt.  It might cost the watchtower more sats than the theft is worth to send even a batch recovery transaction, causing Bob to write off the lost amount and allowing Mallory to claim it.

Does this have implications for watchtower reserves?

Example 3: Bob again has a vault and Mallory again obtains his trigger authorization key.  This time she uses revaulting to split his total vault value into about 3,000 evenly-sized chunks (3,000 being my estimate of the maximum number of revaultings possible in a single block).  Each of these chunks may be economically viable for Bob's watchtower to recover, but to do so the watchtower's hot wallet must contain enough funds to pay for a spend of 3,000 inputs.  I quickly estimate the total input vsize of an authenticated recovery path at 100 vbytes, so even with maximal batching, Bob's watchtower will need to pay to put 300,000 vbytes onchain in a potentially short amount of time.  At 100 sats/vb, that implies paying 0.3 BTC in fees.  I think there are entirely plausible sets of circumstances that could lead to at least an order of magnitude higher feerates than that.

If a watchtower may need to pay 0.3 BTC (or more), that implies a watchtower needs to keep 0.3 BTC in its hot wallet.  Any watchtower that fails to keep a sufficient amount will not be able to initiate a full recovery, allowing Mallory to successfully steal some of Bob's funds.  Someone who uses multiple fully capable watchtowers for redundancy and who assumes that, in the worst case, only one watchtower will be left standing at the time of an attempted theft will need to keep at least 0.3 BTC in every watchtower (e.g., for x watchtowers, they need a total of 0.3x BTC in watchtower hot wallets).  Funds stored in hot wallets may be at increased risk of theft compared to non-instantaneous spending methods.

I think the second and third concerns above can probably be addressed by suggesting or requiring a relative block delay between respends of the same vault output.  E.g., if the trigger authorization script included a `1 OP_CSV` delay, Mallory could only practically create one uneconomic withdrawal before Bob had a chance to sweep the rest of his funds into recovery.  This would, of course, reduce the flexibility of vaults for frequent spending and make them harder to compose with scripts for contract protocols.

In the absence of a relative block delay, I'm not sure vaults below 0.3 BTC should be considered as providing security beyond the security of the trigger authorization script as an attacker could always make the entire vault value uneconomical to recover (potentially at a profit to the attacker).

-------------------------

ajtowns | 2024-02-06 03:29:59 UTC | #2

[quote="harding, post:1, topic:521"]
Example 1: Bob has a vault that’s guarded by a watchtower than only looks for confirmed transactions (no mempool monitoring). Mallory obtains his trigger authorization key and creates a relatively low feerate transaction that steals a relatively small amount of Bob’s funds, revaulting the remaining amount. For Bob to ensure he recovers the funds, he’ll have to create a fairly large recovery transaction that pays a relatively high feerate to get confirmed in time. That may cost him more sats than the amount Mallory stole is worth, so it might be rational for him to write off the lost amount and allow Mallory to claim it.
[/quote]

I think that's trivially true if you take an extreme enough case: a miner could discover the trigger key, and create a tx spending 1 sat to themselves via the delayed CTV path, with the rest being revaulted. It wouldn't be worth trying to recover that sat, though you would likely still want the watchtower to apply the recovery operation to all the other funds that are spendable via your compromised trigger key.

I don't think the recovery transaction needs to be particularly large though; it would have multiple inputs for each utxo that comprise your vault, with the witness data being:

 * 65B control block (tapleaf version, ipk, one step merkle path for the trigger leaf)
 * 34B script (`<scriptPubKeyHash> VAULT_RECOVER`)
 * 1B output index

If you added a recover authorisation key, that would be an extra 33B to the script and 65B for the additional witness item for the signature. It would also need an additional input to cover fees. Because of the ability to specify the same output index for each input, you'd only need one output, which could also include the change.

So including the extra input for the funds Mallory is trying to steal would be an extra 66-91 vbytes by my count.

[quote="harding, post:1, topic:521"]
Then Mallory creates another offchain spend from the revaulted output in another zero-fee transaction that steals a relatively small additional amount of Bob’s funds. Mallory repeats this process until she has roughly a block full of small-value spends from Bob’s vault.
[/quote]

Seems like a fair attack, though more in the realm of vandalism than theft: Mallory will be losing more income by not not mining txs from the normal mempool than they'll be gaining in the dust they're stealing from Bob.

[quote="harding, post:1, topic:521"]
if the trigger authorization script included a `1 OP_CSV` delay, Mallory could only practically create one uneconomic withdrawal before Bob had a chance to sweep the rest of his funds into recovery. This would, of course, reduce the flexibility of vaults for frequent spending and make them harder to compose with scripts for contract protocols.
[/quote]

Yeah, I think this fixes that attack and is good advice. I don't think it's a big constraint on potential contracting things -- if you're scheduling multiple payments from a vault in the same block, you might as well just include them all in a single CTV, since CTV can commit to multiple outputs. Your contracting tool already needs to be able to decompose the taproot scriptPubKey to get to the CTV, going a step further and decomposing the CTV to get to just one desired output seems fine. And given there's a potential recovery step, any contracting things like that can't really rely on the payment until the CTV script path spend is confirmed anyway.

-------------------------

