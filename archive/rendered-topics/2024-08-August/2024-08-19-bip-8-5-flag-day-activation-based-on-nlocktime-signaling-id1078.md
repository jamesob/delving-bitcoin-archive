# BIP 8.5: Flag day activation based on nLockTime signaling

1440000bytes | 2024-08-19 04:25:36 UTC | #1

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

