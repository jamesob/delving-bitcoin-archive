# Disclosure: Critical vulnerabilities fixed in LND 0.19.0

morehouse | 2025-12-04 20:02:58 UTC | #1

One denial-of-service and two theft-of-funds vulnerabilities were fixed in LND 0.19.0.  Users should immediately upgrade to [LND 0.19.0](https://github.com/lightningnetwork/lnd/releases/tag/v0.19.0-beta) or later protect their funds.

## The Infinite Inbox DoS

Large internal queue sizes and an unrestricted incoming connection policy enabled attackers to quickly exhaust LND's available memory and cause it to crash or hang.

More details are provided in the corresponding [blog post](https://morehouse.github.io/lightning/lnd-infinite-inbox-dos/).

##  The Excessive Failback Exploit #2

A variant of the [previously disclosed](https://delvingbitcoin.org/t/disclosure-lnd-excessive-failback-exploit/1493) excessive failback bug could still be exploited to steal funds from LND nodes.  The variant was discovered while drafting an [update](https://github.com/lightning/bolts/pull/1233) to BOLT 5 that was intended to help prevent similar vulnerabilities in the future.

More details are provided in the corresponding [blog post](https://morehouse.github.io/lightning/lnd-excessive-failback-exploit-2/).

## The Replacement Stalling Attack

Weaknesses in LND's sweeper system enabled an attacker to stall LND's attempts at claiming expired HTLCs on chain.  After stalling for 80 blocks, the attacker could steal essentially the entire channel balance.  This vulnerability was discovered during code review of LND's sweeper rewrite in 2024.

More details are provided in the corresponding [blog post](https://morehouse.github.io/lightning/lnd-replacement-stalling-attack/).

-------------------------

AntoineP | 2025-12-05 15:24:40 UTC | #2

Thanks for your sustained security work in this area, it's hard to quantify how many bullets are dodged when vulnerabilities are responsibly disclosed and patched before they become an opportunity for bad actors. Especially as a critical issue (like those or like previous ones you've reported) being exploited would likely have repercussions beyond the affected LND users, as it would cast doubts on the Lightning industry more generally.

Some of the timelines mentioned here are concerning, in particular for vulnerabilities that enables theft of users funds.

-------------------------

instagibbs | 2025-12-05 15:33:08 UTC | #3

[quote="morehouse, post:1, topic:2145"]
the attacker could steal essentially the entire channel balance

[/quote]

If the channel allows the whole balance as concurrent HTLCs? (I’m not sure if LND caps this by default)

-------------------------

morehouse | 2025-12-05 16:12:51 UTC | #4

[quote="instagibbs, post:3, topic:2145"]
If the channel allows the whole balance as concurrent HTLCs? (I’m not sure if LND caps this by default)
[/quote]

That's correct.  The protocol provides the `max_htlc_value_in_flight_msat` channel parameter to limit this.  But LND's [default behavior](https://github.com/lightningnetwork/lnd/blob/a76f22da9d167bb456c0828c3c723c6e41e1f01d/server.go#L1599-L1605) is to impose no limit.  And AFAICT, there's actually no way to override that default behavior in LND.

-------------------------

ariard | 2025-12-06 01:31:02 UTC | #5

Thanks for the disclosures – Browsed over the Replacement Stalling Attacks one, if my memory is correct on this subject, I think Bastien Teinturier was the first back in 2021 sharing with few lightning devs the idea of “cat-and-mouse” attacks, i,e adversarially aggregating and disaggregating `option_anchors` second-stage HTLCs transactions. Sounds what you found is one variant of those “cat-and-mouse” attacks, but this is nothing very specific to LND.

About disclosure timeline, 90-days is generally the basic infosec embargo timeline. It can be a bit dry, assuming it’s a “simple” bug that can be fixed solely by the implementation (so no cross-layer vulnerability), letting a 4-6 months sounds more generous and in line with the usual release cycles of Lightning implementations. It’s really depends on the type of vulnerability, however the drawback with an isolated release with only 2 or 3 lines of changelog, with one altering some critical code paths, is to make it too much obvious there is “covert fix” somewhere and lowering the bar for any external actor to know what to look for to reverse-engineer.

On all the logic area you’re pointing you, i.e claimable outputs detection, generation of claims transactions, fee selection and scheduled rebroadcast of the claims of transactions, it has been floated few times the idea, at least since 2020, to write a specific BOLT encompassing all this flow with the do and don’t. While there is no necessity at all of inter-compatibility among the Lightning implementations on logic, despite being one of the most critical area for channels funds security, somehow Lightning implementations kept put one’s foot in one’s mouth on this flow. An (incomplete) [experience](https://github.com/discreetlogcontracts/dlcspecs/blob/9cd9148938c616690c79d99ec6f330e213c246c5/Non-Interactive-Protocol.md) has been attempted for the DLC spec on what such piece of recommendations can cover.

Otherwise, we can also keep eating popcorns and making laughs of Lightning implementations being insecure again and again on their on-chain logic management.

-------------------------

morehouse | 2025-12-06 15:31:27 UTC | #6

[quote="ariard, post:5, topic:2145"]
Thanks for the disclosures – Browsed over the Replacement Stalling Attacks one, if my memory is correct on this subject, I think Bastien Teinturier was the first back in 2021 sharing with few lightning devs the idea of “cat-and-mouse” attacks, i,e adversarially aggregating and disaggregating `option_anchors` second-stage HTLCs transactions. Sounds what you found is one variant of those “cat-and-mouse” attacks, but this is nothing very specific to LND.
[/quote]

The general weakness in LND was that transaction fees could be manipulated to stay too low, and the reaggregation+rebroadcast period was infrequent (once every two blocks).  As a result, an attacker could pull off the attack for cheap.  At the time I looked at other implementations and found their fees couldn't be manipulated this way, and they generally responded to double spends immediately after confirmation.  But I think there certainly is [much that can be improved](https://morehouse.github.io/lightning/lnd-deadline-aware-budget-sweeper/) about other implementations' fee bumping strategies to protect against replacement cycling and similar attacks in general.

[quote]
About disclosure timeline, 90-days is generally the basic infosec embargo timeline. It can be a bit dry, assuming it’s a “simple” bug that can be fixed solely by the implementation (so no cross-layer vulnerability), letting a 4-6 months sounds more generous and in line with the usual release cycles of Lightning implementations. It’s really depends on the type of vulnerability, however the drawback with an isolated release with only 2 or 3 lines of changelog, with one altering some critical code paths, is to make it too much obvious there is “covert fix” somewhere and lowering the bar for any external actor to know what to look for to reverse-engineer.
[/quote]

Yes, though 90 days may be a bit aggressive when so much money is at risk.  Especially for LND, which has historically had a lot of upgrade friction.  Perhaps 6 months is a better disclosure timeline.

[quote]
On all the logic area you’re pointing you, i.e claimable outputs detection, generation of claims transactions, fee selection and scheduled rebroadcast of the claims of transactions, it has been floated few times the idea, at least since 2020, to write a specific BOLT encompassing all this flow with the do and don’t. While there is no necessity at all of inter-compatibility among the Lightning implementations on logic, despite being one of the most critical area for channels funds security, somehow Lightning implementations kept put one’s foot in one’s mouth on this flow. An (incomplete) [experience](https://github.com/discreetlogcontracts/dlcspecs/blob/9cd9148938c616690c79d99ec6f330e213c246c5/Non-Interactive-Protocol.md) has been attempted for the DLC spec on what such piece of recommendations can cover.
[/quote]

Perhaps this would be beneficial.  I've also long thought that LN implementations are barking up the wrong tree with all these force close fee optimizations. Getting too fancy in this area inevitably leads to vulnerabilities.  Perhaps we should focus on making force closes less frequent (e.g., by reducing state machine bugs, implementing 0-fee commitments) rather than saving ~30% or less on fees by batching.

-------------------------

