# How wallet fingerprints damage Payjoin privacy

arminsdev | 2026-03-26 15:06:13 UTC | #1

I wrote up an analysis of how wallet fingerprints can be used to partition Payjoin transaction inputs and outputs by owner, potentially recovering the payment amount and undermining the privacy benefits. 

The takeaway: Payjoin privacy depends on fingerprint uniformity between sender and receiver. When wallets diverge on any observable dimension, that divergence becomes a partition signal.

Full post: https://payjoin.org/blog/2026/03/25/wallet-fingerprints-payjoin-privacy/

Interested in feedback on the methodology, additional fingerprint dimensions worth exploring, or cases where this analysis would fail.

-------------------------

0xB10C | 2026-03-27 09:52:01 UTC | #2

Great post! 

While reading, I stumbled over this part in [Example 1: Samourai Payjoin](https://payjoin.org/blog/2026/03/25/wallet-fingerprints-payjoin-privacy/#example-1-samourai-payjoin): 

> Input 0's DER-encoded signature is **71 bytes**. The r-value is 32 bytes with no leading zero pad, meaning r < 2^255. This is a grinded, low-R signature. Input 1's signature is **72 bytes** (the r-value requires a leading `0x00` pad, meaning r ≥ 2^255). This is an ungrinded, high-R signature. A single wallet running a consistent signing policy would either grind all signatures or grind none. Observing one of each is a strong signal that two different signing implementations contributed inputs.

I don't think this is correct. Today, around 60% of the ECDSA signatures on-chain are 71 bytes (https://mainnet.observer/charts/bitcoin-script-ecdsa-length/) and 60% have a low-r value (https://mainnet.observer/charts/bitcoin-script-ecdsa-r-value/). If no wallet would be grinding, it would be 50%. This is because a wallet that does not grind still has a 50% chance to produce a signature that is low-r. I think the only thing you can infer from a transaction with a high-r / 72 byte signature is that it's not been produced by the Bitcoin Core wallet (or Electrum, who also grind). This doesn't hold in the payjoin case though: Just because you see a 71 byte and 72 byte signature, it does not tell you that it were two wallets producing these. It could have been one wallet where it randomly used a low-r and a high-r value.

You might find the PR description of https://github.com/bitcoin/bitcoin/pull/13666 interesting too.

-------------------------

AdamISZ | 2026-03-28 12:52:17 UTC | #3

Great to see someone flagging this and it getting some attention.

It was discussed a fair bit, here and there, when payjoin first became a "thing".

The problem is only obvious once you think about it: unlike the existing onchain privacy-enhancing protocol of coinjoin, this one actually *tries* to mix into the crowd of non-collaborative transactions, and so suddenly these fingerprints are a problem.

A fair counterpoint is, they are *already* a big problem! As they aid blockchain tracing considerably.

There probably should have been an effort to address this in the wallet developer community already, except for the nasty reality: it conflicts, sometimes dramatically so, with the entire purpose of many wallet development projects. A nice example of this is to look into how Green wallet was originally conceived and developed by Lawrence Nahum. Though you could easily find 10 other examples. And an even simpler example is: try to convince wallets that they should "standardize" fees in any ways, and the users, let alone the developers, will tell you to shove it :grinning_face_with_smiling_eyes: . The point is, you cannot realistically expect wallets to remove fingerprints, and while they could try to "80% remove fingerprints" the nature of set intersection analysis is such that, well, that doesn't reduce the problem by 80% ...

In that framing, this is a just a side note, but: there's a fascinating argument to be had about randomize fingerprints or standardize them? I tend to side with randomizing just because it's less brittle. Like nLockTime being not just one value but smeared out, for example. The story of the (non)-adoption of BIP69 is particularly interesting here.

-------------------------

bitgould | 2026-03-29 06:47:21 UTC | #4

[quote="AdamISZ, post:3, topic:2354"]
The point is, you cannot realistically expect wallets to remove fingerprints

[/quote]

I agree with this in practice. It seems tenable to expect transaction producers to make overlapping or sufficiently ambiguous fingerprints in my experience, which may be sufficient to address the core issue, like the 71/72 byte problem @0xB10C brought up in this post’s first comment.

-------------------------

1440000bytes | 2026-03-29 07:14:47 UTC | #5

[quote="arminsdev, post:1, topic:2354"]
Interested in feedback on the methodology, additional fingerprint dimensions worth exploring, or cases where this analysis would fail.
[/quote]

I had built a tool for spoofing wallet fingerprint in 2023 which could be improved further to fix wallet fingerprinting: [https://gitlab.com/1440000bytes/goldfish](https://gitlab.com/1440000bytes/goldfish)

<details>
<summary>Archive</summary>

https://web.archive.org/web/20260329071213/https://delvingbitcoin.org/t/how-wallet-fingerprints-damage-payjoin-privacy/2354/5

</details>

-------------------------

arminsdev | 2026-03-30 12:43:50 UTC | #6

You're correct. A single low-r/high-r pair isn't conclusive and the language here is definitive. However, the signal could compound across the graph. If we traverse the graph and observe a skewed distribution of low-r signatures between the two candidate partitions, this could be evidence for one assignment over the other. Will have to reword this section, thanks.

-------------------------

ZmnSCPxj | 2026-04-02 00:39:05 UTC | #7

It strikes me that all your examples also refer to the round-number heuristic.

As PayJoin is already a multi-round negotiation, it strikes me that both parties can partially break the round-number heuristic by running a lottery for, say, up to +/-1% of the value to be transferred: both exchange hashes of entropy, then exchange their entropy and validate that the received entropy from remote matches the hash, then XOR (or more generally any group addition operation) their entropies, then feed the sum of entropies into a CSPRNG that determines how the 1% lottery goes one way or the other (such as by extracting a large 256-bit number then just doing a simple modulo division by 2% of the value and subtracting 1% of the value; there's a bias there, but a large 256-bit number would have fairly hard-to-detect bias no matter what the divisor is).  Then the amount transferred from the payer to the payee is adjusted by the result of this lottery.

It's not a perfect solution, as anything within 1% of a round number would still be suspect, and while 10% would be better I doubt that would be so easily acceptable to end-users.

-------------------------

