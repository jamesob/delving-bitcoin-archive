# Pragmatic definition of consensus for light clients

Nuh | 2026-08-26 13:27:47 UTC | #1

# A Pragmatic Definition of Consensus (for Economic Actors)

I've been trying to articulate, primarily to myself, a practical, pragmatic, and realistic definition of *consensus*, viewed strictly from the perspective of economic actors rather than theoretical correctness. The question I've found most valuable to ask is: *What works in practice, but not in theory?* Below, I'll share my current thinking.

## Core Premises

Looking back, my chain of thought breaks down into four core premises:

1. **Value drives follow-through.**  
   An economic actor is incentivized to follow the chain where they expect to hold the most value, both in the present and over the long term.

2. **PoW (the heaviest chain) provides a large part of the answer.**  
   It informs your node which chain is actually valued by the majority of economic nodes. There is a slight delay (visible when comparing hashrate changes to price changes), but it gives a reliable, and objective market-driven signal.

3. **Utreexo supplies the rest of the verification piece.**  
   Demand alone isn't enough if your UTXOs have been fraudulently spent on that chain. However, the decision isn't binary: if only a small fraction of your UTXOs are stolen, you might still rationally stick with the heaviest chain because its overall economic value outweighs the personal loss. The Ethereum vs. Ethereum Classic split is a good example; sticking with the chain that reverted some transactions may have made economic sense even for those personally affected.

4. **Future speculation about each chain's value is the wild card.**  
   This is inherently subjective and impossible to codify into a light client. The only viable fallback is to halt, notify the user, and require a subjective, manual decision.

---

## A Practical Light‑Client Protocol

So how would that translate into a practical light‑client design? Here's a seven‑step approach:

1. **Sync headers** just like a standard SPV client.

2. **Only consider forks whose tip is very close to the current wall‑clock timestamp.**  
   This filters out stale, abandoned forks from the outset.

3. **Use header timestamps for ad‑hoc fork detection.**  
   If blocks start taking unusually long to produce, assume a fork might be occurring, and aggressively seek out peers that might be mining on that alternative chain.

4. **When a fork is suspected (whether you find it or not), verify, but not everything.**  
   Instead of fully validating every block (as Floresta does), only check that the UTXOs relevant to *your* wallet (and possibly covenants and bridges) remain unspent fraudulently.

5. **If you find proof that a fork fraudulently spends a significant (user‑adjustable) portion of your funds, broadcast that fraud proof to your peers** and abandon that fork immediately, waiting for another that doesn't violate your property rights.

6. **If you find no such proof (and receive no credible fraud alerts)**, present the user with two clear options:
   - **A)** Require a substantially higher number of confirmations than usual, potentially scaled by the ratio of observed work between forks or the drop in hashrate on your current chain.
   - **B)** Halt all incoming transactions and notify the user that a subjective, manual decision is required before proceeding.

7. **If you receive a fraud proof from a peer** (affecting any UTXOs, not necessarily your own), take it as a strong warning. Present the user with one of the options above.

---

## Why This May Be Safer Than Full Validation

Counterintuitive as it sounds, a light client following the heaviest chain can be safer than a full node enforcing rigid local rules. Full validation mistakes a local policy for global consensus, but we have no direct way to measure the economic weight behind any given rule.

Take Segwit: its economic backing is only *inferred* by counting BTC locked in Segwit UTXOs, a weak signal compared to the behavior of miners as evident by block templates. If the majority of economic value started using older Bitcoin Core versions or new patched ones ignoring Segwit, for any reason, then a Segwit-enforcing node would be stranded on a worthless minority chain, if any block included an invalid transaction according to Segwit.

This weakness is magnified with soft forks like Taproot. Rules buried in unspent Taproot leaves are practically unobservable until spent; their economic weight is effectively observable only at spending time, yet a full node treats them as absolute.

Now consider a deliberate hard fork, say, an inflation bug is discovered and the community (exchanges, major holders, applications) decides it simply cannot tolerate it and patches it out. A subset of miners may refuse to upgrade. A full node that blindly follows local rules will track that heaviest *"valid"* but economically abandoned chain. Our light client, however, detects the fork (via block time anomalies or hashrate drops), halts, and asks the user to make a subjective value judgment, or at least requires hundreds of confirmations that never arrive, forcing the user to react. That is exactly the intervention needed to align with the chain where actual economic value resides.

