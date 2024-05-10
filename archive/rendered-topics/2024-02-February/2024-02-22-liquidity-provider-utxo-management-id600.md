# Liquidity provider utxo management

remyers | 2024-02-22 14:49:27 UTC | #1

# Problem

Current wallets are not optimized for liquidity providers like Lightning nodes that fund requests for liquidity via [liquidity ads](https://github.com/lightning/bolts/pull/878). 

Liquidity provider wallets have a usage pattern with these characteristics:
 - frequently spend utxos of known amounts to liquidity transactions
 - infrequently refill the wallet when the balance gets low
 - liquidity transactions may not confirm before the next spend (simultaneous spends)
 - liquidity transactions may not confirm at all due to inputs from other wallets (unsafe change)

Ideally every liquidity transaction could be funded with a single input and without a change output. This has the lowest weight/fees and does not tie up funds in a change output that may take a long time to become spendable or that may not confirm at all (see [liquidity griefing](https://delvingbitcoin.org/t/liquidity-griefing-in-multi-party-transaction-protocols/264/7)).

Given these constraints and goals, how should a wallet manage the utxo set for liquidity providers? 

We have a proposed solution, but would appreciate any thoughts on how best to optimize utxo management to decrease aggregate on-chain fees and increase the ability to fund multiple simultaneous funding transactions.

# Proposed Solution

To fund a new liquidity transaction, perform the following steps:

1. Import a user defined set of target utxos we want to have available to fund liquidity transactions. 

To provide liquidity of 0.001 BTC with a single input and no change you need a utxo of that amount plus the cost of the transaction at the current fee rate. Each target requires a bucket of utxos suitable for different fee rates. The number of utxos in a bucket depends on how many simultaneous transactions you need to support and the range of likely fee rates. Coin selection algorithms will select a utxo that overpays fees when it costs less than adding an additional input and change output.

2) Proactively refill buckets in some situations.

When fee rates are low, or a bucket is extremely depleted, we should proactively create change outputs to refill buckets. Initially I propose adding the largest utxo that is not in a bucket as an additional input to the transaction.  This input will be split in step 4 to refill depleted buckets. We should not deplete full buckets to refill depleted ones. At that point it makes more sense to refill the wallet with new value from cold storage.

3) Opportunistically refill buckets if coin selection adds a change output.

When coin selection can not find a solution with a single input and no change, it will search for a solution with a change output of at least some minimum amount of value. That change output should opportunistically refill one (or more) of our target buckets. This can be done in different ways. Initially I propose setting the minimum change output size to an amount that refills the most depleted target bucket. This may add utxos from existing buckets. 

4) Split the change output, if any, to refill buckets.

When coin selection generates a transaction with a non-zero change amount, that amount should be split into outputs that refill our target buckets.  I initially chose to iteratively select the most depleted bucket until the remaining change amount was less than the next depleted bucket or when all buckets are at capacity. The remaining change amount then goes into the last change output.   

# Testing

