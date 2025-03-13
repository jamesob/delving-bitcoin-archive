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

ajtowns | 2025-03-13 00:46:49 UTC | #4

[quote="josh, post:1, topic:1517"]
My interest in runes is chiefly as a protocol that could facilitate the issuance of stocks and other securities natively on Bitcoin.
[/quote]

Stocks and securities seem like a perfect case for centralised management -- whichever company issued the stocks or physically controls whatever is being securitised has a strong interest in managing this stuff properly and if you can switch from a decentralised model to a centralised one you can make things much more efficient. Doing this via an "ecash" model would allow anonymous trades if that's desired, having wrapped BTC or stablecoins available on the same platform would allow automated market makers, and aggregating multiple securities onto a single "stock exchange" platform would make for easy cross-security trades. So personally, my expectation is that doing these things on bitcoin will end up out-competed by custom solutions.

[quote="josh, post:1, topic:1517"]
For context, I’ve been thinking about how to create multiple buy offers for multiple UTXOs at once, in a single PSBT. As it stands, it is impossible to do this without creating a separately signed PSBT for every UTXO you wish to make an offer for.

This is a pressing problem faced by the runes protocol. Users can make a trustless PSBT offer, which anyone can accept, but users cannot make the equivalent PSBT buy offer. Doing so would require making a separate PSBT offer for every single UTXO holding the rune the user wishes to make an offer for.
[/quote]

I'm not sure I understand the problem statement here. I think what you're saying is essentially "I know of a dozen existing txids, each of which imply control of a different rune R1 .. R12, and I am happy to buy any one of those runes for X BTC".

Because rune transfers can be modified by the presence of an OP_RETURN output, and you obviously also want to specify your own output, I don't think a SIGHASH_SINGLE signature is sufficient when creating an offer to buy a rune, but a SIGHASH_ALL signature (or CTV hash) would also need to commit to an receiving address for whoever you're paying for the rune, which is almost certainly a different signature for each of different runes you're willing to purchase.

