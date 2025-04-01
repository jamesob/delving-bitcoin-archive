# Timewarp attack 600 second grace period

sjors | 2024-12-17 07:53:01 UTC | #1

**Background**

The [original Great Consensus Cleanup soft fork proposal](https://github.com/TheBlueMatt/bips/blob/7f9670b643b7c943a0cc6d2197d3eabe661050c2/bip-XXXX.mediawiki) by @MattCorallo says the following:

> * Sadly, some deployed mining hardware relies on the ability to roll nTime forward by up to 600 seconds[3]. Thus, only requiring that the nTime field move forward during difficulty adjustment would allow a malicious miner to prevent some competitors from mining the next block by setting their timestamp to two hours in the future. Thus, we allow nTime to go backwards by 600 seconds, ensuring that even a block with a timestamp two hours in the future allows for 600 seconds of nTime rolling on the next block.

The footnote explains why 600 is probably the upper bound:

> [3] While no official stratum specification exists, the btc.com pool server (one of the most popular pool servers today) rejects shares with timestamps more than 600 seconds in the future at https://github.com/btccom/btcpool/blob/e7c536834fd6785af7d7d68ff29111ed81209cdf/src/bitcoin/StratumServerBitcoin.cc#L384. While there are few resources describing hardware operation today, timestamp rolling can be observed on the chain (in some rare cases) as block timestamps go backwards when a miner rolled one block nTime forward and the next does not, but only incredibly rarely more than 600 seconds.

`nTime` rolling is also not likely to go away. It's potentially useful for ASIC devices that can go beyond 280 TH/s. As explained in https://github.com/stratum-mining/sv2-spec/blob/52e1fa22f68c343a3d25a2b1a04f93f8e701eced/05-Mining-Protocol.md#511-standard-job : 

> The size of the search space for one Standard Job, given a fixed `nTime` field, is `2^(NONCE_BITS + BIP320_VERSION_ROLLING_BITS) = ~280Th` , where `NONCE_BITS = 32` and `BIP320_VERSION_ROLLING_BITS = 16` . This is a guaranteed space before `nTime` rolling (or changing the Merkle Root by sending a new Job).

This `nTime` rolling could be limited to similar numbers. E.g. a hypothetical 3 peta hash beast would need to roll the timestamp by 10 seconds every second. If it gets a new template every 30 seconds, it would roll by 300 seconds. Beyond that a pool proxy could (and should) just roll the extranonce and feed the miner a new template more frequently.

**Current proposal and implementation**

The [timewarp fix currently deployed on testnet4](https://github.com/bitcoin/bips/blob/f88f1e4392c871f206fe7ee70674c0a049d32ca7/bip-0094.mediawiki#user-content-Time_Warp_Fix) also allows `nTime` to go backwards by 600 seconds.

Currently when Bitcoin Core proposes a new block template it will determine the timestamp as follows:

1. Current time
2. If the MTP rule requires it, bump time
3. On testnet4, for the first block of a retarget period, if the previous block is from the future, bump time again, but minus 10 minutes. 

See https://github.com/bitcoin/bitcoin/blob/733fa0b0a140fc1e40c644a29953db090baa2890/src/node/miner.cpp#L33-L40

**The problem**

The 600 second grace period is cutting it too close imo, and we should consider increasing it to 2 hours for any future proposal (see https://delvingbitcoin.org/t/zawy-s-alternating-timestamp-attack/1062 and https://delvingbitcoin.org/t/great-consensus-cleanup-revival/).

For the discussion here I'll assume that the pool software[0] and miner firmware doesn't ignore the template timestamp.

Now if a malicious miners set their timestamp 2 hours in the future, relative to our node clock, _and_ if our template is used by an ASIC that wants to roll nTime forward by up to 600 seconds, this is only safe if we assume all our peers have a matching clock. But that defeats the purpose of the 2 hour future rule: *we shouldn't assume nodes have accurate clocks*.

We should also strongly discourage `nTime` rolling beyond a few minutes, so as to not eat too much into the network wide two hour tolerance for inaccurate clocks.

[0] public-pool (https://github.com/benjamin-wilson/public-pool/commit/4282233d2f11ceecbd0d142e8292ccc9c37ea999) and SRI ([fix](https://github.com/stratum-mining/stratum/compare/main...GitGab19:stratum:fix-timestamp-bug) coming) ignored the template timestamp, which only became obvious on testnet4 because its timestamps are far in the future. But these are not production environments.

-------------------------

zawy | 2024-12-17 08:54:33 UTC | #2

The most strict MAX_TIMEWARP I remember was my 1-day recommendation (and on every block), if the MTP is more than that far in the past. I thought it was known that something small like 600 was dangerous. Off-hand I can't remember the issue but I believe it's because you want to be sure MTP is the strictest limit to assigning times in the past. Maybe it was just a compatibility issue such as software somewhere assuming MTP is always the limit. Limiting it only on difficulty adjustment is even more of a red flag since it's more strict than the MTP. Block at height H has MTP as the limit, then H+1 has -600, then H+2 is back to MTP ... seems like someone will find a way to break something. 

If a miner has that much hashrate, his 1st timestamp should be in the past at 1/2 his average forward seconds of rolling, if MTP allows. Either way, having a known procedure discloses his hashrate if it's not already known, or discloses who won if hashrates are known.

A miner won't roll past the new future time limit if the prior block was at the limit (and for some reason he uses that as his starting timestamp), unless his hashrate is more than the network hashrate. Time advances faster than his rolling, so anyone accepting the prior block would accept his.

-------------------------

sjors | 2024-12-17 11:39:56 UTC | #3

The original consensus cleanup proposal uses 600 seconds for the reasons I describe above.

The testnet4 implementation started out with 7200 seconds. This was to allow miners to use the system clock. But that was considered a bad idea and so it was reverted back to 600 here: https://github.com/bitcoin/bitcoin/pull/30647

And so I'm proposing to bring it back to 7200, but not for the reason testnet4 initially did it.

As you say, anything less than 1 day is probably good enough.

Seperate from that is your observation here: https://delvingbitcoin.org/t/zawy-s-alternating-timestamp-attack/1062

-------------------------

zawy | 2024-12-17 12:09:30 UTC | #4

I didn't see any reasoning in your OP that led me to think it should me more restrictive than the MTP.  I don't understand @TheBlueMatt's comment that 7200 reverse would allow slightly more inflation than 600.  I prefer @murch's recommended 2 weeks limit over 2 hours.  I meant anything more than a day is good and safe  and anything less is risky.

-------------------------

sjors | 2024-12-17 13:11:28 UTC | #5

[quote="zawy, post:4, topic:1326"]
I didn’t see any reasoning in your OP that led me to think it should me more restrictive than the MTP
[/quote]

Correct, I was worried about the value being too low. I have no opinion on whether it could be (much) bigger.

-------------------------

zawy | 2024-12-17 13:29:34 UTC | #6

Then 7200 is too low because it could be more restrictive than the MTP, and usually would be is if it's trying to correct a +7200 advance in time.

-------------------------

AntoineP | 2024-12-19 15:02:23 UTC | #7

[quote="sjors, post:1, topic:1326"]
Now if a malicious miners set their timestamp 2 hours in the future, relative to our node clock, *and* if our template is used by an ASIC that wants to roll nTime forward by up to 600 seconds, this is only safe if we assume all our peers have a matching clock.
[/quote]

Hmm i agree that we should seek to minimize the potential for creating an (even temporarily) invalid block. But that's a lot of if's to get there:
- A miner is malicious; and
- This miner finds the last block in a period; and
- Our ASIC rolls `nTime`, we find the first block of the new period; and
- Our ASIC finds a solution with `nTime + n`; and
- We find the block in less than `s` seconds; and
- A majority of the hashrate has a clock behind ours by more than `n + s - 600` seconds.

-------------------------

MattCorallo | 2024-12-18 13:50:35 UTC | #8

I strongly disagree. The StratumV2 spec explicitly (I thought?) only allows a miner to roll nTime once per second. In Sv2, machines with more than 280 TH/s can either request multiple jobs from the pool by using multiple "channels", or can build their own work by requesting a merkle path to the coinbase where they can roll the extranonce. I'm unaware of any existing miners that roll nTime much more aggressively than one per second, but the existence of pool software that hard rejects rolling past 600 seconds strongly suggests it didn't exist at the time that document was written.

One of the reasons for making testnet4 limit at 600 seconds is to create a testing ground and make sure that no miners are broken by the limitation, giving us more data to make sure 600s is safe (though we believe it is for the above reasons).

I further don't buy that we need to make inferior protocol design decisions on the basis of some theoretical future mining device which ignores protocol restrictions on nTime rolling both in Sv2 and implicit in the Bitcoin protocol.

-------------------------

ajtowns | 2024-12-18 17:01:40 UTC | #9

Provided we don't roll `nTime` by more than whatever the value is that we can push `nTime` backwards (600 seconds in both cases here) towards real time, this doesn't seem like much of a concern? If our block's timestamp is invalid, then the malicious block's timestamp will also have been to be too far in the future, so any node that would have rejected our block due to the timestamp would also reject it due to its parent, so no matter what timestamp we gave it our block would be rejected...

Redoing the math: nNonce rolling gives 4GH; nNonce+BIP320 gives 280TH; if you bump nTime once per second (as expected) that's 4GH/s or 280TH/s. If you want to get into the PH/s range (seems like the best antminer currently advertised is in the 0.5PH/s range), then assuming you only provide new work every 30 seconds, then you probably want 7 bits of nTime to roll (128 seconds), which gets you 1.2PH/s. If you need the final bits of nTime to be zeroed to roll them, then the total offset is roughly doubled (+0 to +254 seconds, about 4 minutes). If you've got a range of 600s, then you can roll 8 bits of nTime for 2.4PH/s; if you also provide new work every 10s, then you've got 7.2PH/s; if you provide 4 units of work every 10s, that's ~30PH/s. So afaics 600s should be pretty fine, though I don't have any objection to increasing it.

We're considering here a rule that the first block of a new period's timestamp has to be bounded below by both mediantime and `prev->nTime-K`. A different way to achieve the same goal would be require that the last block of a period's timestamp has to be bounded below by mediantime, but also bounded above by `mediantime+K`. K here should perhaps be something on the order of 3 hours (one hour because mediantime already lags wall-clock time, and then another 2 hours on top of that to ensure there's some room for rolling, slow blocks, etc).

The downside of such an approach is that existing mining software that simply continually bumps nTime (once per second, eg) will eventually exceed the upper bound, and produce invalid blocks.

Sorry if that was a bit stream of consciousness.

-------------------------

sjors | 2024-12-20 06:18:13 UTC | #10

[quote="MattCorallo, post:8, topic:1326"]
The StratumV2 spec explicitly (I thought?) only allows a miner to roll nTime once per second
[/quote]

If that's the intention, then the spec should be clarified. Currently there's only an ambiguous statement buried in the discussion section. See https://github.com/stratum-mining/sv2-spec/blob/52e1fa22f68c343a3d25a2b1a04f93f8e701eced/10-Discussion.md#102-rolling-ntime and the header-only-mining (HOM) discussion here: https://github.com/stratum-mining/sv2-spec/pull/98/files#r1746795461

> make sure that no miners are broken by the limitation,

Unfortunately this bug could exist for years without detection, only revealing itself in the distant future when chips are fast enough to cause a problem.

[quote="ajtowns, post:9, topic:1326"]
If our block’s timestamp is invalid, then the malicious block’s timestamp will also have been to be too far in the future, so any node that would have rejected our block due to the timestamp would also reject it due to its parent
[/quote]

If the sv2 spec explicitly disallows accelerated nTime rolling then indeed the attacker would take the same risk as their victim.

But otherwise the malicious miner could use `extra_nonce` rolling to give themselves a bigger safety margin than their accelerated nTime rolling competitor.

[quote="MattCorallo, post:8, topic:1326"]
I further don’t buy that we need to make inferior protocol design decisions
[/quote]

I'm not convinced (yet) that we need to make these numbers so tight. It seems that having a few hours of padding, instead of 10 minutes, avoids some actual bugs (pool software ignoring nTime) and theoretical future bugs. While the only downside is a minuscule increase in worst case inflation.

-------------------------

AntoineP | 2025-01-29 15:37:36 UTC | #11

[quote="sjors, post:10, topic:1326"]
If the sv2 spec explicitly disallows accelerated nTime rolling then indeed the attacker would take the same risk as their victim.
[/quote]
So what is the attack scenario here? Minority miner A tries to get miner B to waste its hashrate by producing an invalid block. Miner A somehow figures out miner B is using `nTime` rolling past 600 seconds and its local clock is ahead of everyone else's. Miner A mines a block such as it is invalid to everyone else but miner B, in the hope that miner B would start mining with `nTime_A - 600`, roll the timestamp past `nTime_A`, and find a block with `nTime_B = nTime_A + s` in less than `s` seconds such as A's block is valid to the rest of the network but B's block is not. And all that before the rest of the network found a different pair of block.

At this point this is not an attack, it's a footgun. I don't see how A could ever expect to gain anything from trying this.

[quote="sjors, post:10, topic:1326"]
I’m not convinced (yet) that we need to make these numbers so tight. It seems that having a few hours of padding, instead of 10 minutes, avoids some actual bugs (pool software ignoring nTime) and theoretical future bugs. While the only downside is a minuscule increase in worst case inflation.
[/quote]
I don't think it's a fair characterization of the downside. Faster subsidy emission is only one of the harms of an artificially increased block rate. And if the leeway is large enough, the block rate increase isn't minuscule anymore.

~~Here is some numbers:~~ EDIT: the numbers are incorrect as they assume gains accumulate across periods, which isn't the case once the timestamp of the first block needs to move forward.
```
Leeway: 10 minutes. Max diff decrease per period: 1.0004960317460319. Number of periods to take the diff to 1 is 65169.20533651417, to halve the diff is 1397.7312609540088 and to reduce it by 10% is 212.45947546985983.
Leeway: 60 minutes. Max diff decrease per period: 1.0029761904761905. Number of periods to take the diff to 1 is 10874.99226688242, to halve the diff is 233.24385460225878 and to reduce it by 10% is 35.453787426590814.
Leeway: 120 minutes. Max diff decrease per period: 1.005952380952381. Number of periods to take the diff to 1 is 5445.563646962276, to halve the diff is 116.79495712078635 and to reduce it by 10% is 17.753194781141502.
Leeway: 240 minutes. Max diff decrease per period: 1.0119047619047619. Number of periods to take the diff to 1 is 2730.8374380149667, to halve the diff is 58.57025317383194 and to reduce it by 10% is 8.902859666282216.
```

<details>

<summary>Code</summary>

```python
import math

CURRENT_DIFFICULTY = 108522647629298

for leeway in [10, 60, 2*60, 4*60]:
    max_rate_decrease = 1 + leeway/20160
    diff_1 = math.log(CURRENT_DIFFICULTY, max_rate_decrease)
    diff_half = math.log(2, max_rate_decrease)
    diff_ninety = math.log(1/0.9, max_rate_decrease)
    print(f"Leeway: {leeway} minutes. Max diff decrease per period: {max_rate_decrease}. Number of periods to take the diff to 1 is {diff_1}, to halve the diff is {diff_half} and to reduce it by 10% is {diff_ninety}.")
```

</details>

-------------------------

sjors | 2024-12-23 04:06:55 UTC | #12

[quote="AntoineP, post:11, topic:1326"]
A could ever expect to gain anything from trying this.
[/quote]

I agree this is pretty unlikely.

[quote="AntoineP, post:11, topic:1326"]
`Leeway: 240 minutes. Max diff decrease per period: 1.0119047619047619. Number of periods to take the diff to 1 is 2730.8374380149667, to halve the diff is 58.57025317383194 and to reduce it by 10% is 8.902859666282216.`
[/quote]

This doesn't seem that big a deal either. Assuming this gets deployed by the next halving, it 10% is only 0.15 BTC.

Maybe 150 minutes is a good enough number? There's no circumstance I can (currently) think of where that can introduce a bug. While at the same time it takes over 10 retarget periods to reduce difficulty by 10%.

-------------------------

AntoineP | 2024-12-23 15:53:31 UTC | #13

[quote="sjors, post:12, topic:1326"]
Maybe 150 minutes is a good enough number?
[/quote]

There is plenty of good enough numbers. Unless we have a good reason not to i'd say let's stick to the already proposed value of 600 seconds. I'll email the mining dev mailing list to ask if there is anything we're overlooking here.

-------------------------

sjors | 2024-12-24 08:03:59 UTC | #14

I would turn that round. We shouldn't introduce a soft fork rule that can be broken by accident, unless there's a really good reason to. The original motivation for the timewarp attack was to prevent a very destructive attack that could happen in a matter of weeks. That doesn't require a 600 second limit.

If the timewarp attack had been fixed with a 24 hour limit,  I don't think anyone would have proposed a subsequent soft fork to lower it to 600 seconds.

Above I linked to two actual mining pool software bugs that caused invalid blocks on testnet4 because they were using the system clock instead of the block template `nTime`. Bitcoin Core itself (briefly) had a bug where it could accidentally violate the timewarp rule. So that's three real bugs found with minimal testing. We have no idea how much (closed source, unmaintained) mining software is out there, and we can't expect every pool, individual mining farm and solo operator to thoroughly test their entire infrastructure on testnet4.

Most modern soft forks have used standardness to protect miners against accidentally producing an invalid block if they don't upgrade their node. They also need to upgrade their node to not accidentally mine on top of an invalid block. And as a community we rely on a very large majority to upgrade their node in order to safely deploy the soft fork network wide.

The timewarp fix with a 600 second limit deviates from that because it requires miners to also check their entire stack of mining software. Whereas a 150 minute threshold only requires them to upgrade their node.

Miners should of course, regardless of the soft fork, check their mining software for the above bugs (they only became visible because on testnet4 the 20-minute-difficulty-1 rule is exploited more aggressively than on testnet3, pushing MTP structurally past wall clock time). But that's an orthogonal issue.

-------------------------

zawy | 2024-12-24 11:46:34 UTC | #15

I aree with everything @sjors has said.

I don't see any reason 600 should be used.

The onus shouldn't be to prove there's a danger. We know there's a danger to more restrictions  breaking something.

I don't understand why 600 was proposed. How is being more restrictive beneficial?

Bitcoin should be like a rock upon which developers can build without getting rug-pulled like a Facebook API. Microsoft build its profitable evils on the back of a similar rock: every developer's program since 1981 still runs on it.  

If you break someone's software, you destroy far more systemic bitcoin value due to perception than the harm to the victims.

I regret giving walls of text on theoretical ideals if it distracted from @murch 's wise recommended fix that is the least restrictive and therefor safest. It's not even the same type of change as this one which has to specify a somewhat arbitrary number. 

In the distant past nullc's (even wiser?) recommendation on timewarp was "don't touch it". His argument was that it can be easily fixed if it happens and there's already a non-fixable fundamental break in the security (>50%) if it does. On testnet it's more of a risk that needs addressing. With a some imagination, leaving it alone on main chain could be a honey pot you want to keep.

-------------------------

AntoineP | 2024-12-24 15:18:03 UTC | #16

[quote="sjors, post:14, topic:1326"]
We shouldn’t introduce a soft fork rule that can be broken by accident, unless there’s a really good reason to.
[/quote]

This is a truism. I don't think it's necessary to say i agree.

[quote="sjors, post:14, topic:1326"]
The original motivation for the timewarp attack was to prevent a very destructive attack that could happen in a matter of weeks.
[/quote]

Again you only give one of the multiple motivations for fixing timewarp. I already called you up on that in a previous post. As i've [described in my writeup](https://delvingbitcoin.org/t/great-consensus-cleanup-revival/710#should-we-really-fix-it-3) as well as in private discussions i don't think it is the most likely scenario. It'd make more sense for miners to artificially increase the block rate thereby bringing available block space up and, all other things equal, fee rates down. With the current concentration of mining we shouldn't underestimate the odds of it happening. We should not keep in place the incentive for them to pull it off by making the fix too loose.

[quote="sjors, post:14, topic:1326"]
If the timewarp attack had been fixed with a 24 hour limit, I don’t think anyone would have proposed a subsequent soft fork to lower it to 600 seconds.
[/quote]

That's plausible because a soft fork is risky and expensive. Not because somehow 24 hours is a better value than 10 minutes.

[quote="sjors, post:14, topic:1326"]
Above I linked to two actual mining pool software bugs that caused invalid blocks on testnet4 because they were using the system clock instead of the block template `nTime`.
[/quote]

Those softwares are already broken today due to the MTP rule. "Existing broken software may produce invalid blocks under exceptional circumstances" is not a valid argument for looser bounds if said software can already produce invalid blocks with current rules under exceptional circumstances!

[quote="sjors, post:14, topic:1326"]
Bitcoin Core itself (briefly) had a bug where it could accidentally violate the timewarp rule.
[/quote]

What is this bug?

[quote="sjors, post:14, topic:1326"]
We have no idea how much (closed source, unmaintained) mining software is out there, and we can’t expect every pool, individual mining farm and solo operator to thoroughly test their entire infrastructure on testnet4.
[/quote]

I agree, but it is not on itself an argument for a looser value. Assuming you are still proposing to use 150 minutes instead, could you provide an argument for why this value would possibly better accommodate broken software we do not know about?

[quote="sjors, post:14, topic:1326"]
The timewarp fix with a 600 second limit deviates from that because it requires miners to also check their entire stack of mining software. Whereas a 150 minute threshold only requires them to upgrade their node.
[/quote]

How so? Could you be more specific? I don't think we've established that.

In conclusion, let me state i do not have a too strong opinion in favour of a 600 seconds grace period. It just makes sense to me and i don't think we should change it unless there is a good reason to.

There is good reasons for the fix to be neither too loose nor too tight. 600 seconds is a good sweet spot as it seems to get rid of the incentive for miners to artificially increase the block rate (with a 600 seconds grace period a 10% block rate increase [would take 212 periods](https://delvingbitcoin.org/t/timewarp-attack-600-second-grace-period/1326/11?u=antoinep)) while being loose enough that using higher values wouldn't materially decrease the probability of creating an invalid block. You have dismissed the reasons for avoiding a looser fix without providing compelling arguments why going for a 150 minutes grace period would lower risks.

-------------------------

zawy | 2024-12-26 10:00:35 UTC | #17

They can't increase the block rate to get 10% more unless they also have > 50% and privately mine for 212 periods to reliably keep the MTP held back. Publicly they'd need something like 99%. But if they do a private mine they get 50% more blocks per time than they would publicly after the 1st adjustment for as long as they mine. 

But they don't get the 10% gain for the same reason. Delaying 600 s in 1st cycle lowers the difficulty enough for him to get an extra block in the 2nd. But getting 1 more block means difficulty goes up in the 3rd cycle to offset the gain if he doesn't do the -600 in the next cycle. So he can only maintain getting 1 extra block per cycle. It's not an advantage that can accumulate. So he gets only 212 blocks which isnt important compared to getting 212 × 2016 excess blocks from doing the private mine (over a public mine). So it's a 0.05% gain over a private mine and a -3 hour limit is a 0.9% gain.

* cycle => blocks/cycle at 50% =>  blocks per 2 weeks
* 1 => 1008 (normal before attack) => 1008
* 2 => 2016 (private mine begin -3 hr timestamp) => 1008  (takes 4 weeks)
* 3=> 2016 (-3 hr) => took 2 weeks minus 3 hr due to previous stamp
* 4 => 2016 (-3 hr) => took 2 weeks minus 3 hr

He keeps the last timestamp in a cycle at current time. There's not a way to hold it back or push it forward to help. The only thing he gets over a private mine is more time at the end to mine 1 extra week than he otherwise could have. After 112 cycles he would get a gain of 2 weeks to get 2016 +18 blocks more blocks than doing nothing, less than 2% of the "excess" gains of just being a private mine ( 2016/2 * 112).

I see no reason to dismiss the concerns that -600 will break something. This is my 3rd request that someone explain how it helps.

-------------------------

sjors | 2025-01-03 12:41:43 UTC | #18

[quote="AntoineP, post:16, topic:1326"]
It’d make more sense for miners to artificially increase the block rate thereby bringing available block space up and, all other things equal, fee rates down. With the current concentration of mining we shouldn’t underestimate the odds of it happening. We should not keep in place the incentive for them to pull it off by making the fix too loose.
[/quote]

I don't think this is dangerous for the network for the numbers we're discussing, i.e. a 10% speedup after a sustained 51% attack for 59 difficulty periods. And why would a miner want to reduce their own fee revenue?

[quote="AntoineP, post:16, topic:1326"]
[quote="sjors, post:14, topic:1326"]
Above I linked to two actual mining pool software bugs that caused invalid blocks on testnet4 because they were using the system clock instead of the block template `nTime`.
[/quote]

Those softwares are already broken today due to the MTP rule. “Existing broken software may produce invalid blocks under exceptional circumstances” is not a valid argument for looser bounds if said software can already produce invalid blocks with current rules under exceptional circumstances!
[/quote]

But no miner ran into that bug so far, because it takes significant hash rate to push MTP above wall clock time and perform this attack. With the timewarp rule, griefing a miner that uses wall clock time only requires finding the last block in a retarget period and setting its time (a bit more than) 600 seconds in the future. Both attacks are free, but the latter only requires luck, no large percentage of hash power. 

[quote="AntoineP, post:16, topic:1326"]
[quote="sjors, post:14, topic:1326"]
Bitcoin Core itself (briefly) had a bug where it could accidentally violate the timewarp rule.
[/quote]

What is this bug?
[/quote]

https://github.com/bitcoin/bitcoin/pull/30681

The main change adds this to miner.cpp:

```cpp
if (consensusParams.enforce_BIP94) {
        // Height of block to be mined.
        const int height{pindexPrev->nHeight + 1};
        if (height % consensusParams.DifficultyAdjustmentInterval() == 0) {
            nNewTime = std::max<int64_t>(nNewTime, pindexPrev->GetBlockTime() - MAX_TIMEWARP);
        }
    }
```

The pull request description is focussed on a different scenario, but this (also) fixes the second griefing attack.

[quote="AntoineP, post:16, topic:1326"]
Assuming you are still proposing to use 150 minutes instead, could you provide an argument for why this value would possibly better accommodate broken software we do not know about?
[/quote]

Yes, 150 minutes makes the above griefing attack on broken miner software impossible, because the attacker can't put their timestamp more than 120 minutes in the future.

[quote="AntoineP, post:16, topic:1326"]
[quote="sjors, post:14, topic:1326"]
The timewarp fix with a 600 second limit deviates from that because it requires miners to also check their entire stack of mining software. Whereas a 150 minute threshold only requires them to upgrade their node.
[/quote]

How so? Could you be more specific? I don’t think we’ve established that.
[/quote]

Hopefully the above example illustrates this? With a 150 minute threshold there's no (additional) danger in having a bug in mining software that ignores the `nTime` value provided in the Bicoin Core template.  They might ignore it by accident like SRI https://github.com/stratum-mining/stratum/issues/1324, or they might have custom code that calculates `nTime` correctly under the current rules (but not accounting for the new timewarp rule).

-------------------------

sjors | 2025-01-03 14:42:50 UTC | #19

While trying to illustrate the griefing attack with a functional test, I found another bug :-) Will link to a fix PR here.

Update:

https://github.com/bitcoin/bitcoin/pull/31600

Aside from the fix it illustrates the griefing attack above.

-------------------------

AntoineP | 2025-01-03 17:05:17 UTC | #20

[quote="sjors, post:18, topic:1326"]
And why would a miner want to reduce their own fee revenue?
[/quote]

Miners care about the whole block reward, not only transaction fees. A higher block rate means they can claim more of the subsidy, and possibly more fees. Note also i said it would bring (all others things equal) fee *rates* down, i did not say anything about total fee revenues.

[quote="sjors, post:18, topic:1326"]
I don’t think this is dangerous for the network for the numbers we’re discussing, i.e. a 10% speedup after a sustained 51% attack for 59 difficulty periods.
[/quote]

I agree it would make it much less likely than with today's incentives, but i wouldn't be as confident as to rule it out entirely. Mining today is extremely concentrated, and the most part of miners' revenues (block subsidy) is decreasing exponentially. Of course it would not be marketed as a 51% attack, how about a "miner activated fee rate easing" whereby miners get more revenues and users get lower fees? Note also it's not strictly necessary to reorg out the blocks of honest miners to exploit this, especially with a large share of the network hashrate participating.

-------------------------

sjors | 2025-01-06 10:09:48 UTC | #21

@AntoineP exponential growth is always a problem if it goes on for long enough.  Is there some upper bound to the speedup or can blocks come in every second if the 51% attack persists?

-------------------------

AntoineP | 2025-01-06 11:24:13 UTC | #22

[quote="sjors, post:18, topic:1326"]
[quote="AntoineP, post:16, topic:1326"]
[quote="sjors, post:14, topic:1326"]
Bitcoin Core itself (briefly) had a bug where it could accidentally violate the timewarp rule.
[/quote]

What is this bug?
[/quote]

https://github.com/bitcoin/bitcoin/pull/30681
[/quote]

Wait, this is not a "bug in Bitcoin Core" this is a fix to the PR which changed block validation rules without adapting the block creation logic...

[quote="sjors, post:21, topic:1326, full:true"]
@AntoineP exponential growth is always a problem if it goes on for long enough. Is there some upper bound to the speedup or can blocks come in every second if the 51% attack persists?
[/quote]

I'm not sure what exponential growth you are referring to. Are you asking if in theory it's possible to extremely increase the block rate with any grace period value if the attack persists long enough? If so then yes. 

In theory the difficulty can always be brought down to 1 if you can constantly claim what took you 2 weeks took you 2 weeks + x seconds. See [the numbers i shared above](https://delvingbitcoin.org/t/timewarp-attack-600-second-grace-period/1326/11?u=antoinep). But i'm not sure how relevant it is: for instance with a 600 seconds grace period it would take hundreds of years to bring the difficulty down to 1.

-------------------------

sjors | 2025-01-06 12:29:30 UTC | #23

[quote="AntoineP, post:22, topic:1326"]
without adapting the block creation logic
[/quote]

Which is a bug in Bitcoin Core. It was caught before release. The second bug wasn't, it's fixed by 31600. It just illustrates how easy it is to screw things up when the grace period is only 600 seconds.

Update: it was a known omission before the original PR was merged, so maybe shouldn't count as a bug: https://github.com/bitcoin/bitcoin/pull/30647#issuecomment-2291734952

[quote="AntoineP, post:22, topic:1326"]
Are you asking if in theory it’s possible to extremely increase the block rate with any grace period value if the attack persists long enough? If so then yes.
[/quote]

Exactly. I'm not really worried about a one-off 10% increase in block speed. But compounded over time it's more worrying.

[quote="AntoineP, post:22, topic:1326"]
with a 600 seconds grace period it would take hundreds of years to bring the difficulty down to 1
[/quote]

Hundreds of years is enough time to come up with a plan, but serious problems could happen long before difficulty reaches 1. E.g. deep reorgs would be easier to execute.

If we take a difficulty reduction of 10x as the (arbitrary) definition of "bad", and if assume a 10% difficulty decrease every ~10 retarget periods, it takes 240 retarget periods to reduce to reach this "bad" situation. That's several years, probably enough to discuss and deploy a soft-fork to make it stricter. But not super comfortable either.  

Whereas with a grace period of 600 reaching "bad" takes 20 times longer, which seems very safe.

So this may be a good argument for keeping the number at 600 and to insist that pools either use the `min_time` field provided by Bitcoin Core, or implement their own code for the minimum time that takes both MTP and the new timewarp rule into account.

-------------------------

zawy | 2025-01-06 14:59:52 UTC | #24

[quote="AntoineP, post:22, topic:1326"]
In theory the difficulty can always be brought down to 1 if you can constantly claim what took you 2 weeks took you 2 weeks + x seconds.
[/quote]

Not if you want to win a 51% race. After claiming you took 2 wks + x, you're going to solve the next 2016 blocks in 2 wks - x if you maintain your hashrate. If you don't, you're not going to win chain work. To claim it took 2 wks + x in that round, you have to add 2x to to your 2 wks - x real time. You have only 1 x to manipulate without penalty and the other x you have to add to real time.  As those accumulate past 7200 seconds, no nodes will accept your blocks.

-------------------------

AntoineP | 2025-01-06 15:02:40 UTC | #25

Yes, that's why i started with "in theory". In practice i think any of the discussed values for the grace period make extreme drops in difficulty completely unlikely.

-------------------------

zawy | 2025-01-06 15:57:57 UTC | #26

There's no theory that allows an increase the block production rate. A difficulty drop won't accumulate more than 1 round unless hashrate drops. You want it to drop if that happens.  Difficulty can't drop more 1.2% in 1 and only 1 round if the reverse limit is 7200 and the attacker sets the last block to 7200 in the future.  There's no average decrease in difficulty because it will cause an increase in difficulty in the next round.

-------------------------

sjors | 2025-01-09 10:05:43 UTC | #27

If the difficulty drop doesn't accumulate then I flip back to my preference for a grace period high enough to avoid bugs, and which maintains the rule that the minimum `nTime` of a block is given by `MTP + 1`, without the need for a clock.

Update 2025-01-09: you don't need a clock to honor the timewarp rule.

-------------------------

sipa | 2025-01-07 15:18:24 UTC | #28

@zawy Imagine the proposed rule, that the first block in a period (i.e., one where $n = 0 \mod 2016$) must have a timestamp no more than 600s before its immediate predecessor (the last block of the previous period).

Call this constant $G = 600$ (grace period). Also introduce $P = 2016 \cdot 600$ (the period, 2 weeks).

For simplicity, let's replace the "timestamp must be strictly larger than MTP" rule with a "timestamp must not be below MTP" rule (i.e., we allow timestamp=MTP). This doesn't materially affect the attack, but means we don't need to increment by 1s every 6 blocks.

The following sequence of timestamps for periods is valid, starting at time $0$:
* Period 0: [$0$, $0$, $0$, ..., $0$, $0$, $P$].
* Period 1: [$P-G$, $P-G$, ..., $P-G$, $2P-G$].
* Period 2: [$2P-2G$, ..., $3P-2G$].
* Period 3: [$3P-3G$, ..., $4P-3G$],
* ...
* Period $k$: [$kP-kG$, ..., $(1+k)P-kG$].

Within each period, the difference between the first and last block timestamp is exactly $P$, so the difficulty will not adjust. However, the net timestamp increase per period is only $P-G$, i.e., 10 minutes less than 2 weeks.

So this allows attacker to permanently keep the block rate higher than intended, without incurring a difficulty increase cost.

The reason for the 600s grace period is that is compensates the effect of the off-by-one in the difficulty calculation (it only looks at how long 2015 blocks take, not 2016), which would otherwise result in (under constant hashrate) periods of 2 weeks *plus* 10 minutes. So with the proposed rule, a timewarp attacker can not cause a steady constant-difficulty block production rate of more than 2016 per two weeks.

There is another effect due to asymmetry of the Erlang distribution that results in another 10 minutes lengthening under constant hashrate. I forget why this isn't incorporated for compensation in the proposed GCC rule.

EDIT: ah, it's explained in footnote [1] in https://github.com/TheBlueMatt/bips/blob/7f9670b643b7c943a0cc6d2197d3eabe661050c2/bip-XXXX.mediawiki. The attacker can ignore the first effect but not the second.

-------------------------

zawy | 2025-01-07 21:47:10 UTC | #29

I don't understand how one of the effects can be ignored. Reference [1] switched from saying we're looking at the Erlang of 2015 blocks to 2016 blocks. It seems to imply the attacker received a double benefit, i.e. one by increasing timespan by 600, but then another by somehow going back in real-world time by 600 seconds to get 2016 * 600 seconds to mine instead of being stuck with 2015 * 600.

>[ for time taken to mine the **last 2015 blocks** ] ... IBT is 2016/2014 * 600. This is equivalent to 600 * E(2016 * 600/X) where X~ErlangDistribution(**k=2015**, λ=1/600). In the case of a miner deliberately reducing timestamps by 600 seconds on the difficulty-retargeting block, we are effectively changing the difficulty multiplier to (2016 / (time taken to mine the **last 2016 blocks** + 600)), or 600 * E(2016 * 600/(X + 600)) where X~Erlang Distribution(**k=2016**, λ=1/600), which is effectively targeting an inter-block time of ~599.9999 seconds.

Concerning if this means "difficulty drop can accumulate", yes, for the last block in this sequence he can use real time in for the last timestamp, then the next period will have a lower difficulty, but then the next period would be back to normal. He would have to do 168 periods with a 2 hr limit to drop difficulty to I think 1/2.

I think there's a lot more risk to making it strict. I can see only the very smallest of benefit in making it strict, but I can't estimate the risk that's being increased.

-------------------------

sipa | 2025-01-08 13:05:47 UTC | #30

Just so we're talking about the same thing, my points are:
* I believe a majority-hashrate attacker can use the timewarp attack to permanently increase the block rate beyond 1 per 600s on average, while keeping the hashrate constant, as long as they keep up the timewarping scheme. They can't keep increasing exponentially or anything like that.
* With a grace-period limiting how much time can go backwards between the last block of a period and the first block of the next period, we limit how much that that increased rate is. If the grace period is G seconds, then the attacker can maintain a block rate of roughly one per $\frac{2017 \cdot 600 - G}{2016}$ seconds, at constant hashrate. I don't remember why it's 2017 and not 2018 there (as would be expected if both the off-by-one and Erlang asymmetry are relevant), but I will try to simulate it.
* I have no opinion about when/where to enforce timestamp rules beyond bounding the amount of time the timestamp can go backwards between the last block of a period and the first block of the next period.

-------------------------

zawy | 2025-01-08 01:49:10 UTC | #31

* I understand and agree with your example.
* Right, I don't see why the BIP thinks it's 2017 instead of 2018. I bolded the unexplained jump in logic in the BIP that I didn't follow.
* You agree that making the limit 600 seconds is at least as safe as 7200?

Rather than 2017 / 2018, we're measuring a timespan across 2015 blocks, so via Erlang we expected 2014 blocks.  Therefore the adjustment is supposed to be "2014_blocks / 2015_expected_solvetimes".  But the code calculates "2016_blocks / 2015_times".  Allowing +600 in the denominator makes it 2016 / 2016.  But we should add 1 to the  numerator if we're going to add one to the denominator:  (2014+1) / (2015 + 1) = 2015 / 2016.  So 2016 / 2016 is too large.

For your example of subtracting 600 at the beginning and end, the difficulty is too much as always 2016 / 2015 instead of "staying constant" with 2014 / 2015, i.e. it's taking 600.6 seconds / block (2 weeks + 20 minutes).  So G = 3 * 600 in order to solve in 2 weeks minus 10 minutes?

-------------------------

sipa | 2025-02-07 23:23:09 UTC | #32

Let's say $c_i$ is the actual time at which block $i$ is found, and $t_i$ is its timestamp.

If the attacker (who is assumed to have 100% hashrate) follows this strategy:
* The first block in every period $p$ uses as timestamp the previous block's timestamp minus the grace period: $t_{2016p} = t_{2016p-1} - G$ (the minimum legal time according to the proposed rule).
* The 2014 middle blocks use the minimum legal time, which doesn't really matter as long as it's low enough: $t_{2016p+k} = 0$ for $0 < k < 2015$.
* The final block in the period uses the current time $t_{2016p+2015} = c_{2016p+2015}$.

Then the observed duration of the **2015** blocks in period $p$ (relevant for difficulty adjustment) is $t_{2016p+2015} - t_{2016p} = c_{2016p+2015} - c_{2016p-1} + G$, i.e. the time it took to mine **2016** blocks plus $G$.

The difficulty multiplier $m$ will thus be $m = \frac{P}{X + G}$, where $X$ is Erlang distributed with $k=2016$ and $\lambda=\operatorname{hashrate} \cdot \frac{\operatorname{target}}{2^{256}}$.

In a simulation with $G=600$ I get an effective average block interval under constant hashrate of 599.9997... seconds. With $G=7200$ I get 596.7211... seconds.

-------------------------

zawy | 2025-01-08 17:04:51 UTC | #33

OK, now I see. It's not that an adjustment for both Erlang and "2015 hole" aren't needed, it's that 600 seconds before the previous block isn't a 600 second lie, it's a 1200 second lie because we expected a timestamp 600 seconds after it.

-------------------------

sipa | 2025-01-08 18:05:21 UTC | #34

[quote="zawy, post:33, topic:1326"]
it’s a 1200 second lie because we expected a timestamp 600 seconds after it.
[/quote]

That's a great way to put it.

-------------------------

sjors | 2025-01-31 08:32:51 UTC | #35

See this comment: https://delvingbitcoin.org/t/great-consensus-cleanup-revival/710/66?u=sjors

Although @AntoineP now agrees with increasing the grace period, for reasons explained in the above comment, it might still be useful to investigate the following:

[quote="AntoineP, post:66, topic:710"]
Sjors then raises another concern about pool software ignoring the time in the Bitcoin Core provided block template and instead using wall clock time.
[/quote]

[quote="AntoineP, post:66, topic:710"]
Further, it’s not clear the timewarp fix is worsening things that much since such software is already vulnerable today to having a timestamp lower than 6 or more of its previous 11 blocks. Contrary to the timewarp fix this can happen on any block and does not even require an attack scenario (misconfigured clock).
[/quote]

Does anyone have a log of when their node saw blocks arrive over e.g. the past 10 years? Perhaps we can determine, or rule out, the presence of such a bug in the wild.

I don't know how though.

One problem with such analysis is that any block violating the MTP rule would not get propagated, so you wouldn't observe it. Even compact block relay checks the header: https://github.com/bitcoin/bips/blob/5333e5e9514aa9f92810cfbde830da79c44051bf/bip-0152.mediawiki#user-content-PreValidation_Relay_and_Consistency_Considerations

Another problem is that you don't know know the clock of the (pool) node that generated the block template.

A bug that ignores the template timestamp can be present in the pool software, the ASIC (stock or alternative) firmware or a proxy run by the miner.

If the bug is in pool software, we might be able to observe it live by studying live stratum jobs. This is easy on testnet4 today because MTP is ahead of wall clock time there. But not on mainnet.

Perhaps long term logs of such jobs exist, though again I'm not sure what to look for.

It seems a bit less likely to me that ASIC firmware ignores the timestamp provided in its stratum job. My own experience with old linux machines is that NTP doesn't always work out of the box, so someone would have noticed such a bug the hard way quickly.

I have no idea how stratum proxies work. Those might exist at the scale of a small mining farm that rarely finds an actual block, so it seems undetectable.

-------------------------

zawy | 2025-01-31 11:12:39 UTC | #36

[quote="sjors, post:35, topic:1326"]
It seems a bit less likely to me that ASIC firmware ignores the timestamp provided in its stratum job. My own experience with old linux machines is that NTP doesn’t always work out of the box, so someone would have noticed such a bug the hard way quickly.
[/quote]

Systemic use of NTP breaks security.

-------------------------

sjors | 2025-03-31 16:19:09 UTC | #37

I'm still trying to understand whether the difficulty decrease compounds. Take the following (same MTP simplification as above):

* Period 0: [$0$, $0$, $0$, ..., $0$, $0$, $P$].
* Period 1: [$P-G$, $P-G$, ..., $P-G$, $2P$].
* Period 2: [$2P-G$, ..., $3P$].
* Period 3: [$3P-G$, ..., $4P$],
* ...
* Period $k$: [$kP-G$, ..., $(1+k)P$].

So each period is stretched by $G$, causing the difficulty to drop every time. For $G$ = 2 hours, this would be 0.6% per retarget period. That translates to about 15% per year.

---

Note that any difficulty decrease accumulation can be reset by an honest miner finding the last block of any retarget period.

-------------------------

sipa | 2025-04-01 15:12:06 UTC | #38

That's fair, I think this may work, but even if it does, I don't believe this is a problem.

Given that the end-of-window time (which is bounded by the current time) goes up by $P$ per block, it means the real-time block production rate in your scheme is exactly $P$ per window, exactly the intended rate. So, from the perspective of validating nodes, there is no resource consumption attack.

And beyond that, indeed, this allows miners to reduce the difficulty while keeping the block rate constant. But a colluding group of miners is always able to do that: they can just **agree to all reduce their hashrate**, for any difficulty reduction they'd like. But, all that achieves is making it easier/cheaper for a competing miner who doesn't follow the scheme to take over.

-------------------------

AntoineP | 2025-04-01 15:02:02 UTC | #39

What Pieter said. As soon as you have a restriction on the timestamp of the first block in a period, miners can only "carry over" a fixed block rate increase. There is no compounding.

-------------------------

sipa | 2025-04-01 15:16:54 UTC | #40

Right. Exactly setting the timestamps isn't something a computationally-bounded miner can do because it is still a Poisson process.

If you try to adapt [Sjors' scheme](https://delvingbitcoin.org/t/timewarp-attack-600-second-grace-period/1326/37) to a real algorithm for deciding timestamps, I think you exactly get the scheme I posted [above](https://delvingbitcoin.org/t/timewarp-attack-600-second-grace-period/1326/32). And we simulated that: it has no compounding effect because the attackers' block production rate is affected by the difficulty adjustment they caused, providing negative feedback. But, again, even if there was a compounding effect it would not be a concern, as mentioned above, because it's at worst equivalent to attackers just choosing to reduce their hashrate.

-------------------------

Jonny1000 | 2025-04-01 15:34:25 UTC | #41

Thanks Antoine

I am just trying to wrap my head around why the 0.6% cannot compound.

If miners conduct the timewarp attack, then in epoch 2 they manipulate the difficulty down 0.6%

Then in epoch 3, since the difficulty is lower (But the hashrate is the same), the miners are expected to produce 0.6% more blocks. Then they use the fake timestamp trick again, shifting the difficulty down 0.6%, but that only offsets the impact of the 0.6% more blocks. So with this 2 hour rule, miners can only achieve a **one-off** 0.6% downward shift?

Am I thinking about that correctly?

Miners could of course just produce fewer blocks and keep shifting the difficulty downwards, but then that isnt really an attack with fake timestamps, that is just mining less

-------------------------

