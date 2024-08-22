# BIP Draft: Flag day activation based on nLockTime signaling

1440000bytes | 2024-08-20 07:02:31 UTC | #1

![image|660x500](upload://qWeXTVRJExP7x9TYrAVDRc3OFav.jpeg)

    BIP: XXX  
    Layer: Consensus (soft fork)  
    Title: nLockTime signaling and flag day activation
    Author: /dev/fd0 <alicexbt@protonmail.com>  
    Status: Draft  
    Type: Standards Track  
    Created: 2024-08-19  
    License: Public Domain

## Abstract

This document describes a process to activate soft forks using flag day after `nLockTime` signaling and discussion.

## Motivation

BIP 8 and BIP 9 are controversial. This BIP is an alternative which addresses the problems with other activation methods.

## Specification

- Assign numbers to different soft fork proposals or use their BIP numbers
- Users can broadcast their transactions with one of these numbers used as `nLockTime` to show support
- Miners inlcuding a transaction in block would signal readiness for a soft fork
- Community can analyze these transactions after 3 months and prepare for a flag day activation of soft fork

Example:

Use 119 to signal support for OP_CHECKTEMPLATEVERIFY in `nLockTime`

## Reference implementation

Activation: https://github.com/bitcoin/bitcoin/commit/ab91bf39b7c11e9c86bb2043c24f0f377f1cf514.diff

Exclusion in relay or mining: https://github.com/bitcoinknots/bitcoin/commit/18cd7b0ef6eaeacd06678c6d192b6cacc9d7eee5.diff

---

If a transaction does not get included in block for a long time, users can replace it with another transaction spending same inputs and use a different `nLockTime`.

-------------------------

1440000bytes | 2024-08-20 01:12:44 UTC | #2

Mailing list thread: https://groups.google.com/g/bitcoindev/c/-NbdxKkaMQU

-------------------------

ajtowns | 2024-08-20 07:03:21 UTC | #3

Removed self-assigned BIP number from title

-------------------------

sjors | 2024-08-20 16:21:45 UTC | #4

For some reason only a fraction of bitcoin-dev mailinglist posts make it into my mailbox. So although there's already discussion there, I'll reply here. One argument I didn't see in @fjahr's list there is this:

Transactions with `nLockTime` values in the past (or a lower height) are mined and relayed by default. A miner opposed to a change would now have an incentive to censor the transaction. And users who don't like the change may try to censor relay (as some are trying with inscriptions).

We should not creative incentives for better censorship tooling.

> * Miners including a transaction in block would signal readiness for a soft fork

I'm not sure who you're referring to with "readiness".

If you mean that miners are signalling readiness, then I don't think we should encourage the idea that including a transaction is an endorsement of it.

If on the other hand users are signalling readiness, then inclusion in a block is an unreliable metric since miners could be manipulating that in either direction.

-------------------------

1440000bytes | 2024-08-21 12:13:03 UTC | #5

(post deleted by author)

-------------------------

sjors | 2024-08-22 08:09:36 UTC | #6

(post deleted by author)

-------------------------

1440000bytes | 2024-08-21 12:12:41 UTC | #7

(post deleted by author)

-------------------------

1440000bytes | 2024-08-22 00:35:17 UTC | #8

[quote="sjors, post:4, topic:1078"]
I’m not sure who you’re referring to with “readiness”.

If you mean that miners are signalling readiness, then I don’t think we should encourage the idea that including a transaction is an endorsement of it.
[/quote]

Miners do not endorse any soft forks with signaling. Neither this BIP nor BIP 8/9 signaling is meant for endorsements.

[quote="sjors, post:4, topic:1078"]
If on the other hand users are signalling readiness, then inclusion in a block is an unreliable metric since miners could be manipulating that in either direction.
[/quote]

Users are signaling support as mentioned in the BIP draft. All transactions will be mined sooner or later if there is at least one pool which can include it in the block.

[quote="sjors, post:4, topic:1078"]
We should not creative incentives for better censorship tooling.
[/quote]

Ocean pool (default template) already excludes transactions with nLockTime 21. This BIP does not create incentives for censorship. All transactions will eventually get mined except if all miners decide to exclude it. In case, all miners ignore a transaction, users can use a normal nLockTime to replace the original transaction or increase fees.

-------------------------

