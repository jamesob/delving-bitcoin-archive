# Fork withholding attack

Jassu7082 | 2024-09-12 05:08:09 UTC | #1

Hello everyone, I'm Jaswanth, a computer science student currently working on a project aimed at improving reward mechanics in mining pools to defend against Fork After Withholding (FAW) attacks.

Block withholding is a type of attack similar to selfish mining, where a pool member withholds a mined block to undermine the pool's revenue while submitting shares consisting of PPoWs rather than FPoWs, creating the miner's dilemma. In an FAW attack, an attacker joins a Bitcoin mining pool and submits FPoWs when a non-target miner generates a block. If the pool manager accepts the FPoW, it results in a blockchain fork, potentially selecting the attacker's block. This causes the target pool to receive rewards, shared with the attacker. The FAW attack can yield rewards up to four times greater than the BWH attack in large pools

Countermeasures:

The idea is to have the pool manager track each user's reputation by monitoring metrics like Proof of Partial Work (PPOW). If a block submitted by a user leads to a fork, their reputation would decrease accordingly.I have a solid theoretical understanding of blockchain but lack practical experience. I'd appreciate any guidance on how to implement this in mining pools. Are there any simulation tools or resources that could help me get started? 
ref:https://dl.acm.org/doi/abs/10.1145/3133956.3134019
Thanks in advance!

-------------------------

mcelrath | 2024-09-13 13:57:18 UTC | #2

Very welcome to have you working on this! What is a "Proof of Partial Work"? I can't find reference to it anywhere except this post.

[I jotted down some thoughts on this here,](https://github.com/braidpool/braidpool/discussions/70) but I don't see any really great solutions. I'd welcome some new ideas.

There is also a new [bitcoin mining mailing list](https://groups.google.com/g/bitcoinminingdev?pli=1) that you may be interested in joining.

-------------------------

Jassu7082 | 2024-09-13 14:58:08 UTC | #3

Proof of Partial Work (PoPW) refers to the concept of shares contributed by miners in a mining pool. In the context of mining pools, shares are submitted by miners to demonstrate the work they've done, even if that work hasn't yet resulted in finding a block. These shares are essentially proof of partial work, as they show that the miner has contributed effort

-------------------------