The heaviest chain remains the only objective proxy for economic reality, but it only works in absence of chaos. Both full nodes and sensible SPV clients should halt on clear anomalies (difficulty drops, time lags) and ask the user to intervene. This light-client design does exactly that, making it more honest, and in practice, safer than pretending local rules are infallible, or that relative hashrate is irrelevant once any rule is broken.

---

## What Stops Miners From Stealing?

Miners are not automatically stopped from trying, but theft cannot happen silently. A fraudulent spend of a watched UTXO generates a fraud proof broadcast to peers, triggering confirmation slowdowns, user alerts, and ultimately subjective chain selection. If enough users judge the fraud unacceptable, economic weight shifts to the honest chain, destroying the cheater's market value. Miners follow paying chains; the deterrent is economic exile, not local rule enforcement.

Existing infrastructure—bridges locking BTC or atomic swaps—already blindly trusts the heaviest chain via SPV logic. This design adds a critical fraud-detection safety net absent today.

Consensus is not enforced; it is participated in through vigilance, alerts, and subjective choice, and that collective process, reflected in PoW over time, ultimately protects property rights.

---

### A Note on Utreexo, libbitcoinkernel, and Future Commitments

Building such a client is now practically feasible thanks to **Utreexo** (compact state proofs that eliminate the need for a full UTXO set) and **libbitcoinkernel** as seen in Floresta already.

To further strengthen this design, we could consider two ways to add utreexo commitments:

- **Soft fork:** miners commit to a Utreexo root in the coinbase transaction and the block is considered invalid if the commitment is invalid.

- **Velvet fork:** miners commit to the current Utreexo root and explicitly point to the latest root they endorse. This gives light clients a weaker guarantee than soft fork utreexo commitment, but more valuable than getting utreexo roots from random peers that don't spend PoW and trusting their consensus!

-------------------------

AdamISZ | 2026-08-26 14:19:04 UTC | #2

[quote="Nuh, post:1, topic:2842"]
The heaviest chain remains the only objective proxy for economic reality, but it only works in absence of chaos.
[/quote]

I think one way to think of it is as a *binding agent*. It's not a magic incantation that forces anyone to do anything, but see Schelling etc.

-------------------------

cmp_ancp | 2026-08-26 17:28:53 UTC | #3

Quick (and maybe uninformed) note: one of the bottlenecks on doing fullchain ZKP is the necessity on entire rule set being prooven in each block. If we assume the POW only, maybe just untill few blocks behind the tip, couldn’t we make a light client ZKP based?

In that way, light clients could startup new light clients in seconds, just distributing ZKPs. Those would prove the chain to be valid, to have X size, to be related to Y tip header and to be related to Z utreexo commit.

-------------------------

Nuh | 2026-08-26 18:47:11 UTC | #4

If bitcoin already had a Utreexo commitment as a soft fork then yes you can do that, concretely you would prove the headers up until 2016 blocks ago or a year ago or whatever you like, then starting from that effective checkpoint prove the execution of the remaining tail. This would save a lot of proving time.

However, this is the exact opposite of what I am saying. Because a ZK proving system requires an immutable consensus rules, but that doesn't exist in bitcoin at all, the best we have is libbitcoinkernel, but you have no idea what version of that is the economic majority enforcing, and there is no guarantees that they won't revert to an older version and stop enforcing a soft fork etc...

What I am trying to say; in practice, most of the time you just want to follow the chain that everyone is following, unless there is a major preach that either 1) steals your money 2) OR sets a precedent that destroys the long term value of Bitcoin so you bet against the sustainability of that heaviest chain.

Well, if we admit that the definition of consensus in practice is more about the aggregate decisions of human beings now and in the future, then ZK systems are absolutely useless, because code is _NOT_ law. And the best ZK proofs can do is summarize the headers chain, but Bitcoin headers are very small and grow very very slowly, so the upside is absolutely not worth it.

