# Quantum Attack Game Theory

statoshi | 2026-05-21 12:12:56 UTC | #1

The reactions to BIP-361 revealed that many believe the appearance of a quantum computer would only result in temporary market volatility.

Unfortunately, it’s not that simple. This is my attempt to comprehensively catalog what could happen. Feedback is appreciated!

https://blog.lopp.net/quantum-attack-game-theory/

-------------------------

micah541 | 2026-05-24 22:52:28 UTC | #2

Curious about the math - what's the standard approach to getting this formula? 

![image|690x98](upload://aBSCyj7UswsSs7Yb78WzhKjsTs2.png)

Also, for 

![image|690x69](upload://uuPJdGuyHkP6onSg4AGStrWL9Ro.png)

doesn't this discount the blocks that are found if you win?  The opportunity cost of chasing a failed attempt to reorg involves lost block rewards if you don't win - but you don't pay that cost if you do win.

-------------------------

statoshi | 2026-06-05 11:48:05 UTC | #3

This is by far the most complex part of my post because I don't think there is a "standard" way of calculating the Expected Value in this scenario. The Bitcoin Whitepaper does have a formula for calculating the ability for an miner with a given hashrate to be able to reorganize the blockchain after X blocks, for which I even wrote a [web app tool](https://jlopp.github.io/bitcoin-confirmation-risk-calculator/) a few years ago.

I started off with that formula and then concluded it was a poor fit for this scenario because the "race" is happening in parallel with the rest of the network.

So I ended up writing a [different tool ](https://jlopp.github.io/bitcoin-miner-fee-snipe-calculator/)to try to model this fee sniping scenario.

It looks like you're focusing on the "initial decision" for mining privately which means that there's effectively no additional advantage in terms of block subsidy if your privately mined chain wins vs if you just mined on the tip of the chain. Future "step" decisions as the length of your privately mined chain increases will then add "B\*R" back into the expected value and thus the decision whether or not to continue mining your private chain fork.

-------------------------

