# Challenge: Covenants for Braidpool

mcelrath | 2025-01-06 20:20:08 UTC | #1

I present a challenge for all the covenant aficionados out there, and your favorite opcode. I'm looking for specific covenant proposals which accomplish the following:

## Background

[Braidpool](https://github.com/braidpool/braidpool) (A decentralized mining pool) will need to custody coinbase rewards across multiple blocks. For this I have proposed using a FROST federation which signs two chained transactions RCA and UHPO for this:

Coinbase -> RCA -> UHPO

[Rolling Coinbase Aggregation (RCA)](https://github.com/braidpool/braidpool/blob/6bc7785c7ee61ea1379ae971ecf8ebca1f976332/docs/braidpool_spec.md#payout-update): This is an Eltoo transaction on-chain that aggregates the most recent coinbase with the previous (aggregated) coinbase(es).

[Unspent Hasher Payout Object (UHPO)](https://github.com/braidpool/braidpool/blob/6bc7785c7ee61ea1379ae971ecf8ebca1f976332/docs/braidpool_spec.md#unspent-hasher-payment-output): One or more transactions that spends the RCA and pays all hashers in proportion to the work contributed. This transaction is timelocked and cannot be broadcast until the end of a difficulty adjustment window (the "settlement period"), at which time the most recently created UHPO is broadcast (with all previous UHPOs invalidated by the Eltoo mechanism of its parent RCA) and pays all miners for the preceding settlement period.

I have proposed using a [FROST federation](https://github.com/pool2win/frost-federation) in development by @jungly for this. Our construction has the following properties:

1. The UHPO transaction is constructed and signed or committed to first. All federation signing nodes can independently construct and verify the payouts in the UHPO transaction because they each have a copy of the Braidpool DAG with share information.
2. The RCA transaction is signed or committed to after the UHPO transaction is signed by a quorum of federation nodes.
3. The Byzantine fault tolerant signing quorum is 3f+1 or 67% of the signing nodes.

The FROST Federation has a number of drawbacks, not the least of which is a 51% attack (or 67% attack) on the pool, if federation membership is decided by PoW, which could steal all funds. If federation membership is decided by some other manner this results in a highly political process at odds with the decentralized nature of Bitcoin, and theft is still a possibility. I wish to consider a covenant-based alternative which eschews the need for custody entirely, or achieves a "can't-be-evil" philosophy where only the correct payouts can happen.

## Requirements

1. The RCA transaction must either: be aggregated with another coinbase to create a new RCA transaction (Eltoo) OR (after a timelock) be spent in a UHPO transaction. It must not be spendable in any other way.
2. A block with an incorrect payout not satisfying the requirements of the RCA or UHPO must be spendable (after a possible timelock) as solo-mined block, and will not be considered a "share" by the pool.
3. Coinbase outputs must have a timelock that falls back to solo mining, so that in the event of a pool failure (RCA not constructed/mined), all miners with blocks not aggregated in the RCA can claim their blocks as solo mined blocks.
4. Each new block found by the pool modifies the set of payouts in the next UHPO, so the UHPO from the previous block can't know the correct payouts in the next block and can't commit to it.
5. In the event of an incorrect RCA or UHPO transaction or 51%/67% attack on the FROST federation signing, and a theft attempt is broadcast or otherwise detected against the RCA or UHPO, the "last-known-good" UHPO transaction must become immediately broadcastable, evading its usual timelock, and ensuring that the RCA Eltoo mechanism is invalidated for future blocks. The pool becomes entirely solo-mining or shuts down in this case.
6. Assume that information necessary to construct the UHPO is known at the time a block template is constructed, and can be committed to in a block. (It's the INPUT to this UHPO tx that requires custody across multiple blocks)
7. You must take into account the 100 block coinbase maturity rule.

A couple suggestions:
1. You MAY choose that the UHPO has all the same outputs as the previous UHPO, with amounts that are greater than or equal to the previous UHPO, with additional outputs added.
2. You MAY spend the entire RCA to fees if it is included in a pool's block, in order to get rid of the RCA and have the UHPO directly committed to by the coinbase output. (As long as this can't be stolen by another miner)
3. You MAY eschew the RCA mechanism in favor of taking all coinbase outputs from the previous settlement period as inputs.

Something deviating from the above outlined structure is welcome as well, if it achieves the goal of ensuring that everyone gets paid correctly and uses covenants.

A proof of impossibility would be welcome as well.

P.S. I have avoided describing other approaches that we might take here to keep it concise. (e.g. the P2Pool/Eligus/OCEAN mechanism of paying PPLNS in coinbases -- we can always fall back on that but it's been done has well known drawbacks) If that info would be helpful I can write it up in a separate post or doc on the Braidpool github. (msg me privately if that is of interest)

-------------------------

