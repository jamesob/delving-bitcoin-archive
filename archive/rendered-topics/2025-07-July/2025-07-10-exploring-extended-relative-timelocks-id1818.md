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

