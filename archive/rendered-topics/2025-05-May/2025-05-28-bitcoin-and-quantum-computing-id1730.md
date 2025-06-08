# Bitcoin and Quantum Computing

ClaraShk | 2025-05-28 14:25:32 UTC | #1

We’ve just released a new report in collaboration with @deadmanoz on the potential impact of quantum computing on Bitcoin and possible paths forward for the ecosystem.

You can read the full report here: https://chaincode.com/bitcoin-post-quantum.pdf

The main conclusions are below for a quick overview.

We’d love to hear your thoughts!

> 1. **CRQC Timeline Assessment** Experts believe that CRQCs capable of breaking Bitcoin's ECC foundations could first emerge between 2030-2035, aligning with government directives to deprecate vulnerable cryptography by 2030 and disallow it by 2035. This projected timeline provides a crucial window for preparation, given the unpredictable nature of technological breakthroughs, it is essential to account for both the expected trajectory and the possibility of a significantly accelerated timeline.
> 
> 2. **Scope of Vulnerable Funds** Approximately 20-50% of all Bitcoin in circulation (4-10 million BTC) is potentially vulnerable to CRQC attacks. Long-range attacks target inherently vulnerable script types (P2PK, P2MS, P2TR) and addresses with previously exposed public keys (via address reuse), allowing attackers unbounded time to derive private keys from public information already available on the blockchain. Short-range attacks, which affect all Bitcoin script types, exploit the vulnerability window between transaction broadcast and confirmation (or shortly thereafter) when public keys are temporarily exposed, requiring attackers to act within a timeframe of minutes to hours. Address re-use by exchanges and institutions has created a concentration of vulnerable coins in a small number of addresses - high-value targets that would likely be prioritized by quantum attackers. These holdings, however, represent a manageable quantum vulnerability, as owners retain the ability to transfer these funds to quantum-resistant script types when necessary, or can cease the practice of address reuse. This is in contrast to Satoshi-era and inaccessible quantum-vulnerable coins, which are permanently exposed to quantum attack as they cannot be moved by their owners to quantum-resistant script types. 
> 
> 3. **Immediate Protective Measures** High-value Bitcoin holdings represent the most attractive targets for quantum attackers, particularly those of exchanges and institutions where address reuse practices have exposed public keys. While this creates a concentration of easily identifiable, valuable targets, the risk remains manageable through proactive measures. Since owners retain control of the private keys, vulnerable funds can be immediately migrated to somewhat quantum-resistant address types (P2PKH, P2SH, P2WPKH, or P2WSH). Simultaneously eliminating address reuse practices will prevent future exposure to long-range quantum attacks.
> 
> 4. **Considerations for Bitcoin Mining** The quantum threat to Bitcoin mining via Grover’s algorithm appears limited by physical and economic constraints. Quantum miners would face disadvantages including longer computation times, limited parallelization benefits, and substantially higher capital costs. Research indicates that quantum mining would remain economically impractical even with significant advances in quantum hardware, as the theoretical speedup from Grover’s algorithm is insufficient to overcome the efficiency gap and lack of parallelization compared to specialized classical ASICs. This suggests mining security may prove significantly more resilient to quantum advances than transaction signature security. If quantum mining does become viable, however, there’s the potential for correlated fork events if quantum miners adopt aggressive mining strategies, which could lead to attackers with less than half of the network's hash rate being in a position to execute 51% attacks. And if quantum mining becomes the dominant means of mining on the network, the quantum superlinearity problem could drive extreme centralization, concentrating mining power among just a few operators. 
> 
> 5. **Burn vs. Steal Dilemma** Perhaps the most significant challenge is not technical but philosophical: whether to “burn” vulnerable coins or leave them susceptible to being “stolen” by entities with CRQCs. This decision touches on Bitcoin’s fundamental principles regarding property rights, censorship resistance, and immutability. The economic impact of either choice is substantial, with the potential for significant wealth redistribution or effective supply reduction. This is a polarizing issue, with strong opinions held by many on each side of the argument.
> 
> 6. **Migration Pathways** The Bitcoin ecosystem's transition to quantum-resistant scripts faces significant technical and coordination challenges. Proposed migration mechanisms include the conservative commit-delay-reveal protocol that allows users to securely move their funds from nonquantum-resistant outputs to those adhering to a quantum-resistant signature scheme, the more assertive QRAMP protocol that would enforce migration deadlines after which vulnerable UTXOs become unspendable, and the hourglass strategy, which rate-limits vulnerable UTXO spending. Successful migration necessitates unprecedented collective action by all ecosystem 45participants - individual users, institutions, exchanges, and miners - with extensive preparation including education campaigns, migration tools, and regulatory engagement and compliance. The complexity of this transition demands establishing a shared vision and clear communication channels well before quantum threats materialize, as even the best technical solution will fail without effective cooperation among Bitcoin's diverse stakeholders. 
> 
> 7. **Strategy for Action** We propose that Bitcoin's quantum resistance strategy for action adopts a dual-track approach: contingency measures delivering minimal but functional protection against CRQCs completed in ~2 years, and a comprehensive path allowing thorough exploration of the problem space and the development of a full-featured approach to take ~7 years. This dualtrack strategy balances immediate security needs with rigorous research and development of optimal quantum-resistant solutions, ensuring Bitcoin can respond appropriately regardless of how CRQC capabilities evolve. 
> 
> 8. **Ongoing Efforts & Future Directions** Several technical approaches have emerged to address the potential for a CRQC to derive private keys and forge signatures. Each approach is of varying maturity, and there’s currently no consensus on which direction to take. All current approaches also propose using PQC schemes that have combined public key and signature sizes that are many times larger than the combined size of existing ECC-based public keys and signatures. Given the strong focus on post-quantum cryptography within the broader cryptographic community, continued advancements are likely over time, offering the potential for more refined solutions as the f ield progresses. Several leading cryptographers and Bitcoin developers who have contributed significantly to Bitcoin have begun working on quantum readiness strategies, joined by a number of new and enthusiastic contributors. While there's a vast solution space to explore, and the path forward remains uncertain, the community's ongoing efforts as outlined in this report should inspire confidence that Bitcoin will adapt to the post-quantum landscape in time. These efforts aim not only to meet projected timelines, but also to ensure readiness in the event of a sudden and significant leap in quantum computing capabilities.

