# A Bitcoin Quantum Roadmap: Combining BIP-361 and PACTs to Preserve Ownership Rights

SecondOrder | 2026-11-25 11:32:28 UTC | #1

Hi all. 

I've been following BIP-361 and Dan Robinson's PACT proposal and thinking about how the two could be combined to balance quantum security with preserving legitimate ownership rights.

Proposed quantum roadmap (I'll keep it brief):

1. Introduce quantum-resistant addresses and signatures.
2. Introduce anonymous PACTs.
3. After a generous migration window, freeze vulnerable legacy spending.
4. Keep anonymous PACT registration open beyond the legacy spending freeze, subject to a minimum maturation period (e.g. 365 days from registration) before recovery is permitted.
5. A quantum canary (automatic tripwire) or proactive consensus upgrade permanently closes new PACT registration.
6. Allow indefinite recovery through valid, matured PACTs.
7. Suspend recovery through PACTs that remain immature at the point of registration closure, allowing a subsequent consensus upgrade to establish a final historical eligibility cutoff based on the available evidence.

This aims to:
1. Preserve anonymity by allowing holders to secure ownership without moving their BTC - particularly Satoshi, whose continued existence need not be revealed.
2. Decouple quantum protection from the debate over abandoned BTC and future mining subsidies.
3. Minimise the risk of quantum theft, thereby protecting the integrity of the network.

There are competing interests across the Bitcoin community, but this feels like a reasonable compromise that addresses many of them.

Thoughts and comments?

-------------------------

