# BMAX: pricing “sats now vs sats later” via a mining sharechain (no L1 changes, no custodians, no oracles)

VzxPLnHqr | 2025-12-16 19:00:26 UTC | #1

I am not sure whether this is better posted in #protocol-design or here. I generally tend to view Bitcoin first more from an economic lens such that all the technical underpinnings (while very important to get right too!) are more of a necessary side effect of **trying to achieve an economic goal of: long lived, self-consistent and decentralized base money for civilization.** So I am posting it here.

In that vain, I have been trying to solve the "sats now" vs the "sats later" problem in the most bitcoin-aligned way I can think of. It is a bit of a moonshot, but here is what I have come up with so far. I welcome any feedback. 

# BMAX

## Introduction and Motivation

Bitcoiners like to say "1 sat = 1 sat." In a narrow, protocol sense, that is true: the rules do not distinguish one satoshi from another. But economically, it is obviously false. A sat you can spend in the next block is not the same good as a sat that is unspendable for ten years. Time, and the ability (or inability) to act in time, is part of what you are holding.

Markets notice what we would sometimes prefer to ignore. Even if Bitcoin were perfectly fungible on‑chain (it is not), the time value of money makes **"a future sat"** a different economic object from **"a present sat."** Today, Bitcoin gives us a very rough expression of this: newly mined coins are timelocked for 100 blocks, so miner rewards are, in effect, 17‑hour zero‑coupon bonds backed by Bitcoin consensus. After that brief delay, all sats become "fungible enough," and the protocol ceases to care about time as a distinct economic dimension.

The rest of global finance does not stop there. It runs on *term structures*: 3‑month, 2‑year, 10‑year, 30‑year instruments and their corresponding yields. These imperfect, politically distorted yield curves still perform a real function: they let people compare **"1 unit of money now"** with **"1 unit of money later"** in a common language. Long‑lived infrastructure, capital budgeting, insurance, pensions, and even basic corporate planning are all, at bottom, about mapping future cash flows into present decisions via some discount function.

**Bitcoin, if it is to be the monetary base for civilization, cannot forever outsource this role to fiat credit systems and central banks.**

What is missing is **a Bitcoin‑native way to discover a discount curve for sats themselves**—a way to price *time* in Bitcoin terms—without:

- requiring any changes to Bitcoin,
- introducing custodians who issue IOUs,
- relying on oracles, interest‑bearing stablecoins, or fiat yield curves as inputs.

In other words, we want a mechanism that lets the market say, in sats, *"this is what 1 BTC delivered in 5 years is worth today"*—and we want that mechanism to be as trust‑minimized and Bitcoin‑aligned as possible.

BMAX is an attempt to get as close as we can to that ideal, *without touching Bitcoin at all.*

---

At a purely mechanical level, BMAX is “just” a p2pool‑style, proof‑of‑work sharechain that mines Bitcoin blocks. From Bitcoin’s perspective, BMAX is a normal mining pool that:

- produces fully valid block templates,
- sometimes wins the coinbase,
- and splits that coinbase according to its own off‑chain rules.

Inside BMAX, however, we deliberately structure those off‑chain rules to surface what Bitcoin currently hides:

- **Shares** are perpetual, equity‑like claims on future BMAX‑mined Bitcoin rewards. Each share has a probabilistic chance, via a deterministic random selection (p2share), of receiving a full coinbase when BMAX finds a block.
- **Bonds** are created by **burning** shares to enter a FIFO queue that is serviced by future coinbases. When a bond is redeemed, its principal shares are destroyed (and no longer eligible for selection in the random-share mechanism), permanently shrinking share supply.

If you accept that "1 sat now" and "1 sat in ten years" are different economic goods, BMAX is a proposal for letting Bitcoiners *price that difference in Bitcoin terms*, without asking anything more of Bitcoin than what it already provides.

## Abstract