Perhaps using [elements-style introspection](https://github.com/ElementsProject/elements/blob/abbc17048a320910e68e347fa3ebbea770539cda/doc/tapscript_opcodes.md) you could solve that (writing a script that asserts there are only 2 outputs, that the first is your address, and the second is not an OP_RETURN), but you would still need to differentiate the 12 acceptable txids as inputs from any other random input. Doing that via a single PSBT would require a custom field ("I've used this complicated script, which can be satisfied by any of these utxos as the second input along with including one of these [signatures/merkle paths] in the witness"), which doesn't seem much better/different than just having separate PSBTs in the first place.

[quote="josh, post:1, topic:1517"]
My understanding is that OP_CTV was explicitly designed to avoid recursive covenants, which is why users of OP_CTV can commit to the outputs of a transaction, but not the inputs.
[/quote]

If the CTV hash committed to the inputs, then you couldn't construct a (spendable) scriptPubKey that included a CTV hash -- the CTV hash would be `H=hash(SPK, outputs, nlocktime, etc...)`, but the scriptPubKey would be `SPK=hash(H CTV ...)` so every time you calculate H you have to recalculate SPK which means you have to recalculate H, and repeat. Because they're 256-bit hashes, you would have to do about $2^{128}$ calculations to come up with a pair of values that are consistent ([related discussion](https://gnusha.org/pi/bitcoindev/20160108153329.GA15731@sapphire.erisian.com.au/)), at which point you've essentially solved mining, and are probably able to guess everyone's private key.

(The same loop doesn't occur for signatures (which do commit to the inputs) because the signature is included in the witness rather than scriptPubKey)

-------------------------

josh | 2025-03-13 02:12:51 UTC | #5

Thank you for the thoughtful responses! They're super helpful. I'll split up my follow-ups into a few replies.

[quote="ariard, post:2, topic:1517, full:true"]
See the original post of 2013 by Gregory Maxwell on the idea of covenant.

[“CoinCovenants using SCIP signatures, an amusingly bad idea”.](https://bitcointalk.org/index.php?topic=278122.0)

There is a lot of comments on the idea of recursivity among a chain of transactions.
[/quote]

Thanks for sharing this! It's fascinating how long these ideas have been discussed. I can understand why people find recursive covenants concerning.

@ariard Do you think that a covenant that can only commit to the mined location of a spent outpoint (and not the outpoint independent of its location) would be safe from the concerns Maxwell raised of permanently encumbered sats?

-------------------------

josh | 2025-03-13 02:21:42 UTC | #6

[quote="JeremyRubin, post:3, topic:1517, full:true"]

These are some really solid questions and ideating!

[/quote]

Thank you for the encouragement!

[quote="JeremyRubin, post:3, topic:1517, full:true"]

Your idea for relative references is similar to John Law’s inherited ID’s proposal.

[/quote]

Interesting! I hadn't heard of this proposal before. I need to better understand it, but my instinct is that it wouldn't help much with the problem I presented (make a blanket buy offer of X runes of type R for Y sats, across all UTXOs with a balance of at least X runes of type R.

To do this, you need to commit to all the outputs of the "buy" transaction *and* the spent outpoint of the UTXO you wish to buy, because you need assurance that the `OP_RETURN` will direct the right quantity of runes of type R to the output you control.

[quote="JeremyRubin, post:3, topic:1517, full:true"]

CTV can actually accept multiple inputs, and you can use them to write option contracts.

[/quote]

Can CTV commit to the outpoints of the inputs in the transaction? I thought it couldn't, just the sequences, ScriptSigs, input index, and the input count.

Or are you just saying that a transaction using CTV can have multiple inputs?

Appreciate the clarification!

-------------------------

JeremyRubin | 2025-03-13 02:37:26 UTC | #7

[quote="josh, post:6, topic:1517"]
sequences, ScriptSigs, input index, and the input count.
[/quote]

Just the sequences, ScriptSigs, input index, and the input count are committed to. So if you have an "offer" branch where funds need to be added, it can bind to a payment to a particular user.

But if the other input is a p2sh or legacy, the commitment to scriptsig containing a signature will commit to the specific inputs.

So if you gather the signature, e.g., using ANYONECANPAY, then you can get it to work because you avoid the hash cycle and commit to your input and can form a CTV, the CTV would be able to commit to the other inputs.

This is a bit better than using a signature for presigning, since there's no other way to spend once it's in CTV.

-------------------------

josh | 2025-03-13 14:29:22 UTC | #8

[quote="JeremyRubin, post:7, topic:1517, full:true"]
Just the sequences, ScriptSigs, input index, and the input count are committed to. So if you have an "offer" branch where funds need to be added, it can bind to a payment to a particular user.

But if the other input is a p2sh or legacy, the commitment to scriptsig containing a signature will commit to the specific inputs.

So if you gather the signature, e.g., using ANYONECANPAY, then you can get it to work because you avoid the hash cycle and commit to your input and can form a CTV, the CTV would be able to commit to the other inputs.
[/quote]
Oh I got it! That's pretty cool, but sadly I don't think it will help with my use case.

You can create a rune buy offer today for a single UTXO holding $X$ runes of type $R$ by creating a PSBT with SIGHASH_ALL committing to the entire transaction, leaving the seller's input for them to sign. This is how `ord` implements [inscription buy offers](https://github.com/ordinals/ord/blob/ca9950a1dc702bad082ba016c62eaa88456e99fa/src/subcommand/wallet/offer/create.rs).

The problem is that you can't realistically do this at scale, without creating a ridiculous number of signatures. You would need to sign a distinct PSBT for each UTXO with a balance of at least $X$ runes of type $R$, which could number in the 1000s or 10s of 1000s, or more.

Unless I'm missing something, CTV can't help here.

My idea was for the bidder to first create a transaction $T_0$ that commits to a P2TR output that has a unique script spend path $S_i$ for each UTXO $U_i$ they're bidding on. There would be no signatures in the script spend path, just a commitment to a transaction $T_i$ that has the buyer's output, the seller's output, the seller's input, and an `OP_RETURN` that sends $X$ runes to the buyer's output.

The transaction $T_0$ could then be easily broadcast, and any owner of rune $R$ at $U_i$ could create and sign the "reveal" transaction $T_i$ that sends the runes to the bidder and claims the sats for themselves.

This though would require some way to commit to the spent outpoint $U_i$, but I'm afraid there's no way to do that with CTV.

-------------------------

josh | 2025-03-13 14:48:52 UTC | #9

@ajtowns Thank you for the detailed response! Below are some of my thoughts:

[quote="ajtowns, post:4, topic:1517, full:true"]

[quote="josh, post:1, topic:1517"]

My interest in runes is chiefly as a protocol that could facilitate the issuance of stocks and other securities natively on Bitcoin.

[/quote]

...my expectation is that doing these things on bitcoin will end up out-competed by custom solutions.

[/quote]

That's very fair, and I think that's the dominant perspective right now. My perspective is somewhat contrarian. If you're a public company like Microstrategy / Strategy and you want to raise bitcoin *directly from bitcoiners*, you need to go to where the bitcoin is. You can't expect the bitcoin to come to you. Features like speed and programmability are less relevant when all you care about is raising capital and the alternative is DTCC.

Bitcoin holders don’t want to move their bitcoin to an exchange, where it could get frozen, only to then sell for dollars and wait days to move it to a brokerage, just to buy the stock. Nor do they wish to take on the hassle and risk of bridging to another blockchain or giving someone else custody.

If a company issued their stock natively on Bitcoin, a secondary market could develop directly between bitcoin and their stock, and the company could easily raise bitcoin by selling shares into the market. Likewise, they could easily buyback their stock and issue bitcoin dividends, and prospective investors could easily invest or divest with a single trustless transaction, without needing KYC or intermediaries.

[quote="ajtowns, post:4, topic:1517, full:true"]

I’m not sure I understand the problem statement here. I think what you’re saying is essentially “I know of a dozen existing txids, each of which imply control of a different rune R1 … R12, and I am happy to buy any one of those runes for X BTC”.

[/quote]

Here is a more formal problem statement:

*With a single signature, make a blanket buy offer of $X$ runes of type $R$ for $Y$ sats, across all UTXOs with a balance of at least $X$ runes of type $R$.*

This unfortunately isn't practical to do today:

[quote="josh, post:8, topic:1517, full:true"]
You can create a rune buy offer today for a single UTXO holding $X$ runes of type $R$ by creating a PSBT with SIGHASH_ALL committing to the entire transaction, leaving the seller's input for them to sign. This is how `ord` implements [inscription buy offers](https://github.com/ordinals/ord/blob/ca9950a1dc702bad082ba016c62eaa88456e99fa/src/subcommand/wallet/offer/create.rs).

The problem is that you can't realistically do this at scale, without creating a ridiculous number of signatures. You would need to sign a distinct PSBT for each UTXO with a balance of at least $X$ runes of type $R$, which could number in the 1000s or 10s of 1000s, or more.

[/quote]

The idea I was exploring was to commit to multiple possible transaction templates with a single signature, which any of the potential sellers could use to accept their specific buy offer. 

[quote="josh, post:8, topic:1517, full:true"]

My idea was for the bidder to first create a transaction $T_0$ that commits to a P2TR output that has a unique script spend path $S_i$ for each UTXO $U_i$ they're bidding on. There would be no signatures in the script spend path, just a commitment to a transaction $T_i$ that has the buyer's output, the seller's output, the seller's input, and an `OP_RETURN` that sends $X$ runes to the buyer's output.

The transaction $T_0$ could then be easily broadcast, and any owner of rune $R$ at $U_i$ could create and sign the "reveal" transaction $T_i$ that sends the runes to the bidder and claims the sats for themselves.

This though would require some way to commit to the spent outpoint $U_i$, but I'm afraid there's no way to do that with CTV.
[/quote]

Finally,

[quote="ajtowns, post:4, topic:1517, full:true"]

If the CTV hash committed to the inputs, then you couldn’t construct a (spendable) scriptPubKey that included a CTV hash...

[/quote]

I'm not sure I totally follow, but I'd like to. Just to clarify, I'm not suggesting that you commit to the full input, including the scripts. Just the mined location of the spent outpoint you wish to buy (to ensure the correct number of runes are in the transaction, without enabling recursive covenants).

-------------------------

