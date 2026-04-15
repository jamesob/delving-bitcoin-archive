# Commit-Reveal for PQ Migration

JeremyRubin | 2026-04-15 18:35:21 UTC | #1

I wanted to propose a commit-reveal style approach for post-quantum (PQ) migration that may reduce coordination and liveness requirements. I’m not so up to date on all the PQ proposals, but in a brief discussion it seems this is a novel idea with some merit, so I figured I would write it up.


The idea is to allow holders to precommit, within a defined multi-year window, to a future PQ migration transaction without revealing the underlying keys or scheme today.


After the window closes, only coins with valid prior commitments would be eligible for migration via a later reveal step.


The reveal would open the commitment by providing the PQ public key and signature material, proving consistency with the earlier commit.


This avoids requiring immediate proof-of-control while still letting participants secure their coins ahead of time.


It also has the nice property that early or privacy-sensitive holders can prepare for migration without revealing themselves until they choose to. This is really the main motivation for such a proposal.


The design could allow multiple redundant commitments per coin, enabling users to hedge across several PQ schemes and pick the best one later.


That flexibility comes with some tradeoffs, especially in multiparty settings where additional commitments may increase attack surface.


A simple baseline could even rely on Lamport-style signatures as a fallback, with commitments binding to those keys and reveals done via PQ-enabled transactions.


Overall, this seems like a potentially low-coordination path to PQ migration that preserves optionality while enforcing a clear cutoff for valid participation.

-------------------------