BMAX is a p2pool‑style, proof‑of‑work sharechain that coordinates decentralized Bitcoin mining without any changes to Bitcoin. BMAX issues two instruments—probabilistic equity‑like **shares** and FIFO queue‑based **bonds**—that determine how the Bitcoin coinbase is allocated when the pool finds a valid block. A deterministic random selection mechanism (**p2share**) picks one share and constrains the Bitcoin coinbase: if chosen share is **unbonded**, its owner receives the full coinbase; if it is **bonded**, the full coinbase is used to redeem up to $M$ bonds from the head of the queue. Bonds are created by burning unbonded shares and redeemed in FIFO order, burning their principal shares at redemption and making share supply deflationary over time. A difficulty‑style control loop adjusts the share cost of creating new bonds so that the expected time to redeem half the current bond queue tracks a target horizon. The purpose is to let the market discover a Bitcoin‑native term structure for sats funded solely by actual Bitcoin block rewards.

## Protocol summary

BMAX nodes are, by definition, also full Bitcoin nodes.

### Core state (at sharechain height $k$)

Difficulties:

- $D_{S,k}$: sharechain difficulty (in units of $2^{32}$ hashes).
- $D_{B,k}$: Bitcoin difficulty (same units), taken from validated Bitcoin headers.

Share supply:

- $U_k^*$: total **unbonded** shares outstanding.
- $V_k^*$: total **bonded** shares outstanding (principals of all unredeemed bonds).
- $S_k^* = U_k^* + V_k^*$: total shares outstanding.
- $\alpha_k = \dfrac{V_k^*}{S_k^*}$: bonded fraction.

Bond queue:

- $Q_k$: number of outstanding bonds (FIFO).
- Each bond has a principal $\pi$ measured in shares (set at creation).
- $M \ge 1$: max number of bonds redeemed per redemption event (genesis constant).
- Optional $B_{\text{per\_block\_max}}$: cap on newly created bonds per sharechain block.

Bitcoin reward:

- $R_k$: sats paid by the Bitcoin coinbase in a Bitcoin win (subsidy + fees).

Share issuance:

- Per sharechain block, the miner receives newly issued unbonded shares
  $$
  U_k = c \cdot D_{S,k}
  $$
  where $c$ is a genesis constant.

Fees:

- Sharechain transaction fees are paid in **unbonded shares** and collected by sharechain miners.

### p2share: deterministic random selection on Bitcoin wins

When a sharechain block at height $k$ is also a Bitcoin‑valid block:

1. All nodes deterministically compute a selected share index
   $$
   q = \text{sha256}(\text{"q selector"} \,\|\, H_{k-1}) \bmod S_k
   $$
   where $H_{k-1}$ is the previous sharechain block hash and the state used for $S_k$ is fixed by consensus (e.g., “pre‑block state”).

2. Coinbase allocation rule:

- If the selected share is **unbonded**: pay the full $R_k$ to that share’s owner.
- If the selected share is **bonded**: treat this win as a **redemption event** and allocate the full $R_k$ to the bond queue as follows:
  - Let $m = \min(M, Q_k)$.
  - Redeem the first $m$ bonds in FIFO order.
  - Each redeemed bond receives
    $$
    \frac{R_k}{m} \text{ sats.}
    $$
  - For each redeemed bond with principal $\pi_{\text{bond}}$, burn its principal shares (these shares become ineligible for future random selection):
    $$
    V_{k^+}^* = V_k^* - \pi_{\text{bond}},\quad S_{k^+} = S_k - \pi_{\text{bond}}.
    $$

### Bond creation (bonding)

To create a bond at sharechain block $k$, the creator pays a bonding price $\pi_k$ in unbonded shares.

Consensus effects per bond created:

- $U_{k^+}^* = U_k^* - \pi_k$
- $V_{k^+}^* = V_k^* + \pi_k$
- $Q_{k^+} = Q_k + 1$

All bonds created within the same sharechain block use the same $\pi_k$. Optionally enforce $B_{\text{per\_block\_max}}$.

## Control loop: dynamic bonding price $\pi_k$

Goal: steer the expected number of sharechain blocks needed to redeem **half** of the current bond queue toward a target $T_{\text{target}} > 0$.

### Expected half‑clear time $T_k$

Define:

1. Expected sharechain blocks per Bitcoin win:
   $$
   \theta_k = \frac{D_{B,k}}{D_{S,k}}.
   $$

2. Redemption events needed to redeem half the queue (by count):
   $$
   N_k^{(1/2)} =
   \begin{cases}
     0, & Q_k = 0,\\[4pt]
     \left\lceil \dfrac{Q_k}{2M} \right\rceil, & Q_k > 0.
   \end{cases}
   $$

