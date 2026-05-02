# Commit-Reveal for PQ Migration

JeremyRubin | 2026-05-02 17:17:57 UTC | #1


I wanted to propose a commit-reveal style approach for post-quantum (PQ) migration that may reduce coordination and liveness requirements. edit: ~~I’m not so up to date on all the PQ proposals, but in a brief discussion it seems this is a novel idea with some merit, so I figured I would write it up.~~ `There seems to be a history rich with similar proposals, feel free to include more in the comments.`

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

reardencode | 2026-04-26 00:04:06 UTC | #2

How would someone prove the age of their pre-commitment? OTS or something?

-------------------------

JeremyRubin | 2026-05-01 18:35:29 UTC | #3

https://www.paradigm.xyz/2026/05/pacts-protecting-your-bitcoin-from-a-quantum-sunset >> seems like a similar idea; perhaps a bit more fleshed out.

-------------------------

JeremyRubin | 2026-05-02 17:07:48 UTC | #4

collecting some similar proposals:


https://gnusha.org/pi/bitcoindev/1518710367.3550.111.camel@mmci.uni-saarland.de/


https://arxiv.org/pdf/2303.06754


https://groups.google.com/g/bitcoindev/c/LpWOcXMcvk8/m/DjaiWnViAQAJ

h/t @jonasnick for the literature

-------------------------

JeremyRubin | 2026-05-02 17:14:48 UTC | #5

https://gist.github.com/phyro/64f99a4b26b26e69a4092fc434b62e2f

-------------------------

