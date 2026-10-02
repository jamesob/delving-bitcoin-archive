# Is the pre-finality double-spend race intrinsic to single-owner coins?

Hal_03 | 2026-10-02 08:56:37 UTC | #1

I've been trying to solve the pre-finality race problem: Alice (payer) can pre-broadcast a spend of her own coin an instant before paying Bob (payee), so her spend anchors first and Bob's payment becomes a double-spend.

To be concrete, the coin is a single-owner output. A payment spends the payer's note, revealing its nullifier, and creates a change output plus an output to the payee. Revealing the same nullifier twice is a double-spend. At every instant, each unit of value is owned by exactly one party; there is no shared or locked UTXO.

I'd like to avoid the 2-of-2 channel, for two reasons:

- **Lightning's punitive model** doesn't fit a consumer product. If a user loses their phone, they must be able to recover from a backup, and if that backup is a slightly old state, broadcasting it gets them punished, they can forfeit their whole balance. The proper fix for that (non-punitive recovery) leads to eltoo, which needs BIP-118, not yet activated.

- **eltoo** has the semantics I want (latest-state-wins, no punishment), but it needs SIGHASH_ANYPREVOUT , and I'd like a solution with no soft fork. It also still wants a watchtower for offline parties, the component I'm trying to eliminate.

The underlying motivation: I want people's money to stay in their own exclusive custody, no locked UTXO, no co-owned (2-of-2) funding, changing hands only at the moment of payment. 
So far, all the potential solutions I have tried to implement have merely served to mitigate the problem to varying degrees, by adjusting the time window available for Alice to act, without actually resolving it.

So my question is this: is the pre-finality race an intrinsic property of single-owner coins, or is there a construction I'm missing that avoids both the 2-of-2 and the race? 
Furthermore, even though there is currently no possible solution, since the problem is intrinsic, is anyone working on finding a solution similar to what I am trying to achieve?

-------------------------

