# BIP 360 + PQ authorization as a native Bitcoin PQ vault lane?

attestor_zero | 2026-04-11 08:30:01 UTC | #1

I’ve been thinking about the destination problem in post-quantum Bitcoin discussions. A lot of current work focuses on migration: how vulnerable coins could be moved under current rules, or how users might buy time if current signature assumptions weaken. That seems useful, but it still leaves an open question:

If the goal is to stay native to Bitcoin, where does the coin actually live afterward?

The direction I’ve been exploring is an opt-in Bitcoin PQ vault lane on mainnet. The idea is that any existing Bitcoin UTXO could enter a native post-quantum vault path, remain there under PQ vault and recovery rules, and later exit back to any native Bitcoin lane.

So this is not a bridge idea or an L2 pitch. It’s a question about whether Bitcoin eventually needs a native post-quantum resting place for value, not just migration techniques.

This seems related to the current BIP 360 direction, but I think it points at a gap. BIP 360 looks useful as structural groundwork for a vault-like output model, but not as a complete PQ destination by itself. My intuition is that a complete version of this probably needs both:

* a vault-oriented output structure, likely something Merkleized

* and a native post-quantum authorization path

A few questions I’d be interested in hearing thoughts on:

1. Is “native PQ vault lane” a useful way to frame the destination side of the problem?

2. If Bitcoin ever adds a PQ authorization path, is vault/recovery the right first use case?

3. Does “enter from any native lane, exit back to any native lane” feel like the right design goal?

I’m asking this as an architecture question first, not a polished final proposal. I've been working through an architectural design for this and would be happy to share more detail if the direction seems worth exploring.

-------------------------

