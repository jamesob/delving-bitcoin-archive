# Disabling min-difficulty blocks on testnet4

batman | 2026-03-22 13:19:24 UTC | #1

## Problem

Testnet4’s 20-minute min-difficulty rule allows any block to be mined at difficulty 1 if 20 minutes have passed since the last block.

CPU miners are taking advantage of this min-difficulty rule, and there is very intense competition over who gets to broadcast the min-difficulty block faster. So intense that some miners send empty blocks to gain a slight advantage on bandwidth and verification time. The result is a network of miners where most blocks do not confirm any transactions. This defeats the purpose of a test network.

Having "free mining" via min-difficulty blocks seemed like a good idea, but it has led to unintended consequences: the competition to exploit the rule makes the network less usable, not more. Having the illusion of free mining without the usability of a test network is far worse than having a usable test network where coins simply aren't free to mine.


### Proposal

Disable the min-difficulty block rule on testnet4 via a hard fork at block height **201,600** (the epoch 100 boundary, chosen for clean alignment with the 2016-block difficulty adjustment interval and to give node operators sufficient time to upgrade). Testnet will reach this block height at around between August and September of 2027.

After this height:

\- The 20-minute min-difficulty exception no longer applies

\- `GetNextWorkRequired()` and `PermittedDifficultyTransition()` enforce standard difficulty rules

\- Normal retargeting handles the transition naturally

### Why no difficulty reset?

An earlier version of this proposal included resetting difficulty to an arbitrary value (e.g. 1,000,000) at the fork height. After discussion, this was dropped in favor of simply disabling the min-difficulty rule and letting retargeting adjust on its own.

### Links

PR: https://github.com/bitcoin/bitcoin/pull/34420

Mailing list discussion: https://groups.google.com/g/bitcoindev/c/Jsv1VYpewuU

Bitcointalk: https://bitcointalk.org/index.php?topic=5569103.msg66199834#msg66199834

### Open questions

1\. Is height 201,600 (epoch 100) the right fork height, or should it be adjusted?

2\. Does this change warrant a BIP, given that it modifies testnet4 consensus rules?

3\. Are there concerns about the temporary \~1-hour block intervals during the difficulty normalization period?

-------------------------

garlonicon | 2026-03-23 08:05:08 UTC | #2

> Is height 201,600 (epoch 100) the right fork height, or should it be adjusted?

It is just some arbitrary value, as usual. It doesn’t matter that much, which number would you pick. What matters more, is how many users, miners and developers you will have on your side.

Currently, there is no deployed client with your version, which means 0% adoption. If some people will start using your version, and start producing some blocks, then it is a matter of importing it later into Bitcoin Core.

Because initially, testnet4 produced around 40k blocks, before it was included into Bitcoin Core. But well, mempool.space adopted it, some users started using it, some mining pools put some real hashrate into that, and that’s how testnet4 was created. And later, after all of these blocks, it was simply included into Bitcoin Core.

So, in your case, it could be similar. Pick some number, get some support on that, and then, the only question would be “is it better than unmodified testnet4?”. Because now, you are on “where can I see it deployed?” stage.

I assume if it is a hard-fork, then just deploy something, and use it. If it will be better than what we have now, then it will be accepted. And if it won’t be accepted, then still: you, as a user, will benefit from using a better testnet, if your network will be better than testnet4.

By the way: you didn’t have to change things from block 151,200 to 201,600. If you are waiting for some centralized approval, then you are doing it wrong. Just deploy something, see, how it goes, and ask for including it, when it will be up and running.

> Does this change warrant a BIP, given that it modifies testnet4 consensus rules?

Well, it brings back the old rules from the mainnet, so maybe not necessarily. Also because your network is roughly equivalent to running signet with OP_TRUE as a signet challenge, and with 201,600 blocks of premine.

> Are there concerns about the temporary \~1-hour block intervals during the difficulty normalization period?

Currently, ASIC blocks are the ones confirming any transactions, made by users. So, it wouldn’t be worse, if CPU blocks would be eliminated. It would only affect mining, not usage.

-------------------------