-------------------------

cmp_ancp | 2026-08-26 19:16:29 UTC | #5

Excuse me, maybe I hadn't been expressive enough on my question.

I thought on making a ZKP without considering consensus rules (like script executions) at all, but only as a compact way to prove a certain utreexo proof is bound to a tip, and that a tip has N PoW. I mean, taking exactly your trust assumptions, only counting UTXOs in and out of each block, and updating utreexo in the inside logic. If that was possible (and computationally feasable), so the ZKP substitutes the necessity of an utreexo proof commitment in the block.

If this ZKP is recursive (takes a past ZKP and proves upon it the addition on new blocks), so we could have a network of light clients that actually decentralize the network, capable of bootstraping third parties with (minor) trust. Instead of a competition of chains, we have a competition of ZKPs generated by those nodes, and someone just takes the most recent, with the most PoW ZKP.

-------------------------

Nuh | 2026-08-27 00:57:02 UTC | #6

I think [Raito](https://github.com/starkware-bitcoin/raito) is what you are looking for. But also Floresta does the same thing without the complexity of ZK systems, but yes it takes much more time and bandwith, so things like Raito are very effective at compressing that. But you still need p2p gossip like in Floresta, to find out about competing forks etc.. So again, the ZK compression is a marginal enhancement.

-------------------------

ajtowns | 2026-08-27 05:12:10 UTC | #7

[quote="Nuh, post:1, topic:2842"]
I’ve been trying to articulate, primarily to myself, a practical, pragmatic, and realistic definition of *consensus*, viewed strictly from the perspective of economic actors rather than theoretical correctness.
[/quote]

For me, that question isn't sufficiently defined: consensus on what? In the "full node" world, there's two major questions to answer:

 * what are the rules I expect to enforce?
 * what is the most work chain that obeys those rules?

The first determines what software you should run, the second is just the software doing it's job relaying and validating blocks. If you're "in consensus" on the rules you enforce, you'll continue operating on the same chain that exchanges call BTC; if you're not, you'll be spending your time arguing why you're on the "real Bitcoin" and everyone else is conspiring against you. If you're not on the most work chain for those rules, you (hopefully) just have buggy software and should switch to something better.

For a light client, you're only enforcing PoW rules, not all the others. *Someone* is still enforcing those rules though, and you're just letting someone else do it on your behalf. Who is that?
 * Perhaps it's "the economy" by just choosing whichever chain has most proof-of-work, because proof-of-work is expensive and it's unlikely that people will waste resources building an invalid chain
 * Perhaps it's a trusted electrum server or similar, run by a reputable company that you trust to run a full node and enforce the rules you care about for you
 * Or, perhaps you're connecting to your own full node, that validates all the rules you support, and you're just separating responsibilities to different pieces of software

If you're not trying to follow "the sha256d chain that has at least 80% of market value amongst sha256d chains", I think you should either not be using a light client, or be pointing your light client at a peer you trust.

[quote="Nuh, post:1, topic:2842"]
Counterintuitive as it sounds, a light client following the heaviest chain can be safer than a full node enforcing rigid local rules.
[/quote]

That's not merely counterintuitive, it's false. For your other points in that section: there is no difference between "the majority of [the market] started using [full node versions] ignoring Segwit" and "a deliberate hard fork". A hard fork is simply a jargon-y way of saying the majority of the market switched to a version that is not backwards-compatible with what the market had been using previously.

The heaviest chain is only a proxy for economic reality within the set of chains that share the same proof-of-work; it does not have any ability to capture alternatives that alter the PoW rules, and because Bitcoin's PoW rules are very much winner-takes-all, every alternative is, in practice, forced to alter the PoW rules in one way or another to be sustainable.

[quote="Nuh, post:1, topic:2842"]
What Stops Miners From Stealing?
[/quote]

The answer is people running full nodes that enforce the rules that make stealing coins (or printing new ones) invalid. Running a light client and freeloading off other people doing that is fine, but it's still freeloading.

[quote="Nuh, post:1, topic:2842"]
Building such a client is now practically feasible thanks to **Utreexo**
[/quote]

Utreexo isn't a light client, it's a different model for running a full node that's "lightweight". See [utreexod](https://github.com/utreexo/utreexod) or [floresta](https://www.getfloresta.org/) or [optech](https://bitcoinops.org/en/topics/utreexo/) rather than [River's glossary](https://river.com/learn/terms/u/utreexo/).

-------------------------

Nuh | 2026-08-27 08:52:56 UTC | #8

@ajtowns Thanks for your pushback, the purpose of this post wasn't to convince anyone of anything, but to get feedback like yours in case I am missing something.

I believe you are incorrect though, and that is fine, running full nodes is safe and no need to convince people to not do it.

I don't agree with most of your reply, including the semantics of light client vs lightweight, but I am open to changing semantics for the sake of conversations. 

However, I think I offered something that is different from 'running a full node that’s “lightweight”' .. namely that you don't actually have to enforce all the rules the way Floresta would do in the case of suspecting fraud, you could just enforce that your utxos are safe, totally ignoring any invalid witness, choosing to go with any hard forks as long as it seems to have significantly more demand (as witnessed by PoW) than any other fork, OR your client should halt confirming incoming transactions and alert you to take action.

My argument that this was the most pragmatic but also most honest definition of consensus that is close to users expectations. I don't think users would want to double down on following a worthless fork even after most economic value switched to something else by a hard fork or by just abandoning a soft fork (I don't care much that these are equivilant in theory, there are two very distinct ways to reach them, one with coordination and one without it).

