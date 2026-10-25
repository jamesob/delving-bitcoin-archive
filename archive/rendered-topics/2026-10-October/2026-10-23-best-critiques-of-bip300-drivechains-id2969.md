# Best critiques of BIP300/drivechains?

andrewclicks | 2026-10-23 08:05:14 UTC | #1

I'm doing my best to understand the upcoming eCash fork, what are some of the best written critiques you have found, whether about the fork itself, BIP300 (drivechains), or blind merged mining?

One of the better sources of discussion I have found is on stacker.news (there are a few threads there) https://stacker.news/items/1527224 but I would really rather read some well thought out critiques instead of back-and-forth between people.

-------------------------

pinheadmz | 2026-10-24 10:10:14 UTC | #2

https://petertodd.org/2023/drivechains

-------------------------

garlonicon | 2026-10-25 13:20:20 UTC | #3

> no amount of Bitcoin hash power can fake a digital signature

This is mathematically false, and already deployed Proof of Work puzzles can actually measure in practice, how much Proof of Work you need, to break a given signature. Currently, moving coins from [bc1qts0jh2d2nesmketmw3thedwrx939k2tqu04gy90x9hd4049g4uhs83ltjx](https://mempool.space/address/bc1qts0jh2d2nesmketmw3thedwrx939k2tqu04gy90x9hd4049g4uhs83ltjx) means around 8 days of GPU grinding, for a DER signature below 54 bytes. The current decentralized sidechain, running at [bc1qn9vp8l5rs7huyl237s4q9lhrzcs0mzaajt528ysq3wgnzvlkay5sdfz6am](https://mempool.space/address/bc1qn9vp8l5rs7huyl237s4q9lhrzcs0mzaajt528ysq3wgnzvlkay5sdfz6am) is still not pegged-out, even though the private key is known, and equal to one, because nobody made a DER signature below 53 bytes yet.

> Of course, if an output can be spent by any miner, surely it will just be stolen, right? Probably!

Only if coins are signed in the default way, where miners can just take them all. However, it is possible to do it differently: sign only inputs and outputs you care about, and leave everything else to the miners. Then, all they can do, is just collect your fees. For example: here you have [a coin using SIGHASH_NONE](https://mempool.space/tx/8349df0753e80cce322322f1b76789e1d0fd6693aed2f4de4e49576423081ae7#vin=0), where all coins are just deposited to the sidechain miner as fees. However, if it would use just SIGHASH_SINGLE, then it would simply send a given amount to a given address, and only fees would be deposited to the sidechain puzzle.

> BIP-300 proposes much longer timescales of approximately 3 to 6 months.

Only because short timescale was heavily criticized, so they increased it in response. But the original sidechain proposal also contained a quite short timescale, measured in single days.

> A very notable omission in BIP-301 is that BIP-301 does not interact with BIP-300 Hashrate Escrows.

Well, currently we have BIP-300-equivalent, without BIP-301. Which means, it is possible to lock coins to some Proof of Work inside Script, but there is no Merged Mining. Which means, that instead of 80-byte block headers, sidechain miners actually hash Segwit messages, for example this one: https://bitcointalk.org/index.php?topic=5551080.msg65655736#msg65655736

```
MSG = 020000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001280bc5682e0cbe78108912437ab583daa49a8fe66e995001a22a44aec2a3ab06000000288201369f69210279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798acf0d20000000000003d74fbff045b0f32c7adb53ce93b424e7ba280a4328ef880b60058c771b3ef599f9e95840000000081000000
Z = ffffffffffffffedfe243987a5dfa917a1bba0978fd5ce30e6df1b23cf1d8fc4
S = 000000000000002403b78c79c2a4014f1ea49d7614fa4fc457cb59250f722a34
```

So, it works here and now, and is up and running on mainnet, since at least July 2025. The main missing part is connecting it with actually mined 80-byte block headers, because now, all sidechain miners have to decide, if they want to mine BTC blocks, or if they want to grab some satoshis from sidechain puzzles like that.

Also, you never actually know, how many coins are deposited, if you are not connected to the actual sidechain network. Anyone could see 54k satoshis, sitting on [bc1qts0jh2d2nesmketmw3thedwrx939k2tqu04gy90x9hd4049g4uhs83ltjx](https://mempool.space/address/bc1qts0jh2d2nesmketmw3thedwrx939k2tqu04gy90x9hd4049g4uhs83ltjx), but the actual reward was bumped in practice by 200k satoshis. It was easy to spot, because [it was publicly announced on bitcointalk](https://bitcointalk.org/index.php?topic=5551080.msg65619881#msg65619881). However, it doesn't have to be the case for next sidechains.

> Obviously, if drivechains supporting hundreds or thousands of transactions per second catch on, miners will need to invest in thousands or tens of thousands of dollars worth of computing hardware and bandwidth to mine.

It is true, if there is a single layer, and if you have to always synchronize the whole history, starting from 2009. However, when second layer transactions are settled on the first layer, then that history can be discarded by the network, and kept only between interested parties. This is the same case, as Lightning Network transactions: no full node needs to verify the whole lightning history, only peg-ins and peg-outs are actually checked. The rest is stored only by users, or not stored at all, if nobody needs it.

If Bitcoin will want to scale, then, sooner or later, it will be needed to prune some history in that way, or another. If users from 2109 will still need to go back to 2009, and verify everything, just to sweep some coins, received in 2108, then it simply won't scale to billions of users. When there is any upper layer to commit to, then things can be discarded in a safe way, without affecting Initial Blockchain Download. But if there is no upper layer, then people are forced to always verify everything, even if it is not scalable.

-------------------------

