# Modifying BIP54 to Support Future nTime Soft Fork

josh | 2025-12-15 00:36:53 UTC | #1

### TLDR

BIP54 prevents the "timewarp attack" but inadvertently also precludes a future soft fork that could resolve the `nTime` overflow problem, which is expected to halt the chain in 2106. Modifying BIP54 slightly by including a `u64` timestamp in the coinbase can resolve timewarp concerns, while leaving the door open for a soft fork that resolves the `nTime` problem, rather than a hard fork.

### The Problem

[Active work](https://github.com/bitcoin-inquisition/bitcoin/pull/99) is underway to implement [BIP54](https://github.com/bitcoin/bips/blob/6ce21f4eaeb4a73373688e99d7ea74a6c0413f22/bip-0054.md), also known as the [Great Consensus Cleanup](https://delvingbitcoin.org/t/great-consensus-cleanup-revival/710). One of the goals of BIP54 is to prevent the "timewarp attack," whereby an attacker artificially suppresses the median time past (MTP) in order to artificially reduce the difficulty target.

The proposed solution in BIP54 is:
> Given a block of height $N$:
>
> - If $N\ \%\ 2016$ is equal to $0$, `nTime` must be set to a value higher than or equal to the `nTime` at height $N-1$ minus $7200$ $(T_N ≥ T_{N−1} − 7200)$

This eliminates the attack vector, but it inadvertently precludes a future generation from resolving the `nTime` overflow problem via a soft fork, rather than a hard fork.

### A Possible `nTime` Soft Fork

Imagine that it is the year 2080. Developers want to use immutable software that can run for the next 100 years (ex: an SPV bridge on a L2), but a hard fork to increase the `nTime` field from `u32` to `u64` is not yet pressing enough to be viable. Instead, users adopt the following soft fork:

> Given a block of height $N$ and a timestamp $T_A$ at activation height $H$:
>
> - Miners store the timestamp $T_N$ in a `u64` field in the coinbase.
>
> - If $N \ \% \ 2016 < 2015$: miners set `nTime` to $T_A+N-H$.
> 
> - If $N \ \% \ 2016 = 2015$: miners set `nTime` to $(T_A + N - H - 2015) + T_N - T_{N-2015}$.

Consequently, MTP increases by at most 1 each block, ensuring the chain effectively never halts (assuming $T_H$ is sufficiently low).

In addition, the difficulty adjustment remains the same for legacy nodes and upgraded nodes. This resolves a DOS concern @gmaxwell [raised](https://groups.google.com/g/bitcoindev/c/L4Eu9bA5iBw/m/_5HEbNmHAQAJ) in my original post on the mailing list.

### Modifying BIP54

I would not venture to propose that we fix the `nTime` problem today. The `nTime` problem is still too far in the future, and it may turn out that a hard fork is the better option. But I would tend to prefer a fix to the timewarp problem that does not bind the hands of future generations.

A simply modification to BIP54 would resolve the problem:

> Given a block of height $N$:
>
> - Miners store the timestamp $T_N$ in a `u64` field in the coinbase.
>
> - If $N\ \%\ 2016$ is equal to $0$, $T_N$ must be set to a value higher than or equal to $T_{N-1}$ minus $7200$ $(T_N ≥ T_{N−1} − 7200)$.
>
> - If $N\ \%\ 2016$ is equal to $2015$, `nTime` must be set to the `nTime` at height $N-2015$ plus $T_N - T_{N-2015}$.

This effectively prevents a timewarp attack while keeping the door open to a soft fork that fixes the `nTime` problem. The cost is a few bytes in the coinbase, which can be removed if a hard fork ends up being the preferred option.

### Request for Feedback

Has the proposed modification to BIP54 been considered before? If not, would the flexibility of the proposed change be worth pursuing further?

-------------------------

ajtowns | 2025-12-15 03:13:31 UTC | #2

That's not "really" a soft fork, in that it would make the chain's mediantime lag signficantly behind real time, so that transactions with an nTimeLock set by timestamp rather than block height would not be spendable at the time they were expected to be. That's somewhat confiscatory, particularly if the timelock setting is enforced by OP_CLTV.

The "obvious" fix to the nTime overflow issue is to calculate a 64 bit nTime64 from the 32 bit nTime something like:

```
    uint64_t mediantime = pindexPrev->GetMedianTime();
    uint64_t nTimeHigh = mediantime & 0xFFFFFFFF00000000ull;
    uint64_t blockntime = nTimeHigh | uint32_t{block.nTime};
    if (blockntime < mediantime) blockntime + 0x100000000ull;
```

That's a hard fork (but only compared to the chain stopping entirely), and means that a 64 bit replacement for timestamp-based nLockTime would be needed for that feature to still be usable, but otherwise seems pretty straightforward and non-intrusive.

-------------------------

josh | 2025-12-16 16:14:28 UTC | #3

> That’s somewhat confiscatory, particularly if the timelock setting is enforced by OP_CLTV.

Unfortunately, that's true. But practically speaking, we can mitigate the risk of confiscation by signaling activation several decades in advance. I doubt there are many coins with a single spending path timelocked past 2050, for instance. As long as the risk of confiscation is no higher than it already is for BIP54 (i.e. the 2500 `CHECKSIG` limit), I see no reason why the possibility of confiscation would necessarily be an issue.

Generally speaking, we should always avoid a hard fork, if we can. We don't need to decide now, but if we can modify BIP54 at little cost so that a soft fork remains in the cards, shouldn't we?

-------------------------

AntoineP | 2025-12-17 15:05:14 UTC | #4

[quote="josh, post:3, topic:2163"]
Generally speaking, we should always avoid a hard fork, if we can.
[/quote]

I don't think this is quite true? We should make the best protocol design decisions. This often involves designing protocol changes as soft fork, but i don't think this is necessarily an absolute rule. Especially in this case, where a chain split is trivially prevented.

Also, we are not "generally speaking" here. We are talking of an issue that will arise in 80 years, that is in over 4x Bitcoin's total lifespan. It is interesting to consider, but i don't think this justifies making inferior protocol decisions for today's (and the coming decades') Bitcoin.

-------------------------

ajtowns | 2025-12-18 01:12:31 UTC | #5

[quote="AntoineP, post:4, topic:2163"]
[quote="josh, post:3, topic:2163"]
Generally speaking, we should always avoid a hard fork, if we can.
[/quote]

I don’t think this is quite true?
[/quote]

The reason to avoid a hard fork is to avoid (a) having people's software stop working or (b) forcing people to upgrade (which are two sides of the same coin). In the first case, having their software stop working is bad because it effectively confiscates their coins, which is the real issue. In the second case, if you've got a deployment schedule measured in decades anyway, I don't think that's an issue at all.

(There's also (c) that soft forking somewhat constrains your design space in a way that makes a bunch of bad decisions, like changing the 21M limit, much harder to implement. But if you limit your hard forks to one or two per century, I don't think that's too much of a problem)

-------------------------

gmaxwell | 2025-12-18 04:36:52 UTC | #6

I agree that there is no particular value in avoiding a hardfork to fix an issue like this– particularly as the other chain ‘stops’.  Were there a clean non-hardfork solution then it probably would be preferable, but abusing timewarp doesn’t strike me as clean at all.

It was also difficult enough to construct anti-timewarp fixes that were non-currently-disruptive *and* actually stopped timewarp– so I also have a concern about abuse potential ‘fixes’.  I suppose if one was really paranoid one could make the timewarp fix just not active for the very end of the time range (leaving enough room for whatever shenagans might be desired there) and trust that the overflow fix latent hardfork will moot that long before that time is reached.  This at least would leave no abuse potential before that point.

-------------------------

