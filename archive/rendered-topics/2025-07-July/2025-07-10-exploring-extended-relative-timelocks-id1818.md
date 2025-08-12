# Exploring Extended Relative Timelocks

pyth | 2025-07-10 04:50:02 UTC | #1


I’ve been discussing about feedbacks from Liana users with some folks at Wizardsardine about Bitcoin’s 65535-block limit for relative timelocks. Some users are curious why this limit exists and have expressed interest in longer timelocks. I’ve been experimenting with an idea to extend timelocks and would love your thoughts—*not* advocating for a soft fork, just exploring the concept to gauge its feasibility and implications.

## Context and Experiment
I looked into extending relative timelocks to ~10 years by:
- Using bit 21 in `nSequence` to signal a "long" timelock.
- If bit 21 is set and bit 22 is unset, the timelock unit is interpreted as 8 blocks, multiplying the duration by 8.
- This seems to require minimal changes to Bitcoin Core (though I’m not deeply familiar with the Core codebase, so I could be missing complexities). It would be a soft fork, as older nodes would ignore the flag and enforce a timelock 8 times shorter.

I also created a tool to craft addresses and transactions for a simple "anyone can spend after timelock" policy to test this. you can check out the experimental code and tool here:
- Code change: https://github.com/bitcoin/bitcoin/commit/a9187117ccf7224297a74f3dfebc2354ad9fe6e2
- Tool: https://github.com/pythcoiner/csv2

## Discussion Points
- **Use cases**: Are there compelling use cases for ~10-years timelocks?
- **Risks**: What risks or edge cases might arise? Are there consensus or compatibility concerns I’m overlooking?
- **Alternatives**: Can we address the (potential) need for longer relative timelocks without a soft fork ?
- **Implementation**: Does the bit 21 flag with 8-block units seem reasonable, or are there better ways to do this?

To be clear, I’m not pushing for a soft fork—just curious about the community’s take on this idea. I don’t have extensive knowledge of the Bitcoin Core codebase, so I’d especially appreciate feedback on the technical side. Thanks for any insights you can share!

-------------------------

fjahr | 2025-07-10 21:18:02 UTC | #2

Unless you are convinced that quantum computers will never work, I don't think you will want to do this before there isn't a way to make the coins quantum safe at the same time as locking them for an extended amount of time. But even when there is a concept to do that, so much can change over 10 years, so I wouldn't recommend anyone to lock their coins for longer than ~2 years. 

I am not an expert on quantum computers, I just think the chance is definitely non-zero to see them within our lifetimes and for me the use cases for extended time locks don't seem compelling enough to risk loosing all of the funds if quantum attacks become possible.

-------------------------

stevenroose | 2025-07-14 09:29:43 UTC | #3

[quote="pyth, post:1, topic:1818"]
**Use cases**: Are there compelling use cases for ~10-years timelocks?
[/quote]

I think the question is **are there compelling use cases for relative timelocks over the current limit?** and I think the answer is very clearly yes. Whether we move to a 10 years limit is kind of an implementation detail. I think 10 years is long, but if the alternative is 5 years (multiplier of 4), I'd go with 10 years as well.

[quote="pyth, post:1, topic:1818"]
What risks or edge cases might arise? Are there consensus or compatibility concerns I’m overlooking?
[/quote]

I think most risks concern people locking funds for too long, until a point that some future soft fork might affect their ability to spend their coins. People will mention quantum computers etc. Since doing so is currently already possible and easy with absolute locktimes, I don't think they are a good argument against extending the relative timelock limit.

[quote="pyth, post:1, topic:1818"]
Can we address the (potential) need for longer relative timelocks without a soft fork ?
[/quote]

You can always pre-sign a series of relative timelock txs! /s 

I don't think it's possible in a reasonable and manageable way without a softfork. Some covenants could be used to build a contract that forces you to go through multiple CSV periods, but first of all covenants also require a softfork and second it's definitely both ugly and wasteful in on-chain space.

[quote="pyth, post:1, topic:1818"]
Does the bit 21 flag with 8-block units seem reasonable, or are there better ways to do this?
[/quote]

I'm surprised with the elegance of this proposal. Of course the choice for 8 year blocks is arbitrary, but doing this keeps existing 1-block granularity possible while adding additional space.

One might consider if 10 years is long enough, we'd better avoid having to do this dance again some time in the future. 16-block units gives us 20 years. I'm quite indifferent tbh.

One consideration you could make is to not just take plain 8-block units, but 8-block units after the first 65535 blocks. That both gives you an extra ~1 year and avoids that there are two different ways to represent some relative timelocks.

---

As someone for whom the 1 year limit is definitely a reason I'm not using Liana-like inheritance constructions yet, I'm excited about this proposal!

-------------------------

pyth | 2025-07-15 05:52:08 UTC | #4

