# Question about OP_CTV and Non-Recursive Covenants

josh | 2025-03-12 23:02:27 UTC | #1

*This post is not an endorsement of OP_CTV or any specific covenant proposal.*

Hi all, I’m trying to learn more about the OP_CTV proposal and covenants in general. My understanding is that OP_CTV was explicitly designed to avoid recursive covenants, which is why users of OP_CTV can commit to the outputs of a transaction, but not the inputs.

First, is that explanation correct for why the inputs of a transaction cannot be committed to in OP_CTV?

For context, I’ve been thinking about how to create multiple buy offers for multiple UTXOs at once, in a single PSBT. As it stands, it is impossible to do this without creating a separately signed PSBT for every UTXO you wish to make an offer for.

This is a pressing problem faced by the runes protocol. Users can make a trustless PSBT offer, which anyone can accept, but users cannot make the equivalent PSBT buy offer. Doing so would require making a separate PSBT offer for every single UTXO holding the rune the user wishes to make an offer for.

Initially, I thought OP_CTV might provide a solution to this problem. A user would commit to a taproot output with separate script paths for every UTXO the user wishes to make an offer for. The executed script would commit to the outputs and the inclusion of the input of interest. An owner of any of the UTXOs could then accept the offer by signing and broadcasting their version of the reveal transaction (with perhaps a subsequent CPFP fee bump).

Sadly, this doesn’t appear possible with OP_CTV, since you can’t commit to other inputs. At the same time, I wouldn’t say that this feature alone is worth opening the can of worms that is recursive covenants.

This brings to my second question. Would it ever make sense to let users commit to the *location* of an input (the block height, transaction index, and output index of the outpoint)?

This would add complexity to script validation and create a non-txid dependency, but to my knowledge, this wouldn’t enable recursive covenants. Recursive covenants are impossible if you need to know the mined location of an input, which can’t be known ahead of time or while executing the transaction.

There are certain tradeoffs, like the risk the location changes in the event of a reorg, and the difficulty of evaluating the script without access to the full UTXO set, but this functionality would be good enough for the use case I described above.

Would there be a better way to enable multi-UTXO offers, without also enabling recursive covenants?

Appreciate your help understanding OP_CTV and thinking through this problem!

*Disclaimer: I do not own any runes. My interest in runes is chiefly as a protocol that could facilitate the issuance of stocks and other securities natively on Bitcoin. I have no financial stake in the latter, other than a belief that Bitcoin-native capital markets, particularly for companies on a Bitcoin treasury strategy, could deepen people’s understanding and appreciation of Bitcoin as money.*

 *I might also add that this post is purely exploratory and for my own education. I’m not sure yet where I fall on the existing soft fork debate.*

-------------------------

ariard | 2025-03-13 00:27:38 UTC | #2

See the original post of 2013 by Gregory Maxwell on the idea of covenant.

[“CoinCovenants using SCIP signatures, an amusingly bad idea”.](https://bitcointalk.org/index.php?topic=278122.0)

There is a lot of comments on the idea of recursivity among a chain of transactions.

-------------------------

JeremyRubin | 2025-03-13 00:40:24 UTC | #3

These are some really solid questions and ideating!


1. CTV can actually accept multiple inputs, and you can use them to write option contracts. This would enable to do e.g. register an on-chain auction of a ordinal, if you wanted that. There's code some stuff like that here: https://github.com/sapio-lang/sapio/tree/master/sapio-contrib/src/contracts/derivatives
2. Your idea for relative references is similar to John Law's inherited ID's proposal. See https://gnusha.org/pi/bitcoindev/CAD5xwhh-1zUbPgYW6hE8q3CmhFZFdEqjx5pB7+VFM4mV=1FfaQ@mail.gmail.com/

-------------------------