3. A Bitcoin win is a redemption event with probability $\alpha_k$ (a uniformly selected share is bonded with probability $\alpha_k$).

Thus:
$$
T_k =
\begin{cases}
  0, & Q_k = 0,\\[6pt]
  \dfrac{D_{B,k}}{D_{S,k}} \cdot \dfrac{N_k^{(1/2)}}{\alpha_k}, & Q_k > 0.
\end{cases}
$$

### Update rule for $\pi_k$

Genesis parameters:

- $T_{\text{target}} > 0$: target expected half‑clear time (in sharechain blocks).
- $\phi_{\text{up}} \ge 1$: max multiplicative increase per block.
- $\phi_{\text{down}} \in (0,1]$: max multiplicative decrease per block.
- $\pi_{\text{init}} \ge 1$: initial bond price (shares per bond).

Per sharechain block $k>0$:

1. Compute
   $$
   r_k = \frac{T_k}{T_{\text{target}}}.
   $$

2. Clamp
   $$
   f_k =
   \begin{cases}
     \phi_{\text{down}}, & r_k < \phi_{\text{down}},\\[4pt]
     r_k, & \phi_{\text{down}} \le r_k \le \phi_{\text{up}},\\[4pt]
     \phi_{\text{up}}, & r_k > \phi_{\text{up}}.
   \end{cases}
   $$

3. Update
   $$
   \pi_k = \pi_{k-1} \cdot f_k.
   $$

4. Round deterministically to an integer number of shares and enforce bounds:
   $$
   \pi_k =
   \min\left(
     U_k^*,\;
     \max\left(
       1,\;
       \left\lfloor \pi_k \right\rceil
     \right)
   \right).
   $$

Properties:

- $\pi_k \ge 1$ and $\pi_k \le U_k^*$.
- Per‑block multiplicative change is bounded (except where clipped by 1 or $U_k^*$):
  $$
  \phi_{\text{down}} \le \frac{\pi_k}{\pi_{k-1}} \le \phi_{\text{up}}.
  $$
- If $Q_k = 0$, then $T_k = 0$ and the rule pushes $\pi_k$ down (bounded by $\phi_{\text{down}}$) toward 1.

## Security notes

BMAX is an external sharechain, so deep sharechain reorgs must be made expensive and visible.

- **Bitcoin anchoring (optional but recommended)**: participants periodically commit sharechain headers into Bitcoin via compact timestamp commitments. Nodes treat history before the most recent sufficiently confirmed anchor as final for practical purposes (reject reorgs that would cross it).
- **BMAX‑mined Bitcoin blocks as checkpoints**: when BMAX wins a Bitcoin block, the coinbase outputs encode a specific allocation consistent with the sharechain state. Rewriting that allocation would require rewriting Bitcoin, which is outside BMAX’s threat model.

## Economic interpretation

The intended economics of BMAX rely on several layers working well together. The protocol itself only defines shares, bonds, and the control loop; markets decide prices.

- **Unbonded shares $\approx$ solo mining**: perpetual probabilistic claim. When selected on a Bitcoin win, they receive the full $R_k$.
- **Bonds**: created by burning $\pi_k$ shares to enter a FIFO queue. When a bonded share is selected on a Bitcoin win, the full $R_k$ is paid to redeem up to $M$ bonds; redeemed bonds burn their principal shares.
- **Term structure**: the market prices shares and bonds against BTC; the protocol’s only “knob” is $\pi_k$, adjusted mechanically to keep the bond queue’s expected half‑clear time near $T_{\text{target}}$, using only $D_{B,k}$, $D_{S,k}$, $Q_k$, and $\alpha_k$.

While such markets cannot be guaranteed by protocol design alone, both Bitcoin and BMAX are permissionless and L2-capable (i.e. Lightning) networks, so it **is plausible that atomic swap markets will emerge** to support trading between BTC, shares, and bonds.

## Conclusion

If Bitcoin is going to stand alone as global base money, something like this—*Bitcoin‑native time pricing*—needs to exist somewhere. BMAX is one concrete, implementable proposal for how to get there, without asking the base layer to change at all.

-------------------------

VzxPLnHqr | 2025-12-17 19:44:45 UTC | #2

People seem to get caught up on the fact that this is using bitcoin mining under the hood. The following very simplified description might appeal to more people:

BMAX is a decentralized perpetual, fair, and **open lottery for a future UTXO, with _transferrable_ tickets**:

- **decentralized** because it is operated by a p2pool-inspired mining network (which appears to bitcoin as any other source of hashrate)

- **fair** because it is ultimately equivalent in expectation to bitcoin solo mining

- **perpetual** (repeated) because the current pot is distributed every time a mainchain block is found by the lottery participants (miners adhering to sharechain consensus)

- **open** because unlike other lottery schemes on bitcoin, the set of participants need not be enumerated ahead of time

-------------------------

ajtowns | 2025-12-18 01:06:43 UTC | #3

[quote="VzxPLnHqr, post:1, topic:2165"]
If Bitcoin is going to stand alone as global base money, something like this—*Bitcoin‑native time pricing*—needs to exist somewhere.
[/quote]

If Bitcoin is global base money, then I think its risk-free interest rate is simply fixed at 0% as a result of it being fixed supply.

If you're seeing a positive rate, then that's reflecting some degree of risk, eg that your counterparty won't actually give you the bitcoin in future. After all, if the bitcoin were locked in on-chain already, your counterparty isn't getting an advantage, so why would they pay any premium at all?

In this case, I think significant risk comes from the question of whether the sharechain pool has any hashrate in future, or if fees/block reward drop below expected levels. In either case, your hoped for funds and interest won't be available when you expected them to be.

There's nothing wrong with that of course, it just means this system isn't discovering Bitcoin's native time pricing, but rather providing an insurance/continuity feature for mining payouts, which is probably more practical and immediately useful anyway.

IMHO, anyway. And it's economics, where supply of opinions is always greater than demand.

-------------------------

VzxPLnHqr | 2025-12-19 00:39:45 UTC | #4

Thanks for your reply and engagement.

[quote="ajtowns, post:3, topic:2165"]
If Bitcoin is global base money, then I think its risk-free interest rate is simply fixed at 0% as a result of it being fixed supply.
[/quote]

I do not think it is quite that simple. Of course, if Bitcoin allows us, as a society, to approach a true risk-free rate of 0%, then that would be great and usher in a type of utopia because, by definition of a 0% risk free rate, people would value the future as much as the present. 

Alas, so long as we have finite lifespans, and we do not think that (simplifying here) log2_work in our logs will ever approach infinity, then even bitcoin has a positive risk-free rate. Bitcoin's future is uncertain.

Nevertheless it still may be worth trying to quantify, in an effort to reduce, that uncertainty it in a manner which does not rely on central authorities.

[quote="ajtowns, post:3, topic:2165"]
If you’re seeing a positive rate, then that’s reflecting some degree of risk, eg that your counterparty won’t actually give you the bitcoin in future.
[/quote]

I am not sure I follow which counterparty you refer to here. 

@ZmnSCPxj managed to succinctly summarize the BMAX concept this way:
> the point is to be able to create a futures market for Bitcoin-the-asset itself, and not actually a market for Bitcoin mining; hooking into Bitcoin mining is simply used as a proxy for expected value of Bitcoin-the-asset in the future (and thus an economically-rational proxy for Bitcoin futures).

The futures markets referred to would be the hoped-for exogenous effect of the endogenous instruments (shares, bonds) of BMAX which are themselves different types of claims on the stochastic stream of block rewards which are mined by BMAX. That stream is currently non-existent, and even if it did exist, it might be remain empty indefinitely (if there is no demand).

Yes, there is counterparty-risk here, but the sharechain network itself is the counterparty to the shareholders and bondholders. 

This is not fundamentally very different from the type of counterparty that Bitcoin is to its miners. Bitcoin enforces a 100-block (17-hour) lockup of coinbase outputs, during which time a miner is, effectively, holding a not-yet-transferable-on-chain instrument, the counterparty of which is the Bitcoin network. 

Bitcoin already consistently demonstrates this notion of network-as-counterparty for a very short time window. What we are contemplating here with BMAX is a way for a network, as part of its core offering, to explore what it means to have a longer time horizon, while still staying true to Bitcoin.

[quote="ajtowns, post:3, topic:2165"]
significant risk comes from the question of whether the sharechain pool has any hashrate in future
[/quote]