[quote="fjahr, post:2, topic:1818"]
at the same time as locking them for an extended amount of time.
[/quote]

  just a note about _locking_ coins: in constructions like Liana your coins are never locked by policy, there is a mandatory (non-timelocked) spending path that let you spend you coins _without timelock_. The only way of getting your coins _locked_ is by loosing ability to fullfill the condition(s) of this primary path, but this means your coins are lost if you compare with more "standard" constructions like single/multi signatures schemes.

-------------------------

pyth | 2025-07-15 06:09:14 UTC | #5

[quote="stevenroose, post:3, topic:1818"]
One consideration you could make is to not just take plain 8-block units, but 8-block units after the first 65535 blocks.
[/quote]

I've considered this, the main point I have "against" this, is in that case we have to manage 2 types of timelocks (long/short), thus increasing the bug surface (and I dont expect users cares about sub 8 blocks granularity in such constructions)

-------------------------

pyth | 2025-07-15 06:10:54 UTC | #6

[quote="stevenroose, post:3, topic:1818"]
and avoids that there are two different ways to represent some relative timelocks
[/quote]

there is already two ways to represent many values by using bit 22 ^^

-------------------------

pyth | 2025-07-15 06:13:47 UTC | #7

[quote="stevenroose, post:3, topic:1818"]
One might consider if 10 years is long enough, we’d better avoid having to do this dance again some time in the future. 16-block units gives us 20 years. I’m quite indifferent tbh.
[/quote]

8 blocks was an arbitrary choice, I'm also quite indifferent about the value itself

-------------------------

stevenroose | 2025-07-15 15:04:53 UTC | #8

[quote="pyth, post:5, topic:1818"]
I’ve considered this, the main point I have “against” this, is in that case we have to manage 2 types of timelocks (long/short), thus increasing the bug surface (and I dont expect users cares about sub 8 blocks granularity in such constructions)
[/quote]

Oh so you're idea is that it will be easier to just deprecate the old timelock system and entirely use this one, assuming no one will care about sub-8 block granularity? Bold idea. But definitely not crazy.

-------------------------

pyth | 2025-07-16 02:02:26 UTC | #9

[quote="stevenroose, post:8, topic:1818"]
Oh so you’re idea is that it will be easier to just deprecate the old timelock system and entirely use this one, assuming no one will care about sub-8 block granularity? Bold idea. But definitely not crazy.
[/quote]

I'd not say deprecate, I think it can be usefull in some constructions, but like in a Liana context, I really believe nobody uses sub-8 blocks granularity (or better say only uses for test purposes), so in our case it's a no brainer to just change the flag in our wallet logic & change the factor applied to the slider used to select the timelock in our UI if one day we have to upgrade. But no strong opinion.

-------------------------

scgbckbone | 2025-07-16 08:34:31 UTC | #10

[quote="stevenroose, post:3, topic:1818"]
I don’t think they are a good argument against extending the relative timelock limit
[/quote]

agree, current max is too low

one *nit* argument against is that you can do this already with CLTV, when the lock expires, instead of spend (refreshing relative lock), you do sweep to different wallet where you have ability to set nLockTime value again

-------------------------

scgbckbone | 2025-07-16 09:08:20 UTC | #11

[quote="scgbckbone, post:10, topic:1818"]
different wallet
[/quote]

same keys, updating just nLockTime in descriptor

-------------------------

pyth | 2025-07-16 09:27:16 UTC | #12

[quote="scgbckbone, post:10, topic:1818"]
f spend (refreshing relative lock), you do sweep to different wallet where you have ability to set nLockTime value again
[/quote]
afaik BitcoinKeeper uses this kind of constructions, but it comes with some inconvenients, every time you "roll" your timelock you create a new descriptor and then you need:
 - to make a proper backup of this descriptor
 - to register/validate your policy on your signing device(s)

-------------------------

stevenroose | 2025-07-16 10:28:56 UTC | #13

[quote="pyth, post:12, topic:1818"]
every time you “roll” your timelock you create a new descriptor and then you need
[/quote]

Unless we extend the descriptor spec to somehow support semantics like "next round year at least 5 years in the future" and that wallets when scanning expand into all the year marks since the birthday of the wallet (or just last 20 or so). Not saying I'm in favor of something like this, but the descriptor issue could be worked around.

-------------------------

pyth | 2025-07-16 11:27:15 UTC | #14

technically yes, but on an other topic, I think we should try to make the policy/miniscript more "readable" in order to be really validated on the signing device in a sane way by a lambda user, and imho adding more argument to the policy semantic push to the wrong direction ...

-------------------------

pyth | 2025-07-19 00:58:18 UTC | #15

an other pain point I see with using absolutes timelock is it's very likely you'll not be able to spend several inputs with a different timelocks in a same transaction in a safe way with some (or maybe all?) signing devices actually supporting miniscript: as this outputs are paying to a script generated from different descriptors, some inputs will be seen by the devices as an non-owned input whereas they are indeed owned...