I ran some test with a modified fork of [coin_selection_simulation](https://github.com/achow101/coin-selection-simulation) and with the changes to Bitcoin core in draft PR [#29422](https://github.com/bitcoin/bitcoin/pull/29442). 

# Results

My initial results using the above changes appear to yield a 15% reduction in on-chain fees compared to default coin selection. I think further improvements and refinements can be made and I would appreciate any suggestions.

# Questions

- Does the concept of performing pre/post processing on the parameters of coin selection make sense or is there some way to optimize coin selection itself for this scenario?
- Would it make sense to only use branch-and-bound and [CoinGrinder](https://github.com/bitcoin/bitcoin/pull/27877) as coin selection algorithms for this problem to reduce fees?
- Any ideas for a better algorithm for adding inputs and splitting change to converge on our target utxo set with the least on-chain fees?
- Are there other use cases that could use this functionality? coin join users perhaps?

-------------------------

remyers | 2024-03-02 13:03:49 UTC | #2

Perhaps it would help people understand this proposal faster if I translated it to pseudo code.

## Inputs:

* `Spend_Amount`: the amount from a set of liquidity amounts (eg. 10,000 sat, 50,000 sat, 200,000 sat, 1,000,000 sat)
* `Effective_Feerate`: the feerate we want to pay for the transaction

## Outputs:
* `Liquidity_Transaction`: a transaction funded by `Funding_Inputs` and with optional `Change_Outputs` added
  * `Funding_Inputs`: a subset of `Available_Coins` with a total amount greater than `Spend_Amount` 
  * `Change_Outputs`: a set of outputs that will be added to `Available_Coins` when the `Liquidity_Transaction` confirms

## Configuration Data:

```c
Target_Bucket {
  start_satoshis     // defined by the user
  end_satoshis       // defined by the user
  target_utxo_count  // defined by the user
  current_utxo_count // computed from available coins and pending txs

  Capacity() = current_utxo_count / target_utxo_count 
  Target_Amount() = random(start_satoshis, end_satoshis)
}
```

* `Target_Buckets` = array of `Target_Bucket`, one element per possible `Spend_Amount`
* `Available_Coins`: set of confirmed and unspent UTXOs in the wallet
* `Unconfirmed_Liquidity_Txs`: set of unconfirmed `Liquidity_Transactions` previously created with `Fund_Liquidity()`
* `Bucket_Refill_Feerate`: the feerate below which we should create more change outputs than we consume as inputs from the `Target_Buckets`.

## Goals:
* Find the set of `Funding_Inputs` and `Change_Outputs` with the least `Cost` for the the current `Spend_Amount` and `Effective_Feerate`
* `Cost`: is the amount of all `Funding_Inputs` minus the `Spend_Amount` and minus the `Change_Outputs`
* Minimize the overall `Cost` over many transactions with different `Spend_Amount` and `Effective_Feerate`
* The set of `Available_Coins` should always have enough value to find a solution

## Algorithm:

```c
Fund_Liquidity(Spend_Amount, Effective_Feerate)
  Funding_Inputs = []
  Change_Outputs = []
  
  // Compute the `current_utxo_count` for each target bucket from the available coins and  
  // the change outputs of pending liquidity txs.
  Target_Buckets = Compute_Capacity(Target_Buckets, Available_Coins, Unconfirmed_Liquidity_Txs)
  
  // Find the target bucket with least capacity().
  Min_Target_Bucket = Target_Buckets.Min_Capacity()
  
  if Min_Target_Bucket.Capacity() < 0.7:
    // Check if the capacity of our least full bucket is very low, or fees are low.
    if Min_Target_Bucket.Capacity() < 0.3 or Current_Feerate < Bucket_Refill_Feerate:
      // Ensure we create change to refill buckets by adding the largest coin we can spend as an input.
      Largest_Coin = Available_Coins.Max()
      Preset_Inputs = [Largest_Coin]
  
  // Add sufficient new inputs to fund the transaction with an optional change output.
  Change_Target_Amount = Min_Target_Bucket.Target_Amount()
  CS_Result = SelectCoins(Available_Coins, Funding_Inputs, Change_Target_Amount)
  
  if CS_Result.Change_Output_Amount > 0:
    // If a change output is added, split the change output into multiple outputs (see below)
    Split_Change_Outputs = Split_Change(CS_Result.Change_Output_Amount, Change_Target_Amount, Target_Buckets)
    Tx = CreateTransaction(CS_Result.Inputs, Split_Change_Outputs)
    
  else:
    // Otherwise, create a changeless tx.
    Tx = Change_Outputs(CS_Result.Inputs, CS_Result.Change_Output_Amount)
```

## Splitting Change

We split the single change output added by coin selection into multiple change outputs that opportunistically refill target buckets with random amounts within the range of the most depleted buckets first.

```c
Split_Change(Change_Output_Amount, Change_Target_Amount, Target_Buckets):
  // Add the initial change target used by coin selection to the array of output amounts.
  Change_Outputs = [Change_Target_Amount]

  // Deduct the initial output amount from our total change amount.
  Change_Amount = Change_Output_Amount - Change_Target_Amount
  
  // Increment the count of the utxo target bucket that matches our initial output.
  Target_Bucket = Target_Buckets.Find(Change_Target_Amount)
  Target_Bucket.current_utxo_count += 1

  do {
    // Find the target bucket with least capacity().
    min_target_bucket = Target_Buckets.Min_Capacity()
    
    if min_target_bucket.current_utxo_count >= min_target_bucket.target_utxo_count:
      // If no target buckets at less than full capacity; exit the loop.
      break
      
    // Compute a random target amount in the range of the least full bucket.
    target_amount = min_target_bucket.target_amount()

    if change_amount < (change_fee + target_amount):
      // If there is not enough change available for a new output; exit the loop.
      break
    
    // Deduct the change amount AND the fee cost of adding a new output from the change amount.
    change_amount -= change_fee + target_amount
    
    // Increment the current count of the least full target bucket.
    min_target_bucket.current_utxo_count += 1
    
    // Add the new change output to the end of the array of output amounts.
    change_outputs = change_outputs + target_amount
  }

  // any remaining change goes into the last change output
  change_outputs.last() += change_amount

  return change_outputs
```

-------------------------

murch | 2024-03-18 19:48:30 UTC | #3

This seems like a reasonable approach.

[quote="remyers, post:1, topic:600"]
When fee rates are low, or a bucket is extremely depleted, we should proactively create change outputs to refill buckets. Initially I propose adding the largest utxo that is not in a bucket as an additional input to the transaction.
[/quote]

I’m confused here by the phrasing of an *additional* input? If the feerate is higher and you still need to refill, why not just pick a single large non-bucketed UTXO and create a several change outputs for the bucket you want to refill? If the feerate is low enough that you are proactively refilling buckets, why not sum up the amounts of several or all of the UTXOs you want to create and create all of those UTXOs alongside the liquidity transaction output by picking as many of the large non-bucketed UTXOs as necessary to create them?

[quote="remyers, post:1, topic:600"]
That change output should opportunistically refill one (or more) of our target buckets. This can be done in different ways. Initially I propose setting the minimum change output size to an amount that refills the most depleted target bucket. This may add utxos from existing buckets.
[/quote]

Perhaps you could make a few attempts at trying to find a "changeless solution" to creating the liquidity transaction output plus one, two, or three amounts that your buckets are missing. You could perhaps generate a dozen or so different target amounts by combining various missing amounts and try to build transactions for each of them independently, and then use whatever gets a hit. I’m surprised that you consider using bucketed UTXOs for this—I expect that bucketed UTXOs would only be a fallback if you cannot fulfill the amount from non-bucketed UTXOs?

[quote="remyers, post:1, topic:600"]
* Does the concept of performing pre/post processing on the parameters of coin selection make sense or is there some way to optimize coin selection itself for this scenario?
[/quote]

I am not sure I understand which coin selection parameters you consider pre- and post-processing here. If you mean the `min_change`, I think that you might get limited leverage out of that. I expect that the feerate is foreign determined by the general mempool situation. You may consider creating P2WPKH and P2TR outputs depending on the current feerate and what sort of change outputs you are creating. If you are creating bucketed change outputs at low feerates, you may want to make P2TR outputs, while you might want to opt for P2WPKH non-bucketed change outputs when building transactions at high feerates. Are there other parameters that you were considering?

[quote="remyers, post:1, topic:600"]
* Would it make sense to only use branch-and-bound and [CoinGrinder](https://github.com/bitcoin/bitcoin/pull/27877) as coin selection algorithms for this problem to reduce fees?
[/quote]

I would agree that CoinGrinder and BnB would be more useful in your scenario than Knapsack and SRD. It might make sense to modify the `sendmany` RPC to allow restricting the coin selection to a subset of the coin selection algorithms or to patch out the calls to Knapsack and SRD from your node. Given that the post-processing of input set candidates is currently only based on the waste metric, It seems to me that you might want one call with the target consisting only of the liquidity transaction output, and if it doesn’t return a changeless solution, several calls with targets composed from the liquidity transaction output plus one or multiple of the amounts missing from buckets. Given the intent to refill the buckets, perhaps the latter calls should be restricted to the non-bucketed UTXOs. Which brings me to the question, how would you distinguish non-bucketed and bucketed UTXOs? Would they be kept in separate wallets, separate amount ranges, marked in some manner?

[quote="remyers, post:1, topic:600"]
* Any ideas for a better algorithm for adding inputs and splitting change to converge on our target utxo set with the least on-chain fees?
[/quote]

The approach seems reasonable to me. The only thing that comes to mind right now is that I have seen cases of oversplitting before: perhaps one thing to look out for in your simulations would be how often UTXOs split up later get recombined when a larger denomination would have been required and was unavailable. I suspect that being too proactive in refilling buckets may actually reduce the savings.

[quote="remyers, post:1, topic:600"]
* Are there other use cases that could use this functionality? coin join users perhaps?
[/quote]

Your usage pattern reminds me of send-only exchange wallets, but there the possibility to batch many withdrawals into one transaction shifts the concerns to only needing a sufficient minimum of large-amount UTXOs rather than numerous specific values.

-------------------------

remyers | 2024-03-20 09:12:04 UTC | #4

[quote="murch, post:3, topic:600"]
[quote="remyers, post:1, topic:600"]
When fee rates are low, or a bucket is extremely depleted, we should proactively create change outputs to refill buckets. Initially I propose adding the largest utxo that is not in a bucket as an additional input to the transaction.
[/quote]

I’m confused here by the phrasing of an *additional* input? If the feerate is higher and you still need to refill, why not just pick a single large non-bucketed UTXO and create a several change outputs for the bucket you want to refill? If the feerate is low enough that you are proactively refilling buckets, why not sum up the amounts of several or all of the UTXOs you want to create and create all of those UTXOs alongside the liquidity transaction output by picking as many of the large non-bucketed UTXOs as necessary to create them?
[/quote]

By *additional* input I do just mean I select a large non-bucketed UTXO as an input - just as you suggest.

### Improvement 1
When the current feerate is low and buckets are depleted, that is a good suggestion to target the spend for what we need to refill our buckets . I'll add code to select only the non-bucketed UTXOs as inputs and also avoid trying to refill more buckets than we have available to spend from non-bucketed UTXOs.  I'll add it to my list of improvements to try.

-------------------------

remyers | 2024-03-20 14:40:41 UTC | #5

[quote="murch, post:3, topic:600"]
Perhaps you could make a few attempts at trying to find a “changeless solution” to creating the liquidity transaction output plus one, two, or three amounts that your buckets are missing. You could perhaps generate a dozen or so different target amounts by combining various missing amounts and try to build transactions for each of them independently, and then use whatever gets a hit. I’m surprised that you consider using bucketed UTXOs for this—I expect that bucketed UTXOs would only be a fallback if you cannot fulfill the amount from non-bucketed UTXOs?
[/quote]

The reason I considered spending bucketed UTXOs is that I assumed that all of the wallet value would be in bucketed UTXOs except for a few original funding UTXOs larger than the largest bucket. But that mental model may not be correct because when channels are closed they create UTXOs with random amounts that may fall outside of our buckets.

### suggestion 2
I think what you're suggesting here is that when we cannot create a change-less funding transaction we should try to target change that can exactly refill a set of depleted buckets without creating any additional change output.

In my simulations refilling always comes from the largest non-bucketed UTXO. These UTXOs can initially refill all depleted buckets, but eventually either generates a change output that is in a bucket or between buckets. 

Outputs that are between buckets are similar to what can be created when we close channels. Perhaps these UTXOs can be consolidated when fees are low? See also **suggestion 1**.

-------------------------

remyers | 2024-03-20 09:50:16 UTC | #6

[quote="murch, post:3, topic:600"]
[quote="remyers, post:1, topic:600"]
* Does the concept of performing pre/post processing on the parameters of coin selection make sense or is there some way to optimize coin selection itself for this scenario?
[/quote]

I am not sure I understand which coin selection parameters you consider pre- and post-processing here. If you mean the `min_change`, I think that you might get limited leverage out of that. I expect that the feerate is foreign determined by the general mempool situation. You may consider creating P2WPKH and P2TR outputs depending on the current feerate and what sort of change outputs you are creating. If you are creating bucketed change outputs at low feerates, you may want to make P2TR outputs, while you might want to opt for P2WPKH non-bucketed change outputs when building transactions at high feerates. Are there other parameters that you were considering?
[/quote]

I only mean by pre/post processing that we are not monkeying with the internals of coin selection, but just the parameters passed in and adjusting the transaction returned.

I was only looking at `min_change` as a parameter for coin selection.

### suggestion 3

Good idea to adjust which type of outputs we create based on the fee environment. This seems like an additional optimization we can adopt in addition to any other changes.

-------------------------

remyers | 2024-03-20 14:39:32 UTC | #7

[quote="murch, post:3, topic:600"]
I would agree that CoinGrinder and BnB would be more useful in your scenario than Knapsack and SRD. It might make sense to modify the `sendmany` RPC to allow restricting the coin selection to a subset of the coin selection algorithms or to patch out the calls to Knapsack and SRD from your node. Given that the post-processing of input set candidates is currently only based on the waste metric, It seems to me that you might want one call with the target consisting only of the liquidity transaction output, and if it doesn’t return a changeless solution, several calls with targets composed from the liquidity transaction output plus one or multiple of the amounts missing from buckets. Given the intent to refill the buckets, perhaps the latter calls should be restricted to the non-bucketed UTXOs.
[/quote]

I created a branch where I can set which algos are enabled. I have been running some simulations to compare using bnb+cg vs all algos. I'm using the same funding scenario file from my other tests, but none of the other changes related to targeting buckets. 

Counter intuitively my initial tests showed all algos having slightly lower total fees and fewer median inputs than using only bnb+cg. I'm running more tests to try to figure out why.

[quote="murch, post:3, topic:600"]
Which brings me to the question, how would you distinguish non-bucketed and bucketed UTXOs? Would they be kept in separate wallets, separate amount ranges, marked in some manner?
[/quote]

I do not restrict which UTXOs coin selection can pick, except when refilling buckets I pre-select as an input the single "maximum value UTXO that is not in a bucket." I use the bucket ranges preloaded from a json file to pre-filter the available coins based on if they are in a bucket or not.

That makes sense to initially try to select from UTXOs that are in a bucket to create a changeless transaction, but when that fails try to select from only non-bucketed UTXOs. This is related to **suggestion 1**.

-------------------------

remyers | 2024-03-20 15:37:05 UTC | #8

[quote="murch, post:3, topic:600"]
This seems like a reasonable approach.
[/quote]

Thanks for sharing your thoughts and suggestions Murch! I've noted three concrete things I can try from your reply. I will work to produce some simulation results that can test these ideas.

One additional idea I'm exploring is to take advantage of an addition degree of freedom unique to Lightning funding transactions: we do not need to hit the exact amount requested. Any value over (or under) the target funding amount will do when funding a channel. The funder still controls the funding amount in the Lightning channel and can charge for the exact amount added.

When doing coin selection for BnB the `excess` amount is considered waste, but we can add it to the target amount instead of letting it go to fees. If one of your bucketed UTXOs is a little to big or small (eg. within 5%) it should be possible to use it for a changeless funding tx without any waste.

I'm testing out this idea now in simulations to see what impact it has on total fees.

-------------------------

murch | 2024-03-21 19:36:01 UTC | #9

[quote="remyers, post:7, topic:600"]
Counter intuitively my initial tests showed all algos having slightly lower total fees and fewer median inputs than using only bnb+cg. I’m running more tests to try to figure out why.
[/quote]

I expect that you did, but I assume that you set the `consolidationfeerate` to 1000 or 0, which would make CoinGrinder work at every feerate? One downside might be that CoinGrinder prefers the lower input amount all other things equal, and therefore you might create useful change less often. Do you already set custom minimum change amounts in this experiment or is the `min_change` behavior the default one?

[quote="remyers, post:8, topic:600"]
One additional idea I’m exploring is to take advantage of an addition degree of freedom unique to Lightning funding transactions: we do not need to hit the exact amount requested. Any value over (or under) the target funding amount will do when funding a channel. The funder still controls the funding amount in the Lightning channel and can charge for the exact amount added.
[/quote]

That’s an interesting idea. Perhaps at that point, BnB is actually not useful at all, but you could try using just several calls to CoinGrinder with various `min_change` values from minimal plus different bucket minimums.

-------------------------

remyers | 2024-03-22 09:23:39 UTC | #10

[quote="murch, post:9, topic:600, full:true"]
[quote="remyers, post:7, topic:600"]
Counter intuitively my initial tests showed all algos having slightly lower total fees and fewer median inputs than using only bnb+cg. I’m running more tests to try to figure out why.
[/quote]

I expect that you did, but I assume that you set the `consolidationfeerate` to 1000 or 0, which would make CoinGrinder work at every feerate? One downside might be that CoinGrinder prefers the lower input amount all other things equal, and therefore you might create useful change less often. Do you already set custom minimum change amounts in this experiment or is the `min_change` behavior the default one?
[/quote]

We are setting `consolidationfeerate` to 0, so CoinGrinder should always be preferred when there is no BnB solution with fewer inputs in the `all algo` tests. These tests were not using `min_change` to adjust the change target - just the default change behavior. The scenario file was one that mostly sends bucketed amounts, and rarely gets a payment. 

[quote="murch, post:9, topic:600, full:true"]
[quote="remyers, post:8, topic:600"]
One additional idea I’m exploring is to take advantage of an addition degree of freedom unique to Lightning funding transactions: we do not need to hit the exact amount requested. Any value over (or under) the target funding amount will do when funding a channel. The funder still controls the funding amount in the Lightning channel and can charge for the exact amount added.
[/quote]

That’s an interesting idea. Perhaps at that point, BnB is actually not useful at all, but you could try using just several calls to CoinGrinder with various `min_change` values from minimal plus different bucket minimums.
[/quote]

My hope was to make it so that BnB succeeds more often by finding a bucketed UTXO that may have too much (or too little) effective value at the current fee rate. Currently any `excess` increases the waste (or disqualifies) such choices but for funding liquidity it is not actually wasted (or a reason to exclude the UTXO) if the value is "close" to what we want for funding.

Not having change is still a big advantage for liquidity management so single-input changeless BnB solutions should be preferred. Second best is a single input with enough extra value to create change outputs that refill our buckets. Adding multiple inputs to consolidate small non-bucketed UTXOs should be a last resort, or occur when fees are low (as you suggested earlier).

-------------------------

remyers | 2024-05-10 14:29:04 UTC | #11

I've replaced draft  [PR 29442](https://github.com/bitcoin/bitcoin/pull/29442) with a  simpler draft [PR 30080](https://github.com/bitcoin/bitcoin/pull/30080) that only adds new coin selection parameters, primarily `add_excess_to_recipient_position`.

A changeless transaction may select inputs with excess value over what is needed to reach the desired fee rate and output targets. Currently that excess value is considered waste and burned as fees. You may prefer any excess value to be added to an output you control. When fees are high the cost of adding a change output increases and so does the amount of potential excess value spent as fees.

Anytime you retain the value from an over payment then it makes sense to not lose the excess to fees. 

When splicing on-chain value into a Lightning channel it would be better to get credit for the excess value off-chain.  Lightning Service Providers (LSPs) that use [liquidity ads](https://github.com/lightning/bolts/pull/878) will prefer to add any excess as extra inbound liquidity for clients instead of paying the excess as fees.

LSPs that sell inbound liquidity will prefer to maintain a set of UTXOs that are sized to fund a changeless transactions of pre-defined funding amounts *plus* the fee to spend each amount at various feerates without additional inputs or an added change output.

The best case for an LSP is to always have a confirmed UTXO of the requested funding amount plus the fee to spend it at the current feerate. A transaction with one taproot input and two taproot outputs (both a funding and a change output) is approximately 37% more expensive than not having a change output.

If the excess value from a changeless UTXO can be retained by the LSP as extra inbound liquidity then the LSP can cover the range of likely fee rates with fewer UTXOs. Each UTXO can be used at a specific feerate and also any lower feerates up to the point that the excess is greater than the cost of adding a change output.  The potential space of funding amounts and feerates can be "tiled" so that UTXOs exists at non-overlapping intervals smaller than the threshold for adding a change output. This approach can minimize the fee cost of providing liquidity to near the ideal case of always having a UTXO of the amount needed for a changeless funding transaction.

There are likely other use cases where a user may want excess fees applied to their target output because the extra value would be credited to them by a custodian or retailer.

This new proposal assumes the LSP proactively selects a larger UTXO than is needed when fees are low to create change outputs that refill the cache of tiled UTXOs.

-------------------------

