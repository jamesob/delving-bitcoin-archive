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

