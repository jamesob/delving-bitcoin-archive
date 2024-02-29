# Add a SIGHASH_DOUBE flag

cmd | 2024-02-28 22:34:45 UTC | #1

Hello. I would like to discuss the feasibility of implementing a SIGHASH_DOUBLE flag, which would sign for two outputs in a transaction.

I would like this feature as an option when signing transactions, so that I can create a PSBT that can be combined with similar PSBTs in a non-interactive way (similar to SIGHASH_SINGLE), that includes both a payout output and a change output.

In regards to how one would verify a tx with such a flag present, this would be my naive approach:

Imagine we have a tx with four inputs, six outputs, and the following config:
```
vin0: SIGHASH_DOUBLE
vin1: SIGHASH_DOUBLE
vin2: SIGHASH_SINGLE
vin3: SIGHASH_ALL
```

1. Scan all signature flags in a tx, check if SIGHASH_DOUBLE exists.

2. If true, we will count two indices in the output index:
  - one index starting at 0: (n0).
  - one index starting at vin_count + 1: (n4).

3. Initially, treat SIGHASH_DOUBLE inputs similar to SIGHASH_SINGLE, by pairing them with their adjacent output. This gives us consensus on the index of the first output: (vin0/vout0) and (vin1/vout1).

4. For each SIGHASH_DOUBLE input, we will also observe the index starting at vin_count + 1 (n4). This gives us consensus on the second output: (vin0/vout4) and (vin1/vout5).

That's about it. Essentially we use vin_count to offset a second index for the outputs > vin_count. This should work for SIGHASH_DOUBLE, SIGHASH_TRIPLE, or as high as you want to go.

I believe this would be simple enough for consensus on validating signatures. It may help to organize inputs/outputs based on their sighash flag, so that when you are consolidating PSBTs:

- SIGHASH_DOUBLE pairs goes first,
- then SIGHASH_SINGLE pairs,
- then remaining inputs,
- then SIGHASH_DOUBLE outputs,
- then remaining outputs.

Worst-case, your tx gets rejected from the mempool, because you didn't organize things properly.

Edit: I forgot to mention the odd case where you contribute inputs without an output. I'm not sure how often that comes up, but I don't believe these inputs would be compatible with a SIGHASH_DOUBLE flag. Which I think is okay, as the worst-case is still just your tx being rejected from the mempool.

Edit2: This also wouldn't work with other PSBTs that consolidate inputs, as they will be contributing more inputs than outputs. Not sure how to get around that, other than making SIGHASH_DOUBLE only compatible with SIGHASH_SINGLE. But maybe there is a way.

As for how this could be soft-forked into bitcoin, I would look at some way of extending the SIGHASH_NONE flag, so that older nodes will skip validation. I believe taproot introduced a few ways of extending signature validation, though I'm not sure which method would be best to use.

Another interesting use case for SIGHASH_DOUBLE (with ANYONECANPAY), is that you can have a mempool of single-spend PSBTs. This wouldn't be for coin laundering, but to save 7-8 vbytes of overhead per tx, as miners could just consolidate these into a final tx when making a block.

Anyway, it would be a nice feature to have, and it feels like a missing flag option to sign for both a payout and change output in a composable way.

I haven't put a terrible amount of thought into this, so any feedback from smart minds would be greatly appreciated.

-------------------------

moonsettler | 2024-02-29 00:19:03 UTC | #2

why not SIGHASH_BUNDLESTART/SIGHASH_INBUNDLE?

https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2018-April/015862.html

-------------------------

