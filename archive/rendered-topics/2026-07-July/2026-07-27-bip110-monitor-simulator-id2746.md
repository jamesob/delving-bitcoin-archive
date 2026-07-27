# BIP110 Monitor & Simulator

orangesurf | 2026-07-27 08:25:24 UTC | #1

**BIP110 Situation Monitor**
I have made a [BIP110 situation monitor ](https://bip110.orange.surf/live.html)which tracks live miner signalling data and shows the chain tip of both a core node and bip110 node to flag forks. It also has a visual guide to the [activation](https://bip110.orange.surf/activation.html) paths & [forking dynamics](https://bip110.orange.surf/forking.html). Finally it has a [simulator](https://bip110.orange.surf/simulator.html) which allows you to explore the generic softfork chainsplit dynamics. The aim is to educate, particularly targeted at those who have (imo) been misled by proponents of this softfork into thinking that it is 'the safe option'. Any suggestions would be greatly appreciated!

**My Expectation**
*While the above tool attempts to remain as balanced and neutral as possible, I have shared my expectation for what will happen (repeated below). I would be interested in hearing if others expect the same thing.*

BIP110 won't get anywhere near the 55% signalling window in this difficulty window. This will triggering mandatory signalling from block 961,632. The first block most likely won't signal, and therefore BIP110 nodes will ignore it, while the standard chain will be extended.

After an extended period of time a BIP110 miner will extend a stale tip, creating a fork with a less work chain. BIP110 nodes and miners will follow this less work chain creating a persistent fork, before ultimately capitulating or implementing a proof of work change.

**Background on the BIP110 (Reduced Data Temporary Softfork)**
*I have made a [PR](https://github.com/bitcoin-cap/bcap/pull/81/changes/ea058f1c69779149fe87e50f22d2a4a8eccb5d4e#diff-b335630551682c19a781afebcf4d07bf978fb1f8ac04c6bf87428ed5106870f5R936) to add the following details of the BIP110 softfork to the BCAP project, I would appreciate it if anyone has time to review.*

In late 2025 a UASF was proposed which was assigned BIP110 in December 2025. The aim of the UASF was to "temporarily limit the size of data fields at the consensus level" in order to correct what the author described as "distorted incentives caused by standardizing support for arbitrary data".[\[1\]](https://github.com/bitcoin/bips/blob/9783d61f1b9c81231581fee026c8e8cb9499d265/bip-0110.mediawiki)

The soft fork rules invalidate transactions with several data-carrying constructions for 52,416 blocks (roughly one year). UTXOs created before activation are exempt, and once the deployment expires all UTXOs are intended to be unrestricted. At the time of writing, the outcome is unresolved, however miner signaling remains far below threshold ahead of the mandatory signaling window.

The claimed temporary nature of the soft fork is highly contested. Critics argue that consensus rules cannot meaningfully bind future behavior, and that any node that continues enforcing past expiry will treat the reversion as a rule-loosening event (a hard fork from its perspective). Proponents claimed that the expiry is encoded as part of the BIP110 soft fork, so enforcing nodes can relax the rules in lockstep in a manner that is not a hard fork.

The activation mechanism, implemented in a fork of Bitcoin Knots and later shipped in Bitcoin Knots [\[2\]](https://github.com/bitcoinknots/bitcoin/compare/29.x-knots...dathonohm:bitcoin:uasf-modified-bip9) combines features of BIP8 and BIP9. Miners signal via BIP9-style version bits (bit 4), but there is no timeout and no FAILED state; instead a BIP8-like max_activation_height guarantees activation (for enforcing nodes only). Should the threshold not be reached earlier, mandatory signaling is enforced from block 961,632 to 963,647, during which enforcing nodes reject non-signaling blocks.

Unlike standard BIP9's 95% threshold, BIP110 requires only 55% (1,109 of 2,016 blocks). The authors stated rationale is that rejecting data storage is "a matter of urgency" and that a temporary, expiring soft fork justifies a lower bar.[\[3\]](https://github.com/bitcoin/bips/blob/9783d61f1b9c81231581fee026c8e8cb9499d265/bip-0110.mediawiki#deviations-from-bip9) This threshold measures miner signaling only, which as mentioned previously can be spoofed. This would leave enforcing hashrate in the minority at the moment enforcement begins.

The BIP110 soft fork is a consensus change not merged into Bitcoin Core, activated through an alternative client, with two distinct points where a chain split can begin, namely the start of mandatory signaling, and rule enforcement at activation.

As with any soft fork, miners who do not enforce the new rules risk having their blocks reorged if non-enforcing hashrate is the minority. The validity is asymmetric in that blocks produced by enforcing miners remain valid under non-enforcing rules, while the reverse does not hold, so a majority-enforcing chain would be expected to overtake and reorg non-enforcing blocks, and non-enforcing nodes would follow it automatically.

Proponents have cited this to argue that enforcing BIP110 is the safer choice for miners and node operators, but the argument inverts when enforcing hashrate is a minority. The enforcing chain is then expected to remain the less-work chain, non-enforcing miners face little expected reorg risk from it, and enforcing nodes, which reject the most-work chain as invalid, fork themselves onto a low-hashrate minority chain, producing a persistent split rather than a resolution by work.

-------------------------

orangesurf | 2026-07-27 08:21:04 UTC | #2

PS. I hope this is the correct category, and that it's ok to bundle these things together, if not please let me know!

-------------------------

0xB10C | 2026-07-27 14:34:54 UTC | #3

Cool stuff!

We've been discussing BIP-110 monitoring and what data to collect here too: https://bnoc.xyz/t/brainstorming-what-data-to-collect-and-monitor-during-the-bip-110-bip-300-forks-in-august-2026/139 - feel free to chime in if you have anything to add. I'm running a Core and a BIP-110 node attached to a fork-observer instance too and plan to live-stream the (expected) split when mandatory signaling starts.

-------------------------

AntoineP | 2026-07-27 14:59:58 UTC | #4

[quote="orangesurf, post:1, topic:2746"]
After an extended period of time a BIP110 miner will extend a stale tip, creating a fork with a less work chain. BIP110 nodes and miners will follow this less work chain creating a persistent fork, before ultimately capitulating or implementing a proof of work change.
[/quote]

I'm curious why you expect a miner would waste a couple hundreds of thousands of dollars worth of work extending a dying chain? Ocean temporarily subsidizing them for it?

-------------------------