Yes, especially when the sharechain contributes only a trivial fraction of hashrate. However, if this sharechain grows to contribute a non-trivial fraction of Bitcoin hashrate, then the risk of the sharechain persisting might be perceived to reduce. But there are no guarantees. Nor are there any guarantees, from Bitcoin-consensus-observable data, that the next Bitcoin block will be produced, yet, thankfully blocks keep coming in!

[quote="ajtowns, post:3, topic:2165"]
this system isn’t discovering Bitcoin’s native time pricing, but rather providing an insurance/continuity feature for mining payouts, which is probably more practical and immediately useful anyway.
[/quote]

A term structure for anything (stochastic or concrete), so long as it is ultimately realized in sats is, at least in part, a statement about time in Bitcoin-native time pricing. Mining is just the proxy we can use to try to educe rationality into these statements.

Regarding the practicality/usefulness: while mining is how the instruments are created and maintained until they are ultimately settled/redeemed on Bitcoin, the usefulness of the instruments themselves is not limited to the mining industry.

Of course my preference would be that this mechanism, or something simpler which accomplished the same thing, were available on Bitcoin directly. In which case the signal would be more pure. It could be done as a soft-fork I think, but even still, it will will never happen.

So the next best thing I can come up with is this BMAX sharechain thing we are discussing here. Because the sharechain network _is a valid bitcoin miner_, and especially if the sharechain contributes a non-trivial portion of Bitcoin work then the signal can still get through.

How might the sharechain ever contribute a non-trivial portion of Bitcoin work?
- reduce protocol risk (i.e. keep it simple, but no simpler)
- get the economics right to reduce risk premia associated with the sharechain

Something I did not mention in the OP is that a positive side effect of the mechanism we are discussing here is that investors/hodlers can directly participate in bitcoin's future (which is mined into existence), even if they are not in a thermo-geo-economic advantaged position to mine, so long as they have the risk-appetite for it.

The argument is that this expands the pool of available capital which can be deployed in a trust-minimized way to secure bitcoin's future, and a side-effect is that we get a glimpse into the Bitcoin (via BMAX) network's collective time preference.

Naturally, no hodler should bet their whole stack on the future of Bitcoin via BMAX, but hodlers who are willing to allocate a marginal portion of their stack towards "risk," and who have been extremely hesitant to do so (because moving their precious sats in/out of centralized exchanges or buying equity of various supposedly bitcoin-aligned, yet ultimately traditional, companies, comes with a whole host of headaches that is hard to justify) may find solace here.

I suppose that if BMAX were to exist and never manages to contribute more than a de minimus portion of Bitcoin's work, then there is also a signal there too, albeit, a more sad one. In such a case, we can partially conclude:

- BMAX is perceived to be too risky relative to Bitcoin itself (despite the well-intended attempts of the BMAX designer(s) to keep it as bitcoin-aligned, and protocol-risk-minimized as possible), in which case perhaps a better BMAXv2,3,4 can be created to try to overcome those issues.
- or the collective desire for Bitcoin to stand on its own as a self-consistent global money without central authorities has diminished so much that it has been, for all intents and purposes, "captured" by the now-traditional high finance world of trust and custody that Bitcoin was created to obviate.

[quote="ajtowns, post:3, topic:2165"]
IMHO, anyway. And it’s economics, where supply of opinions is always greater than demand.
[/quote]

Your opinion, and others on this forum, is very valuable! Thank you for taking the time to provide it. 

**The big question here is if there is any real merit trying to bring BMAX into existence. Or will it be DOA because demand for this type of is not there anymore and unlikely to return?**

-------------------------

ajtowns | 2025-12-19 13:22:24 UTC | #5

[quote="VzxPLnHqr, post:4, topic:2165"]
by definition of a 0% risk free rate, people would value the future as much as the present.
[/quote]

I wouldn't go that far; it's just that the price pressure on a future BTC seems to narrow to 1:1 -- if I want to have 1 BTC in ten years' time, then if I already have 1 BTC (or more) that's obviously easy, I just time lock 1 BTC somewhere; but if I have less than 1 BTC, exchanging that for some other good, and then hoping I can sell it for a BTC-denominated profit soon doesn't seem very safe.

For inflationary currencies, the opposite is true -- trade $100k for shares, or gold, or land, or a bunch of other things, and you'll probably end up with more than $100k if you sell it a few years later, even without involving a counterparty, or adding value to whatever you bought before you sell it.