-------------------------

kloaec | 2025-07-21 09:22:39 UTC | #16

[quote="pyth, post:1, topic:1818"]
* It would be a soft fork, as older nodes would ignore the flag and enforce a timelock 8 times shorter.
[/quote]

Already answered by Pyth, but just to clarify:
Timelocks aren't really used to lock coins, but to add different conditions in Script that become valid only in the future.
Typically:
- collaborative spend with no timelock
- unilateral spend with timelock
for multiparty constructions

or, in the case of Liana:
- regular spend condition without timelock (let's say a 2-of-2 multisig)
- backup spend condition with a timelock (let's say a 2-of-3 with a 3rd party key, or even a single sig key agent)

-------------------------

kloaec | 2025-07-21 09:28:45 UTC | #17

I don't believe the absolute timelock fits well in dead-man-switch situation, and i despise stacking multiple wallets (including past ones) as we currently cannot "disable" an old wallet in Bitcoin. Users will do mistakes, address reuse of old wallets, fuck up their descriptor backups, and until we solve the xpub reuse properly, it's also bad for privacy and compatibility between wallets (need a state to make sure no key/derivation is reused)

-------------------------

scgbckbone | 2025-07-21 12:55:15 UTC | #18

having larger relative lock is definitely less convoluted than CLTV approach (I mentioned it's a nit argument)

I'll support this proposal any day.

-------------------------

rafael | 2025-07-21 13:45:12 UTC | #19

[quote="pyth, post:1, topic:1818"]
It would be a soft fork, as older nodes would ignore the flag and enforce a timelock 8 times shorter.
[/quote]

Wouldn't it be a risk to not move the coins after the 1x period has passed, since older nodes would still be able to propagate a transaction spending it? And miners could eventually mine it.

-------------------------

stevenroose | 2025-07-23 11:53:33 UTC | #20

Generally new consensus rules are only activated once a large base of miners has flagged support. This would ensure that no tx violating the new rules can be mined anymore.

This new scheme also uses a different bit in the sequence field, so for old nodes, the sequence value would not have any meaning, they would not confuse it with the current relative timelock rules.

-------------------------

chris | 2025-08-03 11:57:18 UTC | #21

When onboarding users to Liana, the most common question I get is whether the timelock can be extended beyond the current 15 months. Personally, I think extending it to 5 or even 10 years would be ideal it would resolve several UX pain points and better align with long-term custody and inheritance needs.

-------------------------

sjors | 2025-08-04 11:24:34 UTC | #22

I wish the original [BIP68](https://github.com/bitcoin/bips/blob/31450c3dbde336cd8503197d78e06ece46c28d7a/bip-0068.mediawiki) authors had used just one more bit.

Why not just do that and include bit 16? So the mask goes from `0x0000ffff` to `0x0001ffff`. If bit 16 set, old software would allow spending strictly earlier, so it's a soft fork. The downside is that if someone set this bit accidentally they'll have to wait a decade longer than expected.

-------------------------

AntoineP | 2025-08-04 16:05:38 UTC | #23

[quote="sjors, post:22, topic:1818"]
Why not just do that and include bit 16?
[/quote]

Because Lightning already uses these bits to store obfuscated commitment transaction numbers. The nSequence bits with no consensus meaning are i think in practice lost as an upgrade hook. I think that if someone wanted to work on extending relative timelocks they should use the Taproot annex, and why not also bundle per-input absolute timelocks under a single "improved timelocks" proposal.

-------------------------

sjors | 2025-08-05 07:37:22 UTC | #24

[quote="AntoineP, post:23, topic:1818"]
Lightning already uses these bits to store obfuscated commitment transaction numbers
[/quote]

Ah, this is burried [deep in BOLT 03](https://github.com/lightning/bolts/blob/6c5968ab83cee68683b4e11dc889b5982a2231e9/03-transactions.md#commitment-transaction):

> the lower 24 bits of the obscured commitment number

It would be good if Lightning folks opened a BIP similar to [BIP320](https://github.com/bitcoin/bips/blob/3e15178a43cf36f6e242bc3ff8caa659f22acb1c/bip-0320.mediawiki) to document this. cc @rustyrussell ?

-------------------------

pyth | 2025-08-12 03:35:40 UTC | #25

Interesting remark, thanks.

I’ve take a look about nSequence usage onchain with this \[tool\]([nsequence_runner](https://github.com/pythcoiner/nsequence_runner)) and what I can observe is:

* there is no nsequence bit that have not been used at least once 
* the combination of bit31 = 0 && bit 22 = 0 && bit 21 = 1 && (nSequence & 0xFFFF) > 0 &&  the spending script contains an OP_CSV (only looked at segwitv0 & taproot)

I’ve no strong opinion on how it should be interpreted btw

-------------------------