Finally, this is not the same as an SPV or following the economic majority blindly, this is different because it adds the following:
1) enforcement of your utxos safety.
2) possibly enforcement of rules that a significant amount of your wealth is bound to, basically being strongly opinionated about one or more rules because it matters to you financially.
3) for any other rules you don't care about, or a general observable forks, slow down confirmations and ultimately halt and ask users to intervene and make a decision.

I find this more valuable for me personally than enforcing rules that I don't actually use or care about. if other people care about them enough, they will fork off just like I would for rules I care about, and if they are a majority they will drag the hashrate behind them and then I will follow them, and until that happens I would be slowing down my confirmations or halting entirely.

> *Someone* is still enforcing those rules though, and you’re just letting someone else do it on your behalf. Who is that?

Rules I care about, I enforced by me and people like me. Rules I don't care about, are enforced by people who do care about these rules. A full node is just a node that cares about all rules, I simply don't, and I will not lose all my money following a minority chain just because someone made an invalid transaction according to a soft fork I never cared about. 

The fact that this node is "lightweight" is incidental, you can do this selective enforcement on "full nodes" by running older Bitcoin Core versions, or a patched client, and in fact you can't even know who enforces what, especially because how taproot and pay to script hash hide these information for long term holders. 

The point here is that for some users, like myself, it might be the smartest course of action to only draw the line in the sand where it actually matters: 1) your own money, 2) enforcement of rules that secures your money 3) your subjective prediction for which fork will be sustainable and valuable long term. Drawing the line for all rules that the latest bitcoin core version enfroces regardless of whether they matter to you, is both excessive and risky and assumes that you would rather lose all your money than follow ANY hard fork, and I suspect not many people actually sign up for that conciously.

I hope this makes more sense to you. If not and you think it is stupid, then of course you should continue to run full nodes.

-------------------------

Nuh | 2026-08-27 09:04:57 UTC | #9

I also want to express another motivation to this discussion about subjective rules enforcement, I want to make it clear that not only is it safe to do on a personal level, but that if it is safe to do, we can expect people to be doing so, which means we don't know what rules do nodes enforce at all, and the only thing we know is what miners enforce, and of course they enforce these rules because their direct exit offramps are enforcing these rules. But we can't observe anything other than miners behavior. 

So we should actually invest in mechanisms that allow miners to very explicitly advertise, at all time, what rules they enforce.

This could be by advertising their client ids, or a bitmap field of rules/forks/bips that they enforce etc..

That + signaling of intentions to enable new rules or disable old ones, would allow the entire ecosystem to do this negotiation entirely on chain, and avoid the constant drama that we keep getting into, because we insist that node runners need to enfroce all rules or nothing.

-------------------------