So to me, the risk free rate is just a different measure of currency inflation vs real goods. The "time value of money", where you're getting a zero-effort profit by loaning someone money now and (hopefully) getting repaid later, adds counterparty risk (whether that be the borrower directly, the intermediary bank, or the guaranteeing insurnace firm, government, or currency issuer) and gets you an corresponding profit.

I don't think you get the same "get interest on your bank deposits with 0 risk" behaviour if you don't have a currency issuer that can turn large scale defaults into just an inflationary shock.

That's how I look at it, anyway. YMMV obviously.

[quote="VzxPLnHqr, post:4, topic:2165"]
Yes, there is counterparty-risk here, but the sharechain network itself is the counterparty to the shareholders and bondholders.
[/quote]


So the way I'd look at this is that the counterparties here are future
miners vs present share holders, and via the FIFO queue, the risk that's
created is how long it will take the queue to drain so that you receive
your funds.

I'm not really following the logic for the unbonded shares though; it
seems buggy to me. Suppose when the sharechain is new, I mine 50% of
shares with 1% of global hashrate over a two week period, and leave all
my shares unbonded. In that case, the sharechain will get ~40 blocks
in a two week period, my shares will be selected for 50% of them,
and I'll get 20 blocks worth of reward, same as I would on any other
pool. But then if I stop mining, over the next two week period only ~20
blocks will be mined, but my shares are still eligible to be selected,
and that will happen about 33% of the time, so I'll get an additional
~6 blocks worth of reward.

So I think at that point everyone else would abandon the sharepool as
non-competitive and it would die (since ongoing participants are losing
funds to me whether or not they bond their shares). I think similar
logic applies after the long term -- once there's any substantial amount
of unbonded shares in the pool, creating new shares by mining into
the sharepool isn't going to be profitable compared to mining against
some other pool. So I think you need a rolling window of some sort for
unbonded shares, rather than a perpetual claim.

Assuming that's fixed somehow, recasting it as a "miners insurance"
type scheme, rather than a "time value of BTC" thing, would look
something like:

 * if your shares are unbonded, you have a chance of getting a big reward
 * if your shares are bonded, you will definitely get a smaller reward,
   but it will be delayed (both due to the time it takes to gather enough
   unbonded shares to buy a bonded share, and the time for the bonded
   FIFO queue to get to your share)

Targeting the expected value of both those to be roughly $R \cdot
\frac{D_S}{D_B}$ should be plausible and make the pool viable.

If you imagine there was some safe way for the sharechain to store rewards
then distribute it later (very non-trivial ofc), you could make the rule be that each share found
for a 2016 block retarget period (at constant target difficulty $D_B$)
accrues a total block reward of $nR$, which is then distributed evenly,
with roughly $\alpha n R$ BTC going to bonded shares, and the remaining
$(1-\alpha) n R$ BTC going to to $n$ lucky unbonded share holders.

That would involve bonded and unbonded share holders getting their payouts
at exactly the same time, so there's no conflict between present and
future value, just a risk tradeoff. You could probably have the bonded
payouts suffer a penalty that increases the unbonded miners' payouts,
perhaps encouraging more participation by large miners, helping to make
your pool more viable.

If you're targeting both forms of reward to have similar present value to
mining in a regular pool, then you're already distributing 100% of the
block reward, and I don't think you have any degrees of freedom left to
get interesting behaviour over the long term.

If you look at it as going from a 1-in-a-million chance at a million
dollars vs a 1-in-1 certainty of $1, you could perhaps consider mid-range
shares as well, where you might have a 1/10000 chance at $10,000 or a
1/100 chance at $100, so to speak. ie, the fundamental limit might be that you can
have 1000 outputs per block reward, but you have 1,000,000 shares per
block on average, then each share has a 1/1M chance of a 3.125BTC reward,
and you could combine 1000 of them to have a certainty of a 3.1mBTC
reward, but perhaps you could provide 100 of them to have a 10% chance of
a 3.1mBTC reward, or 1000 of them to have a 1% chance of a 0.3BTC reward.

Selling (bonded) shares for immediate BTC on an open market with an
automated market maker would likely be very useful, allowing miners
to be immediately liquid, without requiring the pool to act as an insurer.
I expect there's significant demand for a facility like that.

