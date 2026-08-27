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

AdamISZ | 2026-08-27 14:25:12 UTC | #10

The best example here was probably around the Bitcoin Cash fork, because afair that's the only time there was a genuine full-on hashpower split. Not sure if it reached 50-50 at any point but even if it didn't it was very close.

In that case this question became very concrete indeed. Those of us using electrum as part of our workflow had to ask: hang on, what blocks am I being fed as valid?

Would it have been *completely incoherent* to take the position at the time "I'll treat the heaviest chain as the real one"? It would have been quite a mess. If you're really ever in that boat of "I'm willing to entertain both versions of bitcoin right now", you'd be far better off running *both* than this kind of "either" version that could switch your tx history completely from one moment to the next. That basically means you run your own fully validating nodes (both) or you enter into a relationship with a provider (e.g. electrum server(s)) that explicitly tell you what rules they are choosing to fully validate.

In practice, during that very dangerous time when Bitcoin's future appeared to be in the balance (having a 50-50 hashpower split is .. uh.. not great), even those of us who clearly sat on one side had to make very careful engineering/usage choices about handling this "dual reality".


I understand that this thread is more philosophical than anything so at a higher level: when there isn't meaningful consensus divergence (as per my earlier "binding, Schelling" comment) there's not really much to discuss; you can be lazy, perhaps in pockets, about the idea that you don't need to care about validation rules. Maybe not smart to be like that, but you can afford to be. But the problem with the "smooth brain" approach ( :laughing: ) of work-only is it's spectacularly brittle when put to the test. Consensus can't form around a system without rules. Proof of work has been in the past characterized as "signature over work" (instead of signatures over identities) and that's a pretty good way to describe it: it *attests* to state, but it doesn't define it. Without a definition of the state you're attesting to you just have a kind of amorphous goop.

-------------------------

ajtowns | 2026-08-27 15:31:05 UTC | #11

[quote="AdamISZ, post:10, topic:2842"]
the Bitcoin Cash fork, because afair that’s the only time there was a genuine full-on hashpower split. Not sure if it reached 50-50 at any point but even if it didn’t it was very close.
[/quote]

BCH as a percentage of the combined BCH+BTC hashrate peaked at about 12% cumulative and declined from there, though due to BCH's difficulty adustment algorithm fluctuating it had days where it hit 60%+ of hashrate and weeks where it hit 20%+.

```plotly
data:
  - x: [1501593374000, 1502198174000, 1502802974000, 1503407774000, 1504012574000, 1504617374000, 1505222174000, 1505826974000, 1506431774000, 1507036574000, 1507641374000, 1508246174000, 1508850974000, 1509455774000, 1510060574000, 1510665374000, 1511270174000, 1511874974000, 1512479774000, 1513084574000, 1513689374000, 1514294174000, 1514898974000, 1515503774000, 1516108574000, 1516713374000, 1517318174000, 1517922974000, 1518527774000, 1519132574000, 1519737374000, 1520342174000, 1520946974000, 1521551774000, 1522156574000, 1522761374000, 1523366174000, 1523970974000, 1524575774000, 1525180574000, 1525785374000, 1526390174000, 1526994974000, 1527599774000, 1528204574000, 1528809374000, 1529414174000, 1530018974000, 1530623774000, 1531228574000, 1531833374000, 1532438174000, 1533042974000, 1533647774000, 1534252574000, 1534857374000, 1535462174000, 1536066974000, 1536671774000, 1537276574000, 1537881374000, 1538486174000, 1539090974000, 1539695774000, 1540300574000, 1540905374000, 1541510174000, 1542114974000, 1542719774000, 1543324574000, 1543929374000, 1544534174000, 1545138974000, 1545743774000, 1546348574000, 1546953374000, 1547558174000, 1548162974000, 1548767774000, 1549372574000, 1549977374000, 1550582174000, 1551186974000, 1551791774000, 1552396574000, 1553001374000, 1553606174000, 1554210974000, 1554815774000, 1555420574000, 1556025374000, 1556630174000, 1557234974000, 1557839774000, 1558444574000, 1559049374000, 1559654174000, 1560258974000, 1560863774000, 1561468574000, 1562073374000, 1562678174000, 1563282974000, 1563887774000]
    y: [5.280065596928969, 6.138907103956324, 9.568377583101272, 13.161148360831069, 12.104787691279313, 11.956468290059332, 12.063668932498892, 12.551963951700252, 12.755923485877247, 12.580956547741472, 12.053817048608495, 11.19253431842277, 11.535190970045035, 11.401008467979738, 12.356663687977122, 12.158619048735062, 12.177737424654042, 11.959727972365815, 11.564161439030993, 11.200988355197795, 11.293644415065723, 11.266267296255924, 11.177114950714598, 11.193549278095185, 11.062351066519936, 11.04111442908962, 10.980495333376766, 11.044747305669887, 11.106477190889025, 11.046233284627336, 10.953172238126214, 10.862688186165776, 10.815497291117717, 10.777819839261193, 10.66656379404519, 10.538196506548864, 10.421199573618308, 10.418732647473927, 10.513797568937285, 10.663759057392081, 10.796336941587443, 10.847147011279393, 10.833653339702048, 10.819924263196377, 10.89283067608251, 10.889803320357695, 10.852779556201032, 10.872213701613784, 10.888308228615072, 10.927131131738049, 10.853404619473036, 10.764567791123138, 10.671083755860394, 10.55492449921259, 10.43695591192091, 10.304792193425062, 10.197979320061574, 10.097148876108347, 9.984329988059951, 9.851388239268429, 9.753828401097138, 9.68389917814522, 9.592443048177097, 9.500099064439505, 9.427434601604146, 9.3516695516666, 9.352423983251102, 9.364185557601488, 9.305704975495805, 9.220829382825565, 9.118916686486926, 9.002588318314263, 8.912547568470018, 8.816191114535998, 8.71585181902225, 8.618743624476885, 8.531849109853997, 8.436815746707758, 8.344398316818515, 8.248557699749941, 8.159565215159038, 8.079679076385176, 7.997105658396876, 7.920698589772485, 7.842779841564132, 7.766749922293075, 7.701866079523616, 7.6608251460888415, 7.621133082016862, 7.584901143867123, 7.536279548126884, 7.486099090906306, 7.435928306158142, 7.381902200373536, 7.327596331479692, 7.282142291410542, 7.234496622364298, 7.187160014521751, 7.126270649508118, 7.0598651824937, 6.979897117928966, 6.9057415934957485, 6.835474654553007, 6.759386930177361]
layout:
  title: Cumulative work after branch
  yaxis:
    title: BCH percent of total work
  xaxis:
    type: date
```