-------------------------

AdamISZ | 2025-05-29 13:02:49 UTC | #2

Still reading. Just writing to note a typo on page 30, Jan 2025 not Jan 2015.

But I must say, what an extremely thorough piece! I'm doubtful about the timelines at the beginning but there's a lot more detail here than I would have expected, e.g. you actually discuss the BIP32 implications which shows attention to detail.

> While there’s a vast solution space to explore, and the path forward remains uncertain, the community’s ongoing efforts as outlined in this report should inspire confidence that Bitcoin will adapt to the post-quantum landscape..

Probably just being a curmudgeon here, but this doc does not at all inspire confidence, but only remind me that this situation is going to be nearly impossible to resolve cleanly, I just hope (and, actually, think) that it's not a realistic threat in the next 10 years despite the fact that I know *some* experts believe it is.

Fusion was 10 years away in 1985. I'm old :smile:

-------------------------

AdamISZ | 2025-05-29 14:01:00 UTC | #3

About this section of the Conclusion:

> We propose that Bitcoin's quantum resistance strategy for action adopts a dual-track
approach: contingency measures delivering minimal but functional protection against CRQCs
completed in ~2 years, and a comprehensive path allowing thorough exploration of the
problem space and the development of a full-featured approach to take ~7 years

So based on Section 8, I think I understood what you mean by "contingency measures here"; I *think* you mean deployment of any/some PQC based script type, i.e. making it available for use. Am I right? While I see the logic in putting that as a "first step", somehow it feels like the wrong way round; that part is the most difficult to come to a decision on, even if it would be the most useful to have upfront.

My gut feeling was also that we need 2 phases, but I had a different thing in mind - that the first phase would be (a) not a protocol, but a community step: migration to non-reused addresses so as to be in the "only short-term attack vulnerable" portion (also not sharing xpubs which should never happen anyway!), and using script-path taproot where possible, and then (b) a deployment of some CDR protocol variant, *even in the absence of consensus on a PQC sig scheme*. Then over a longer period, actual PQC schemes could be deployed.

I realize that seems illogical, maybe it is, but I am more thinking about what could actually happen, than what I think should happen in an ideal world. Maybe I'm wrong but I expect consensus on adding PQC to Bitcoin is going to be *extremely* slow - unless the threat becomes very obvious indeed.

-------------------------

ClaraShk | 2025-05-29 14:38:02 UTC | #4

Thanks for your comments and pointing out the typo!

As for our estimate, I completely understand your skepticism. There is some work happening that makes me optimistic, and I believe that if and when quantum computing becomes more realistic, the immediate focus will be on finding an initial, practical solution. Time will tell.

From your 2 phases, we are definitely planning to push for the first one, as address re-use is a bad practice in general, and there are immediate solutions for the short-range attack. If this document motivates people to be more careful, it will already be a win.

-------------------------

AntoineP | 2025-06-02 14:48:43 UTC | #5

Thanks for the great report. Good to see the threat to PoW addressed. The argument that it is (much) further away than breaking the ECDLP is convincing. However it seems the leap from breaking the ECDLP to breaking PoW could be smaller than going from nothing to breaking the ECDLP? It also appears to be more of an existential threat to Bitcoin (even as a concept, not today's network specifically).

Some nits i collected as i was reading through.

In the table on page 18 you discuss the resource usage associated with various schemes but two of them have the same name+date identifier:
![image|690x52](upload://7DIo08oI4DmE5dP60o7P1DMfWbI.png)

On page 30 you discuss the amount of BTC behind revealed public keys:
![image|690x103](upload://2d3BgOdMkkqnRuRRyoWwmjAsoI.png)

Did you mean "mid January 2025" instead of "mid January 2015"?

On page 35 you discuss private submission of CRQC-vulnerable transactions to trusted miners:
![image|690x96](upload://eSGv1bGwzK7uhE6Iov9i7nBW6oj.png)
![image|690x44](upload://vn9CAK1EOb9UacqgcVyjybGzrG3.png)

I think it's worth mentioning in the footnote this approach also trusts the attacker wouldn't reorg the last block(s) to steal the funds. For vulnerable transactions spending large UTxOs in a post-2032-halving world this does not seem unlikely at all.

-------------------------

ClaraShk | 2025-06-02 15:03:20 UTC | #6

[quote="AntoineP, post:5, topic:1730"]
However it seems the leap from breaking the ECDLP to breaking PoW could be smaller than going from nothing to breaking the ECDLP?
[/quote]

That depends on where we set the baseline for “nothing.” But if we start the clock today, or even a decade ago, the technical challenges involved in outperforming ASICs still seem substantial. That being said, history has shown that unexpected technological leaps are always possible.


[quote="AntoineP, post:5, topic:1730"]
It also appears to be more of an existential threat to Bitcoin (even as a concept, not today’s network specifically).
[/quote]

Definitely, and ideally we’ll have a solution well before quantum mining becomes a practical concern. It’s an important research direction, and I hope it gains more attention soon.


[quote="AntoineP, post:5, topic:1730"]
Some nits i collected as i was reading through.
[/quote]

Thanks for these!

-------------------------

