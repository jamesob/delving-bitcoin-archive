# Default signet reorg history — was the fork at height 287766 (depth 19) deliberate?

pramodkandel | 2026-06-05 11:46:26 UTC | #1

We run a Bitcoin L2 and use the default signet as our primary testnet, so I'm trying to understand how often and how deep the default signet reorgs.

On a fork-observer instance I found a fork at common-ancestor height **287766** with a competing branch **19 blocks deep**, losing tip at height 287785 (`0000000024ff924ff932668d497bba7da9157559a68d9c87d2f28d22e5e4a001`).

Questions:

1. Was the 287766 reorg a deliberate exercise, or operational (e.g. concurrent signing instances reconciling)?
2. Is there any historical record of default-signet reorgs anywhere --- a log, a feed, a writeup --- or is running one's own monitor the only way to capture them going forward?
3. Should users treat periodic reorgs as expected behavior on the default signet?

Mainly trying to figure out whether deep reorgs here are a designed feature, a rare incident, or something in between, so we can test our stack against a realistic worst case.

-------------------------

ajtowns | 2026-06-05 11:56:51 UTC | #2

Going from memory, that would be due to my internet going down, at which point kalle's miner ("signet-1") took over. The fork became visible when I went online again with a longer/more-work chain, but at that point I manually invalidated my chain so that the chain that had been public would win out.

![signet-old-reorg|690x288](upload://n2lG33sLqhTMeowXseKsW8l2rma.png)

There's https://github.com/ajtowns/stale-blocks-signet tracking some of the recent stale blocks.

Regular reorgs have long been intended on signet, see for example:

https://gnusha.org/pi/bitcoindev/83272afb-ed87-15b6-e02c-16bb1102beb4@gmail.com/

I haven't done anything to try to make deep reorgs happen on any regular basis; I think with my [current setup](https://delvingbitcoin.org/t/regular-signet-reorgs/2549) seeming to be working okay now, I could possibly make deeper ones happen automatically too. Depends on if there's a desire to be able to force double-spends to occur as well; that might be trickier to automate sensibly.

-------------------------

pramodkandel | 2026-06-05 14:27:28 UTC | #3

Thank you! That clears things up : ) It's reassuring to know it was not a designed deep reorg.

On that point -- at least for our use case, it'd be great to keep reorgs from getting too deep. Signet's appeal for us is that it mimics mainnet -- predictable blocks and mainnet-like forks are basically why we picked it over Testnet4 -- so occasional shallow reorgs are fine, but deep ones on the default net would defeat the point for us. 

We may want to test deep-reorg handling, but we'd rather opt into it rather than have the shared default surprise us.

-------------------------

0xB10C | 2026-06-05 14:30:49 UTC | #4

[quote="pramodkandel, post:3, topic:2591"]
We may want to test deep-reorg handling, but we’d rather opt into it rather than have the shared default surprise us.
[/quote]



A counter point might be: There could be a deep-ish reorg on mainnet anytime and you wouldn't have a chance to opt-in (or out) of that. So better to test these a little more frequently on Signet?

-------------------------

pramodkandel | 2026-06-05 14:41:56 UTC | #5

That's a good point, but at least for really deep reorgs, which would not be Mainnet-like behavior, it would be nice to be able to opt-out. I like this idea (quoted from the email) -

> AJ proposed to allow SigNet users to opt-out of reorgs in case they
> explicitly want to remain unaffected. This can be done by setting a
> to-be-reorged version bit flag on the blocks that won't end up in the
> most work chain. Node operators could choose not to accept to-be-reorged
> SigNet blocks with this flag set via a configuration argument.

-------------------------

ajtowns | 2026-06-05 14:43:48 UTC | #6

I think if you wanted to really test deep reorg behaviour on mainnet, you'd want to keep mining both sides of the split for an extended period to mimic miners/pools who were being reorged out by an attack would probably use "invalidateblock" to attempt to save the chain they'd already invested in. Having people actively attempting to do replay protection across the split and potentially having hashpower on each side of the fork change as available fees change etc would give fairly different properties vs the current "let's invalidate some blocks and re-mine them as fast as we can, and then see what happens" thing I've got implemented.

To make sure we're on the same page, I presume the border between "normal" and "deep" reorgs is somewhere around 6-10 blocks?

(If you go up to 288 blocks, that interacts with NODE_NETWORK_LIMITED, and somewhere around/above that number potentially breaks pruned nodes by going back further than the minimum 550MB of block data)

-------------------------

pramodkandel | 2026-06-05 15:50:49 UTC | #7

[quote="ajtowns, post:6, topic:2591"]
I presume the border between "normal" and "deep" reorgs is somewhere around 6-10 blocks?

[/quote]

That sounds about right. In our protocol, we assume 6.

-------------------------