```plotly
layout:
  title: Weekly work
  yaxis:
    title: BCH percent of total work
  xaxis:
    type: date
data:
 - x: [1501593374000, 1502198174000, 1502802974000, 1503407774000, 1504012574000, 1504617374000, 1505222174000, 1505826974000, 1506431774000, 1507036574000, 1507641374000, 1508246174000, 1508850974000, 1509455774000, 1510060574000, 1510665374000, 1511270174000, 1511874974000, 1512479774000, 1513084574000, 1513689374000, 1514294174000, 1514898974000, 1515503774000, 1516108574000, 1516713374000, 1517318174000, 1517922974000, 1518527774000, 1519132574000, 1519737374000, 1520342174000, 1520946974000, 1521551774000, 1522156574000, 1522761374000, 1523366174000, 1523970974000, 1524575774000, 1525180574000, 1525785374000, 1526390174000, 1526994974000, 1527599774000, 1528204574000, 1528809374000, 1529414174000, 1530018974000, 1530623774000, 1531228574000, 1531833374000, 1532438174000, 1533042974000, 1533647774000, 1534252574000, 1534857374000, 1535462174000, 1536066974000, 1536671774000, 1537276574000, 1537881374000, 1538486174000, 1539090974000, 1539695774000, 1540300574000, 1540905374000, 1541510174000, 1542114974000, 1542719774000, 1543324574000, 1543929374000, 1544534174000, 1545138974000, 1545743774000, 1546348574000, 1546953374000, 1547558174000, 1548162974000, 1548767774000, 1549372574000, 1549977374000, 1550582174000, 1551186974000, 1551791774000, 1552396574000, 1553001374000, 1553606174000, 1554210974000, 1554815774000, 1555420574000, 1556025374000, 1556630174000, 1557234974000, 1557839774000, 1558444574000, 1559049374000, 1559654174000, 1560258974000, 1560863774000, 1561468574000, 1562073374000, 1562678174000, 1563282974000, 1563887774000]
   y: [5.280065596928969, 7.05070609896633, 15.942473807006435, 23.804936599512335, 8.345017187745322, 11.327888000955967, 12.599566203532964, 15.365403779901566, 14.083808437761686, 11.261012136377335, 7.521506640638981, 3.395036755662245, 14.66716517217434, 9.998151631931817, 23.041655082196822, 9.811843475237305, 12.41293630659412, 9.19999353538492, 6.563068524309849, 6.700984199092893, 12.54433559064045, 10.903034164713283, 9.965563753207443, 11.414313053905575, 9.419606256762258, 10.760663211359422, 10.16556567262512, 11.957006953335963, 12.001925374667447, 10.148745963838898, 9.518171360477732, 9.435966530729473, 10.004203136217958, 10.101410319658207, 8.646261535711067, 8.144425104277508, 8.219090679297663, 10.374048350767076, 12.422261068023424, 13.669180488278826, 13.474580332034702, 11.913393658263578, 10.552976452911933, 10.538354543684095, 12.469448913414947, 10.8261130354074, 10.070369514016276, 11.346742431901781, 11.265961348881094, 11.923196105162337, 9.08059752541583, 8.546707255007663, 8.411124516094533, 7.756722567435173, 7.526260010127296, 7.039227786075903, 7.359405406004338, 7.374687643222506, 6.756331614658823, 6.219116703252558, 6.98921380951159, 7.488249512452557, 6.775004451493882, 6.5125841674291305, 6.833694216513514, 6.809556356213122, 9.381303594881762, 9.838783688508567, 6.5414791723295975, 4.628519425953451, 3.2173053458192915, 2.5496042525519327, 3.9340678596857694, 3.657813433013932, 3.515342685607924, 3.336809627229443, 3.5509275891027494, 3.1286533322638275, 3.120029739129486, 3.0244790189047728, 3.160297160279125, 3.333185354707205, 3.1921345894243593, 3.2051403927290876, 3.327524832500892, 3.4002458142105, 3.7566935237824346, 5.2114616140320384, 5.158628313536653, 5.318119870108965, 4.5454427490396725, 4.4730529388202624, 4.344972895697582, 4.141669783371672, 4.139004526829368, 4.480333083750567, 4.476120465208893, 4.322275518240184, 3.641400457923573, 3.2942232568552194, 2.876695353803424, 2.8428440725230044, 2.901879582686272, 2.7603459561948327]
```

-------------------------

