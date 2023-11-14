# Fedimint Overview and Fedipool Theorizing

EthnTuttle | 2023-09-29 22:50:14 UTC | #1

As I've spent time in the [fedimint](https://github.com/fedimint) and technical Bitcoin space (not very long), there seems to be a lot of questions about fedimint, and some misconceptions. I hope to unpack some of the comments I've encountered and clarify anything I found to be erroneous. Admittedly, I have [more understanding and bias toward Fedimint](https://opensats.org/blog/bitcoin-and-nostr-grants-august-2023#fedimint-modules-and-resources) so I might miss the mark on other Bitcoin-related technologies, especially mining (out of principle, I run an under-clocked S9 at 500w). 

# Intro

I began looking into fedipool as a concept after reading [this blog post.](https://www.discreetlog.com/fedipool/) I read some more about [Stratum v2 (Sv2)](https://github.com/stratum-mining) and came to the conclusion that Sv2, along with the decentralization of mining, might be the most important aspects of Bitcoin to work on. (Perhaps in another thread I can elaborate on this for debate.) This, coupled with some [other reasons](https://yakihonne.com/article/naddr1qqxnzd3exvenzvpj8q6rjd3jqgs9ajvxw0lwe25ntjq52qtfygf9ngh3xf35u7y35rzn0h9k8meulvgrqsqqqa283uvvyw), encouraged me to begin working toward a goal, implementing fedipool. 

In this post, I'll first cover fedimint and hopefully clarify some misconceptions. Then I'll cover a basic (probably naive) implementation that I think could enable fedipools ASAP. Finally, I'll discuss a more involved implementation outline using a fedimint module. Hopefully this can serve as an introductory and exploratory post for those considering fedimint or those interested in fedipools.

## fedimint

According to the [protocol website](https://fedimint.org/docs/intro)

> Fedimint is a modular open source protocol to custody and transact bitcoin in a community context, built on a strong foundation of privacy.

I see the following as key features:

**Privacy** - [Chaumian ecash](https://en.wikipedia.org/wiki/Ecash) (the data is the asset)

Probably the most well known aspect of fedimint is the privacy trade off for the federated custody. Every ecash note is 1:1 with custodial bitcoin by the fedimint nodes. The most common critique of fedimint also stems from this 1:1 bitcoin:ecash since [ecash proof of liabilities](https://gist.github.com/callebtc/ed5228d1d8cbaade0104db5d1cf63939) is not a solved problem.

**Offline** - It's data (transfer it without the internet)

This can make it ideal for places with poor connectivity (conferences/large gatherings or the developing world). This presents a trust race condition though, if the sender were malicious, they could race to get online faster than the receiver, and redeem the "sent" ecash thus causing any attempts by the receiver to redeem the ecash notes to fail.

**Modular** - Default modules are wallet, mint, and LN

Different modules can be designed using a core primitive called `Transaction` that has a set of inputs, a set of outputs, and a signature. These are then signed into consensus by the nodes of the fedimint. However, a module does not need to use a `Transaction`, it can just store arbitrary data as a `ConsensusItem` in the fedimint. A simple example of this is a password manager. A more complex example would be stability pools (stable fiat price for users doing finance magic I don't understand yet). I believe fedimint modules can become an interoperability point for lots of other tech and Bitcoin.

**You don't have to do the LN thing** - Best feature, imho

A fedimint node runner, does not have to also run a Bitcoin node or a LN node. They can just run `fedimintd` and connect to some other Bitcoin node of their choice with all the standard trade offs. A LN node can join a fedimint as a special client called a `gateway`. They can then offer their liquidity to the fedimint users. Ecash can then be swapped for LN based sats, and vice versa, using the module system. This means a super awesome LN node operator can do what they do best and the fedimint node runner doesn't have to manage the challenges of a LN node. Currently CLN and LND are supported and both require the running of `gatewayd` to connect/interact with the fedimint.

There are probably some more features, but these are the ones top of mind to myself.

## Naive Approach

A mining pool needs a coinbase address to put into a block. The pool just joins a fedimint and gets a deposit address for each block they mine. After the block confirms beyond the fedimint's desired block count, the pool can then redeem ecash from the fedimint. Using the same accounting for shares and payout, the pool sends ecash to the miners. The miners can then go to the fedimint at their leisure and swap for on-chain or LN sats. The pool operator is not managing a fedimint node, nor a LN node. The trust remains relatively the same between the miners and pool. You're trusting the pool to not rug you and/or double spend you. However, this would require the miners to form another relationship with a new entity: the fedimint and its node runners.

## `poolimint` module (all modules should be `-imint`)

There are some ideas discussed in [this GitHub Discussion](https://github.com/fedimint/fedimint/discussions/1504) but I hope to explore some more thoughts and pose some questions I have.

First, some observation I have made/learned (please correct if wrong):

- latency is a big deal in mining
- Sv2 enables anyone to generate the block template
- LN doesn't work for pools because liquidity challenges
- small miners want faster/smaller payouts (me!)

Since latency for block generation is a critical aspect of pool technology, I think it best that fedipools follow the Sv2 model and delegate that to where it makes sense. Perhaps that is as part of fedimint consensus, but perhaps not. I lean toward the latter so do not consider this approach.

Pool shares do not seem like something that would be latency sensitive. Validating mining shares using fedimint consensus now decentralizes the validation of these shares to multiple entities. All fedimint nodes would have to agree that a share is valid in order for it to be considered in the pay out accounting. Only shares with the correct coinbase address would be considered valid. There are other criteria to evaluate shares on but I am ignorant of them so will avoid incorrect assumptions. 

Which piece(s) of a mining share is(are) critical to validate? What else might a pool care to have validated by a quorum?

Once a block is found and confirmed beyond the required block count, ecash can be made ready for immediate redemption in any amount. This increases speed, privacy, and optionality of the miners being paid out. This does not prohibit on-chain payments for those miners so inclined. A LN gateway can also provide liquidity to those that desire payment via the lightning network. 

Are there other solutions offering this optionality to miners?

A pool can decentralize custodial risk of the funds held on behalf of the miners, as well as reduce the amount of on-chain activity that may be needed. This means cost savings for an industry with razor thin margins and cut-throat dynamics. Additionally, a pool operator will only hold one key of a threshold needed to move funds so it introduces some interesting questions around who technically has custody of the miners’ funds awaiting withdrawal from the pool. All of this while still maintaining a similar trust trade-off that miners and pools already have. 

Are these trade offs appealing to pool operators and/or miners?

There must be some way for a pool to differentiate miners from each other, to my understanding this is done with a `user:password` combination. Since a fedipool would also have to understand this differentiation, we need a privacy preserving mechanism of claiming ecash from shares of a certain "account". This makes me consider [nostr](https://nostr.com/) as a solution, since anyone can generate a nsec/npub key pair or any other cryptographic key pair, and then used derived secrets as their mining pool "account". The poolimint modules could then only redeem ecash for someone proving ownership of the correct keys. 

Are there better ways to safe guard privacy of miners? 

--------

I look forward to your comments and questions.

-------------------------

MattCorallo | 2023-11-14 00:45:48 UTC | #2

Federating the hot wallet of a mining pool isn't really an interesting problem to solve - pools could just pay out over lightning today every second and the float would be ~zero. The fact that we don't see this used broadly today is probably a great indication there isn't much demand for this :p.

More generally, there are lots of difficult issues with mining centralization, but the float the pool owes the user isn't really generally high on that list. I spent some time discussing this at https://github.com/fedimint/fedimint/discussions/1504#discussioncomment-5020472 and elsewhere in that thread.

-------------------------