-------------------------

optout | 2025-12-20 05:24:25 UTC | #6

Superficially this seems to have quite some overlap with https://hashpool.dev/. Hashpool aims to solve by issue of valuing probabilistic future mining shares via mining futures, or a marketplace for current value of the mined shares (discounted price now vs. full price in the future when a block is found).
How do you see BMAX in relationship with hashpool?

-------------------------

VzxPLnHqr | 2025-12-21 02:13:51 UTC | #7

[quote="optout, post:6, topic:2165, full:true"]
Superficially this seems to have quite some overlap with https://hashpool.dev/. Hashpool aims to solve by issue of valuing probabilistic future mining shares via mining futures, or a marketplace for current value of the mined shares (discounted price now vs. full price in the future when a block is found). How do you see BMAX in relationship with hashpool?
[/quote]

Thanks for chiming in. I do recall coming across hashpool, and it may be appropriate and within the risk-tolerance for some users. However, due to the custodial nature of ecash, I have not investigated hashpool in relation to BMAX at all. 

One nice thing about BMAX, is that because it is its own network, and because BMAX nodes are also Bitcoin nodes, then BMAX consensus can serve as a sort of "observer" of relevant Bitcoin on-chain metrics (feerates, difficulty, etc), and can settle contracts/bets about these things, all without needing an oracle or federation. 

[quote="ajtowns, post:5, topic:2165"]
... to me, the risk free rate is just ...
[/quote]

We may simply be squabbling over subtly different definitions, but I do think we agree on the general effect: it is not generally rational for someone to pay more than $R$ sats _now_ to accept an offer whereby they will receive $R$ sats $T$ time-units from now. 

Rather, the amount someone might consider paying is simply the present value $P$ of that offer, where $P = R e^{r T}$ and where $r$ is the buyer's "discount rate." The value of $r$ implicitly accounts for whatever the actor cares about, including, but not limited to: counterparty default risk, inflation risk, and opportunity cost. For the latter, and at least for a subset of actors, it includes peeking at the current offerings of the traditional finance world, such as "what interest will the bank or a treasury bond give me?" and because that question is not denominated in sats, the buyer must then take a guess at translating that other unit into sats (at a future time), discounting accordingly, and so on.

We do not get to directly observe what exactly is the buyer's calculus with regard to $r$, but if we are lucky and can observe the prices paid for many similar such claims, we can begin to infer some information about the collective time preferences.

When $T$ is small (seconds to days), the inferred information about time preference is not very useful to anyone other than hedgers and speculators who want to act in such short time spans. However, when $T$ is large (months, years, decades), the inferred information about collective time preference becomes much more relevant for economic actors who have longer planning horizons. 

Miners themselves are good examples of these types of actors, but they are not the only ones. Any person or business which seeks to embark on a long-term capitally-intense journey should ask them selves: "what if we do not do this, and just buy bitcoin instead?" 

Right now, trying to answer that question is, in my opinion, too reliant on the traditional fiat world (and even then, the horizons in the fiat futures markets for bitcoin are not long), and we can do better.

I think you already know all of that, but I wanted to try to lay it out for others who may come across this thread in the future.

[quote="ajtowns, post:5, topic:2165"]
Selling (bonded) shares for immediate BTC on an open market ... would likely be very useful, allowing miners to be immediately liquid, without requiring the pool to act as an insurer. I expect there’s significant demand for a facility like that.
[/quote]

I suspsect there is demand for something like this too. While miners are the obvious initial "customers" / users of such a thing, I see no reason why it would not very quickly expand beyond that segment, and that is a good thing. 


[quote="ajtowns, post:5, topic:2165"]

I’m not really following the logic for the unbonded shares though; it seems buggy to me. Suppose when the sharechain is new, I mine 50% of shares with 1% of global hashrate over a two week period, and leave all my shares unbonded ...

[/quote]

The BMAX design needs to be robust to aptly raised concerns like this. Thank you @ajtowns for providing such helpful comments and ideas! Some of what you elucidated I had thought about, but had for some (clearly wrong!) reason dismissed at some point. Nevertheless, I am working to update the design so that these concerns may hopefully be alleviated while still preserving the desired underlying properties of incentive compatibility, oracle-free, non-custodial.

-------------------------

