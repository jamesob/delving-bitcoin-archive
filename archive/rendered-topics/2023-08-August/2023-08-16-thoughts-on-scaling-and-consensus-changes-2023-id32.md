# Thoughts on scaling and consensus changes (2023)

jamesob | 2023-08-16 15:22:13 UTC | #1

Back in June, AJ published an [underread blogpost](https://www.erisian.com.au/wordpress/2023/06/21/putting-the-b-in-btc) about scaling. I’ll do a hasty job of paraphrasing the post --  which you should read in full -- by noting that it hypothesizes that the way that we might scale to **1 billion** users transacting at about **1 tx/week** (which is checking-account volume, my personal target) is by having about 50,000 kinda-sorted-trusted off-chain entities that bear the brunt of most payment volume.

Here I discuss whether I think this is realistic, and if it tells us anything about possible consensus changes.

## Off-chain entities?

If you wanted to be especially coarse you could just say “bitcoin banks.” Examples of what I mean here are

1. a traditional, federated sidechain like Liquid,
    - You send your coins to be controlled by some third party, but in return you get more expressive scripting abilities or confidential assets or faster blocks or some combination of all of them.
    - Drivechains are sort of like this, but the peg-in and -out are mediated by bitcoin miners.
2. some kind of Chaumian ecash bank like fedimint or cashu,
    - You send your coins to a federation (or even an individual operator), and you get some privacy benefits.
3. or some yet-to-be-discovered design that doesn’t hinge on trusting some third party to return your coins ("peg out"), instead relying on enforcement via smart contract (i.e. bitcoin script) and a magic sprinkle of *time-sensitivity* on the part of its users. 
    - This is like a [coinpool](https://coinpool.dev) or something vaguely in the direction of [Ark](https://www.arkpill.me/) (which is itself right now just a vague direction).
    - You could think of this as behaving like the generalization of a lightning channel from 2 parties to `n` parties. It would probably require interactivity across all participants for each payment, but the holy grail would be some design that allows subsets of participants to generate off-chain activity without requiring everyone to sign. Though whether the latter is even possible is an area of ongoing research.
    - For the purposes of this post, this could even be some kind of [ZKP rollup](https://github.com/john-light/validity-rollups/blob/fecc3433d8e95ab40263db8c2941794e0115d2d5/validity_rollups_on_bitcoin.md) thing - after all, “fraud proof” and “penalty transaction” seem basically equivalent. (*Edit: John Light [notes](https://twitter.com/lightcoin/status/1691828900748226860) that validity proofs don't require fraud proofs or liveness.)*
    - We’ll just refer to this thing as “coinpool” for short.

There are other options, like [spacechains](https://github.com/RubenSomsen/spacechains), that I won't cover in detail here. 

The last option, this hypothetical coinpool thing, seems clearly preferable since it’s not relying on a third party. But it’s still a hypothetical, and then even if it weren’t (as we’ve seen with lightning) there are operational caveats around actually maintaining a working, live presence that understands a layer 2 protocol and is able to validate new activity and not get sybiled, etc.

Even relative experts aren’t necessarily able to run networked infrastructure that can lose money if it goes offline. It’s a full-time job, probably for multiple people.

## Layer 1: uncomfortable

Okay, so to get back on track here: in our projected scaling scenario, each user holds value with 3 or 4 of these semi-trusted/time-sensitive offchain services that more or less resemble community banks. These “banks” are probably networked with payment channels (or atomic swappability) so that they can handle cross-bank liquidity operations, like doing the equivalent of wire transfers. I.e. I want to pay you, but we’re at different bitcoin banks, so the entities themselves facilitate some kind of batched settlement over a lightning gateway.

The quiet part out loud here is that by the time 1 billion people want to use bitcoin, the main chain is very expensive to transact on. Note that I say “very expensive” and not “impossibly expensive,” because if users lose the ability to take some form of layer 1 physical custody, bitcoin is just gold with less friction: a paper market will develop and all the nice properties of bitcoin will diminish, as AJ talks about in his post ([link again](https://www.erisian.com.au/wordpress/2023/06/21/putting-the-b-in-btc)).

Note also that part of what keeps these offchain constructions workable is the threat of their depositors somehow interacting with layer 1. Even in the case of some kind of "trustless" coinpool thing, *the ability to appeal to layer 1 is what makes it all work*. So if L1 is too expensive to appeal to for most people, the coinpool design doesn’t really work. Maybe among poorer users, that appeal process happens in groups to amortize base layer fees over a set of users.

These bank things are incentivized to keep operating (and not run away with the money) because (i) they collect fees and (ii?) they may be subject to some kind of local legal jurisdiction. But one of the saving graces here is that the the nature of bitcoin makes it feasible for each customer to also be an auditor.

So let's assume for simplicity that throwing a transaction on layer 1 costs maybe a few hundred bucks: something many could afford in case there’s a Big Problem with their “bank” and they need to flee to layer 1 bitcoin, but not something everyone can afford, and definitely not something that you'd want to do every month or even every year.

## How we scale?

I think this is roughly what scaling to a billion people looks like for Bitcoin: a constellation of 50,000 networked “banks,” with (hopefully!) roughly uniform distribution of BTC managed across each bank, and a hub-and-spoke use of a few of those banks at the individual level.

The alternatives? Can’t just raise the blocksize. Distributing the equivalent of the world’s checking account activity over a network of validating nodes just doesn’t allow you to preserve decentralization.

I truly don’t think that your average person is going to manage a lightning node (directional liquidity and uptime are annoying), and the Phoenix model of a semi-trusted LSP that is by definition a kind of friendly sybil still requires pushing around a UTXO, which isn’t workable at the scale of a billion people. Assuming 41 bytes a UTXO, 1 billion of those things equates to about 380GB just for your UTXO set. This would make timely validation very difficult, although I dunno maybe it's worth some experimentation. [Utreexo](https://bitcoinops.org/en/topics/utreexo/) does help somewhat with this, and would probably be prerequisite to even approaching the problem.

Is the 50,000-”bank” thing  a compromise away from layer 1 ownership? Yes. But UTXO ownership for the entire world -- barring some unforeseen magic, which of course I’m rooting for! -- isn’t in the cards. At least with bitcoin’s technological context, we can have good auditing and custody tools to ensure that fraud is easily detectable and theft is basically impossible.

### Depression pitstop

When reading a draft of this post, my wife got kind of depressed right around here. Her macabre question, basically, is: "if everyone just winds up interacting with sidechains, and not Real Bitcoin, hasn't the whole thing failed? At every point you're dealing with some token that isn't really bitcoin."

I think this is an uncomfortable realization, and I don't enjoy it. I hope the conclusion is wrong, and that we come up with some great cryptographic construct that solves the mismatch between decentralized validatability and volume, but I don't think we should plan for that. 

Even if this is how things pan out, I think it might sound worse than it is. The win over the existing financial regime is that we still get a base-layer money that can't be inflated. A particular bank might attempt some mischief, but bitcoin-native software tools make trust into a finer gradient, and they ensure the mischief will be much more readily evident (Proof of Reserves?) and in certain cases relative to the legacy system, just impossible (DLCs disintermediating brokers? collaborative custody? onchain observability?).



So, assuming you’re willing to entertain these broad strokes, this architectural target gives us a few hints about bitcoin development in the short- to mid-term.

## Design for exit

Because we’re probably going to see a few dominant second layer (“bank”) designs emerge, we have to account for correlated failure. While reviewing this article, [Rijndael](https://twitter.com/rot13maxi) referred to this as "the thundering herd."

Take for example the [lnd fiasco(s)](https://github.com/btcsuite/btcd/issues/1906) within the last year that could have caused mass channel closures amongst lnd nodes. Once a few workable L2 designs come into common use, the probability of a mass exit event goes up, making this seem like a crucial usecase to plan for.

Because layer 1 space is scarce, we need to be able to pack all the exits we can into some timespan of blocks. The way to do this seems to be decoupling the actions “get out” from “actually reclaim your funds” (i.e. create new UTXOs) so that we can facilitate a timely exit for as many participants as possible. This is more or less congestion control, an idea largely pioneered by [Jeremy Rubin](https://rubin.io) during the mid-days of CTV development.

Locking funds for later claim to a prespecified path (or set of paths) should be made as concise as possible so that in times of cataclysm, we can pack in as many exit transactions as possible.

For a case like this, I don’t think you can get more concise than the bare script `<32 byte hash> OP_CHECKTEMPLATEVERIFY`.

It’s worth noting that a P2TR scriptPubKey is the same size (`OP_1 <32 byte pubkey>`), but then actually spending that coin comes with 17vB overhead (= `(34vB (controlblock) + 33vB (CTV script)) / 4`). If you’re trying to fill a block with 2-in-1-out CTV unrolls, this could be about 10% overhead.

The burden of this overhead decreases as you add inputs or outputs of course, so “batch” unrolls for multiple users won’t benefit as much from bare script CTVs (vs. P2TR script paths). But the most common use-case these days would be lightning channel closes, which are 1-in-2-out.

There are rival proposals to CTV that are more flexible, like `OP_TX`/`OP_TXHASH` and others, but in a correlated crisis requiring mass exit from an L2, flexibility isn’t what’s needed - judicious use of layer 1 is. So whether or not `OP_TX` et al. are good for other things (I don't know of any usecases?), I think CTV is needed simply because it is the most concise way to articulate exit from a contract on-chain at a time when blockspace might be precious.

Per some basic analysis, using CTV's spend compression (congestion control), you'd be able to pack in [~33% more lightning closes](https://github.com/jamesob/verystable/blob/18a57207fe4fc106c4befaffc1784b012afc696c/examples/txn_size.py) into a block in times of crisis:

```shell
src/verystable (init 18a5720) % python examples/txn_size.py
CTV txn: 128vB
non-CTV txn: 171vB
CTV txns per block: 31250
non-CTV txns per block: 23391
more per block with CTV: 33.6%
```

## Bomb-proof custody patterns

The other observation that comes to mind is if there are going to be ~50,000 entities managing large sums of bitcoin each, many of them perhaps managed by federated signers, there need to be relatively straightfoward custodial patterns that let us be pretty sure that theft or loss is out of the question.

It shouldn’t come as a surprise that I think the functionality in [`OP_VAULT`](https://github.com/jamesob/bips/blob/e2ff23b3f07215450e75779f7f944d24660a9d47/bip-0345.mediawiki) is an important piece here, because it enables “reactive security.” As Jameson Lopp [says](https://blog.keys.casa/why-bitcoin-needs-covenants/):

>With the right tools, we can be *reactive* rather than only *proactive* with regard to recovering from key compromises.
>
>Vaults allow for the creation of a new set of game theory. Similar to how you can run watchtowers to look out for a Lightning channel counterparty trying to cheat you, you would be able to run a watchtower to make sure no one has compromised your bitcoin vault. If you find that a compromise has occurred, sweeping the funds to safety is simple enough that you can automate it!
>
> It is my sincere belief that every bitcoin self-custody user and every wallet developer should be salivating over the prospect of user-friendly vault functionality. The ability to "claw back" funds that have been lost due to a compromised security architecture means that bitcoiners can sleep more peacefully at night, knowing that they can be fallible, make mistakes, and not have to suffer from catastrophic loss due to a single oversight.

I think that vaults that are enforced on-chain are probably a necessary part of successfully scaling custody to a large number of collaboratively managed pools of bitcoin. And `OP_VAULT` seems like the most chainspace-efficient way to do this. You can maybe emulate `OP_VAULT`'s behavior with, say, [MATT](https://merkle.fun/). But I think the point above about CTV and congestion control tells us that common operations should be made as efficient as possible ([CISC-like](https://www.baeldung.com/cs/risc-vs-cisc)).

Obviously there are a lot of parameters to be decided, e.g.
- who manages a watchtower, and how much do they know?
- what does recovery look like: highly secure/inconvenient keys, social recovery, or some time-decayed composition of both?

and I’m sure others. But I don’t think we can rely on proactive security to get 50,000 entities feeling good about putting a lot of capital on the line in an automated way. Even if you could template out some standard array of HSMs, the supply chain attacks (both from hardware and software) seem too feasible.


## Other stuff

Of course I'm leaving a lot out here. Stuff that isn't related to covenants per se, but is essential to allowing time-sensitive L2 protocols to actually work, like good mempool/fee management policy. Lightning is going to fall down without that work. And stuff that just lets Bitcoin work in a censorship resistant way, like BIP324.

Someone should also work on fleshing out a "minimum viable coinpool," and figure out if we need primitives above and beyond taproot and CTV. That would be valuable info.

## Conclusion

This article is already long enough, and there will probably be subsequent writings from me that either build on this or respond to posts from others using this lens.

I think barring some kind of magical development, which I’m not counting on, the way that we wind up scaling bitcoin to handle checking-account volume is with something on the order of 50,000 entities that manage/risk pools of capital to facilitate payment flows. These entities would be networked, probably with lightning, and will offer a gradation of trust. "Osmotic" pressure from the threat of withdrawal to L1 or opposing financial entities will discipline these things into behaving, almost certainly with some bumps along the way.

As [Hal said in 2010](https://bitcointalk.org/index.php?topic=2500.msg34211#msg34211):

> Actually there is a very good reason for Bitcoin-backed banks to exist, issuing their own digital cash currency, redeemable for bitcoins. Bitcoin itself cannot scale to have every single financial transaction in the world be broadcast to everyone and included in the block chain. There needs to be a secondary level of payment systems which is lighter weight and more efficient. Likewise, the time needed for Bitcoin transactions to finalize will be impractical for medium to large value purchases.
> Bitcoin backed banks will solve these problems. They can work like banks did before nationalization of currency. Different banks can have different policies, some more aggressive, some more conservative. Some would be fractional reserve while others may be 100% Bitcoin backed. Interest rates may vary. Cash from some banks may trade at a discount to that from others.
> 
> George Selgin has worked out the theory of competitive free banking in detail, and he argues that such a system would be stable, inflation resistant and self-regulating.
> 
> I believe this will be the ultimate fate of Bitcoin, to be the "high-powered money" that serves as a reserve currency for banks that issue their own digital cash. Most Bitcoin transactions will occur between banks, to settle net transfers. Bitcoin transactions by private individuals will be as rare as... well, as Bitcoin based purchases are today.

It’s important for those of us building layers beneath these entities to give them the tools to do their job safely and efficiently, and to build them in such a way that their users can retain the ability to audit and move freely.

Along with many other things I’ve missed, this implies 
- making concise and quick exit to L1 possible when necessary (in response to failures that will likely be correlated), and 
- making every day custodial operations safe and simple.

-------------------------

EthnTuttle | 2023-08-22 20:21:05 UTC | #2

It seems that there will be a market that could develop around "L1 insurance". Where a large capital allocator collects on "insurance" and promises you exit support during times of congestion/high fees where an average user cannot afford to exit from one of the aforementioned "bitcoin banks".

-------------------------

Ajian | 2023-09-02 14:34:46 UTC | #3

Nice job! Thank you for your contribution!

Agree with that we need a consise way to exit from bitcoin banks, so CTV is very very useful. But that's not enough for making coinpool (congestion control pool) a useful contrsuction. Assume that I register some value with address A via CTV in a coinpool, when I want to leave this coinpool and go to address B, I will need an extra tx -- it also cost some blockspace.

So, we still need to develop the 'coinpool' concept, to ensure that it is useful.

Obviously, there are two way to improve coinpools' utility. One is embracing APO, to enable internel payment within a coinpool while limiting its complexity. The other is construc committed addresses intentionally, e.g. [swap in potentiam](https://lists.linuxfoundation.org/pipermail/lightning-dev/2023-January/003810.html) (a kind of addresses sharing with LSP trustlessly), to reduce the need to go to another address (thus sending another tx).

I wrote a post to discuss the 'coinpool' concept and the usage of APO and CTV. But it is in Chinese: https://github.com/btc-study/OP_QUESTION/discussions/1

-------------------------

