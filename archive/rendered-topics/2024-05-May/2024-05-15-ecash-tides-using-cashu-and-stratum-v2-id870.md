# Ecash TIDES using Cashu and Stratum v2

EthnTuttle | 2024-05-15 16:58:18 UTC | #1

Recommended reading/References: 
- [Cashu ecash protocol](https://github.com/cashubtc/nuts)  
- [ecash Proof of Liabilities](https://gist.github.com/callebtc/ed5228d1d8cbaade0104db5d1cf63939)
- [Ocean TIDES](https://ocean.xyz/docs/tides)
- [Stratum v2 Spec](https://stratumprotocol.org/specification/) (Sv2)

Goals:
- Auditability
- Small payouts
- Privacy

Using the [Blind Diffie-Hellmann key exchange (BDHKE)](https://github.com/cashubtc/nuts/blob/6024402ff8bcbe511e3e689f6d85f5464ecc3982/00.md) from Cashu as a Sv2 [Protocol Extension](https://stratumprotocol.org/specification/03-Protocol-Overview/#34-protocol-extensions), a pool can provide auditable, small scale, private pool rewards in the form of `eHash`.

A hash provider (HP) connects to a pool and begins job negotiation. This aspect remains largely unchanged from current paradigms. However, the HP will also request the active `block_keyset`. This is a list of pubkeys as defined in [NUT02](https://github.com/cashubtc/nuts/blob/6024402ff8bcbe511e3e689f6d85f5464ecc3982/02.md), but the value amounts for the individual keys correspond to difficulty targets for various miners which is used as a share weight in the TIDES rewards. The `unit` for these keysets is left up to specific implementations. A `block_keyset` is rotated once the pool finds a block or when the network difficulty adjustment occurs. 

When the HP finds a valid share, in addition to standard share data, the HP will generate a secret and send a blinded message (`B_`) to the pool. The inputs for this blinded message could be done at an individual ASIC level or as part of a farm proxy implementation, with the latter seemingly ideal for larger farms. This blinded message (`eShare`) would be included in Sv2 channel messaging as a protocol extension. This signals to the pool that there is a valid share and requests a signature using the appropriate key from the active `block_keyset`. The same blinded message could be reused multiple times as long as special consideration is given on the pool/mint side for tallying these submissions. Reuse of blinded messages may be necessary during implementation if computing these blinded messages is too costly for a given HP, otherwise a new blinded message should suffice and lower complexity.

When the pool receives an `eShare`, it first validates the share itself, as usual. Once validated, the pool signs the blinded message, completing the DHKE and providing the blinded signature (`C_`) back to the HP. This share + blinded signature are then added to the time ordered TIDES defined `share_log`. The HP retains the blinded message data inputs (`x`) and the unblinded signature (`C`) as defined in the Cashu protocol [NUT00](https://github.com/cashubtc/nuts/blob/6024402ff8bcbe511e3e689f6d85f5464ecc3982/00.md). 

This share process continues until a new difficulty adjustment occurs, after which a new keyset is issued with appropriate difficulty targets, OR a block is found by the pool. A keyset rotation after a difficulty adjustment accounts for new share target difficulties. A keyset may be reusable across difficulty adjustments as a % delta from network difficulty.

When a block is found by the pool, a full list of shares and associated blinded signatures is published by the pool. This signals to the HP that the `eHash` shares are available for redemption. Now the pool references the `share_log` using the `share_log_window` (defined in TIDES) for what `eHash` is eligible for redemption. As HP's redeem their `eHash`, the `eHash` is recorded alongside the public record of shares and blinded signatures as defined in the [ecash Proof of Liabilities](https://gist.github.com/callebtc/ed5228d1d8cbaade0104db5d1cf63939). 

Redemption of `eHash` can be any of the available Cashu or ecash mechanisms appropriate for the HP.

Other thoughts:
- Cashu supports spending conditions, such as P2PK ([NUT10](https://github.com/cashubtc/nuts/blob/6024402ff8bcbe511e3e689f6d85f5464ecc3982/10.md) and [NUT11](https://github.com/cashubtc/nuts/blob/6024402ff8bcbe511e3e689f6d85f5464ecc3982/11.md))
- the blinded message secret uses `hash_to_curve(x)` computation which could be offloaded to an ASIC, maybe.
- Since this is an ecash usage, [Fedimint](https://fedimint.org/) is also a viable option but has not been considered in depth.

P.S.
I look forward to feedback and considerations I may have missed. This post was an effort to solidify my idea into something more concrete and shareable. A vocal (and probably less comprehensive/coherent) monologue of this idea can be found [here](https://fountain.fm/episode/zfEPkjPmQd8rD2cxq5tR). I hope to create a diagram to help illustrate this proposal.

-------------------------

davidcaseria | 2024-05-15 22:25:05 UTC | #2

In TIDES, shares are paid multiple times. I'm unsure if I follow how an eHash can be redeemed multiple times. Can you clarify that?

This would also enable shares to be exchanged by sending eHash tokens with an atomic swap spending condition (i.e., [NUT-11](https://github.com/cashubtc/nuts/blob/6024402ff8bcbe511e3e689f6d85f5464ecc3982/11.md)).I think this is a huge benefit and something Ocean has said they would like to enable.

This is an exciting proposal. Thanks for putting it together!

-------------------------

EthnTuttle | 2024-05-15 22:54:49 UTC | #3

> In TIDES, shares are paid multiple times. I’m unsure if I follow how an eHash can be redeemed multiple times. Can you clarify that?

This is the trickiest part of the proposal and it *might* be solved as follows. I do hope others will ponder this aspect for other ways to 'reuse' ecash.

When a `share_log_window` is created, it becomes a snapshot shot of valid signatures that can be redeemed. At this time, the new `share_log_window` is known. When the HP redeems `eHash` that still falls within the window, that can be swapped for another `eHash` token AND other redemption methods. The new blinded signature would replace the old in the share log.

I did consider a scenario where each blinded signature could be redeemed more than once but I think that's not a practical solution since after the first redemption, the pool/mint knows secrets (`x` and `C`).

-------------------------

1440000bytes | 2024-05-16 01:04:54 UTC | #4

I don't see any benefits of using Cashu for mining payouts.

[quote="EthnTuttle, post:1, topic:870"]
Goals:

* Auditability
* Small payouts
* Privacy
[/quote]

Auditability and small payouts are achieved with [bolt12](https://ocean.xyz/docs/lightning). LN privacy is debatable however use of Cashu to achieve privacy comes with the below trade-offs:

- Custodial
- Makes the mining pool vulnerable to regulatory actions or use of KYC/AML

-------------------------

davidcaseria | 2024-05-16 01:25:28 UTC | #5

[quote="1440000bytes, post:4, topic:870"]
LN privacy is debatable however use of Cashu to achieve privacy comes with the below trade-offs:

* Custodial
* Makes the mining pool vulnerable to regulatory actions or use of KYC/AML
[/quote]

Mining pools are already custodial and vulnerable to state action, such as enforcing KYC requirements (which some do).

The possibility of immediate liquidity of mining rewards is self-evidently beneficial.

-------------------------

1440000bytes | 2024-05-16 22:18:12 UTC | #6

[quote="davidcaseria, post:5, topic:870"]
Mining pools are already custodial and vulnerable to state action, such as enforcing KYC requirements (which some do).

The possibility of immediate liquidity of mining rewards is self-evidently beneficial.
[/quote]

[TIDES](https://bitcoin.stackexchange.com/questions/120719/how-does-ocean-s-tides-payout-scheme-work) is only used by Ocean mining pool. Based on a private conversation with CTO of Ocean, they plan to stay non-KYC hence doing payouts in the coinbase transaction.

Mining rewards are essentially an IOU in this case until they are redeemed for bitcoin.

-------------------------

davidcaseria | 2024-05-16 10:07:06 UTC | #7

I understand your point about Ocean’s current KYC policy, but I still believe that your argument is weak. Not all of Ocean’s payouts are in the coinbase transaction, and with a minimum payment threshold, shares in the share log already act as IOUs.

I don't think it's an accurate characterization to say there are no benefits and it introduces new trade-offs, because in reality, these legal considerations already exist for a pool using TIDES and are not as clear as you suggest.

I believe this idea is worth discussing on its own merit, even though implementing it seems like a long shot due to the need for custom stratum protocol implementation regardless of potential legal issues.

-------------------------

calle | 2024-05-16 10:20:39 UTC | #8

Thank you, interesting post and very thoughtful use of blind signatures. I have a question on the value field for keysets that correspond to difficulty targets. These normally represent a satoshi amount in Cashu, but it's totally fine to replace them with anything else. Is it correct to assume that you're measuring the value of a blind signature by the target of the submitted PoW share and get that signed by the mint? 

Am I right to assume that when the user wants to convert the `eShares` to satoshis, the mint would look take the target values of the eShares and convert them to satoshis? I understand that the accounting of which target gets what amount is not part of the protocol here but I'm trying to understand why the values aren't just directly Satoshis. I'm sure there is a good reason for that.

Thanks!

-------------------------

plebhash | 2024-05-17 14:53:33 UTC | #9

Great to see this discussion moving forward here! I see there's some differences from what we discussed on the [plebpool repo discussion](https://github.com/plebemineira/plebpool/discussions/7), so I still need to find time to catch up on many details, especially TIDES.

For now, my contribution is limited to this:

[quote="EthnTuttle, post:1, topic:870"]
the blinded message secret uses `hash_to_curve(x)` computation which could be offloaded to an ASIC, maybe.
[/quote]

Unfortunately, that is not possible. Bitcoin Mining ASICs never really output the actual hash they computed, that is a common misconception.
Check this [thread by @skot9000](https://x.com/skot9000/status/1770083820361875818) for more details.

[quote="1440000bytes, post:4, topic:870"]
Auditability and small payouts are achieved with [bolt12 ](https://ocean.xyz/docs/lightning).
[/quote]

While BOLT12 payouts definitely lower the threshold on how small the HP can get paid, there's still a barrier on LN fees. Depending on how small the HP's hashrate is, it could take a while before they can withdraw. Until then, all they can do is watch their dashboard and cross their fingers.

The proposal here improves things a little bit, because now, the HP has cryptographic proof of the submission and acceptance of their PoW.

Sure, one could argue there's still risk of a rugpull by the pool, but this problem is inherent to the nature of any centralized pool. Even OCEAN is custodial for small hashrates. There's no simple solution for this that doesn't imply a fully decentralized pool architecture, which dramatically increases the complexity of the solution, because now we are dealing with a consensus system on its own. 

While some projects (e.g.: [braidpool](github.com/braidpool/braidpool)) are making significant progress in this direction, Bitcoin mining is still going to have centralized pools for a while.

So IMHO this is a substantial improvement to the current mining landscape. Cashu takes tradeoffs on custody and centralization, and incentives are very aligned with centralized pools.

Finally, as my nym loudly states, I'm very excited about pleb mining. While some see it as a silly effort (since the long-tail of hashpower on pleb miners is somewhat insignificant compared to industrial mining), I believe that any efforts that make pleb mining more accessible are worthwhile to decentralize Bitcoin mining.

-------------------------

EthnTuttle | 2024-05-17 01:45:46 UTC | #10

1. How does Bolt12 enable auditability?
2. bolt12 presumes a LN channel which requires liquidity. Perhaps I should rephrase away from "small payouts" to something more fitting for the trade offs. micro payouts with zero prior liquidity requirements and no operational complexity for the pool user.

RE: Custodial, it seems others have alluded to the same but custodial pools are not a new concept. Every custodial wallet should be an ecash wallet so the user at least gains privacy as a trade off. A pool "wallet" should be no different. This proposal does not aim to solve self-custody. Ocean seemingly has that covered.

-------------------------

EthnTuttle | 2024-05-17 02:14:39 UTC | #11

> Is it correct to assume that you’re measuring the value of a blind signature by the target of the submitted PoW share and get that signed by the mint?

Yes. A valid target difficulty for a share to be accepted is generally a threshold below the network difficulty. With various machines of different hash rates, a pool uses a variety of target PoW difficulties to prevent DoSing itself. Giving the same target to a S9 and a S21 would either result in the S9 never finding that target in the given timeframe or the S21 inundating the pool with shares. Given the possibility of small payouts, it seemed appropriate to include hash providers with less robust machines.

> Am I right to assume that when the user wants to convert the `eShares` to satoshis, the mint would look take the target values of the eShares and convert them to satoshis?

Yes. I believe the TIDES documentation cover the weighting of shares as it relates to the `share_log_window` but accounting specifics would be left to the implementation.
Since a blinded signature corresponds to a target difficulty, and TIDES uses the target difficulty as a weighting for valuation within a share_log_window, a `eShare` would not be a 1:1 for satoshis.

-------------------------

1440000bytes | 2024-05-17 02:30:14 UTC | #12

> Using BOLT12 also allows us to prove to the world that a payment was made, the size of the payment, the node to which it was paid, and that it was paid by us.

> This means we can continue to offer fully transparent and verifiable pooled mining while no longer being restricted by the base layer!

 https://njump.me/nevent1qqs8sz359u7ysd8hw39v99hlxl5zs7mzsrrw5rwpsctm0ufart2g0ngpp4mhxue69uhkummn9ekx7mqppamhxue69uhkummnw3ezumt0d5q3gamnwvaz7tmwdaehgu3wdau8gu3wv3jhvq3qqtvl2em0llpnnllffhat8zltugwwz97x79gfmxfz4qk52n6zpk3qn2uecg

I understand that a key objective of this proposal is to get more users for Cashu. Whereas I believe there are very less non-kyc mining pools and we should not **maximize** the probability of them being forced to comply or get shutdown. Mining pools that are already custodial and follow KYC/AML will have to use [NUT 16](https://github.com/cashubtc/nuts/pull/106) if they want to enable micro payouts with zero liquidity. I doubt there is any demand for such micro-payouts and looks pointless to use KYC with eCash. Use of NUT 16 by some mints is a slippery slope that would make it easier for government agencies to take action against other mints.

In simple terms, you could compare this to mining pool using a custodial mixer that provides a token on deposit and do payouts with the withdrawal. 

Hence, I see no real benefits for any mining pools to use this.

-------------------------

calle | 2024-05-17 10:29:21 UTC | #13

[quote="EthnTuttle, post:11, topic:870"]
Yes. I believe the TIDES documentation cover the weighting of shares as it relates to the `share_log_window` but accounting specifics would be left to the implementation. Since a blinded signature corresponds to a target difficulty, and TIDES uses the target difficulty as a weighting for valuation within a share_log_window, a `eShare` would not be a 1:1 for satoshis.
[/quote]

Alright, I get it now. An alternative would have been to let the mint decide the values of blind signatures after the fact (the user doesn't have to decide on the value when sending the blind message). But I think your approach is more simple and doesn't require the mint operator to store all outstanding blind messages and imprint them with value after doing all the accounting. It's much better than my original idea I had discussed with you on GitHub!

-------------------------

MattCorallo | 2024-05-17 17:32:19 UTC | #14

[quote="davidcaseria, post:5, topic:870"]
The possibility of immediate liquidity of mining rewards is self-evidently beneficial.
[/quote]

I honestly entirely fail to see the value of these kinds of systems. To be clear, giving miners their reward quickly such that they can migrate off a pool quickly if the pool is not paying them for their work is certainly valuable, however the difficult part of this is not the "pay out quickly" part, its the "if the pool is not paying out sufficiently, migrate" part.

A key part of Stratum V2's custom work selection proposal is the behavior of client software automatically switching to a fallback pool or solo mining in the case that their pool declines to pay for their work. Indeed, in current implementations this is only accomplished through a message the pool can optionally send to the client informing them that they will be ceasing payouts. Having this be automated on the basis of the actual payout would be awesome! But we certainly don't have to revert to anything complicated to do so - any off-chain payout scheme suffices here, whether lightning (as a handful of pools already support) or ecash via the normal ecash payment negotiation flows.

Client behavior to switch to a fallback pool or solo mine based on comparing expected and actual payout value, is, however, the hard part here - no pool payout scheme today is capable of paying out faster than every few blocks (PPLNS), and the ones used in practice (PPS) can only pay out once a day.

If you're using a PPLNS pool (like TIDES, IIUC), the best you can do is check your payout once the pool finds a block, but this is actually impossible to do in an automated fashion because you cannot reliably detect if/when your pool mined a block. The best you can do is detect that you found a block for the pool, which is generally not all that useful in general.

If you're using a PPS pool, you can at least do a daily check, however this is somewhat complicated by the fact that PPS calculations differ by pool and are changed over time. AFAIU most pools average (with some outlier removal) the fees found over the previous day's blocks and do payouts on that basis, implying they can only do payouts once a day after they can calculate this average.

Thus, no matter how quickly or automatically we can do payouts the software can't really do what we want for us, instead users have to be involved. Given this, we can't really do all that much interesting here, aside from just making sure payouts go out in a timely manner, but heaping on complexity to accomplish that is...just heaping on complexity.

Certainly transparency is important, and we can do things like share log transparency (eg https://github.com/stratum-mining/sv2-spec/discussions/76#discussioncomment-9472619) to aide users in deciding if they've been paid out appropriately and let them decide to switch pools on that basis.

-------------------------

EthnTuttle | 2024-05-19 00:41:14 UTC | #15

[quote="1440000bytes, post:12, topic:870"]
Hence, I see no real benefits for any mining pools to use this.
[/quote]

Understood. Thank you for sharing.

-------------------------

EthnTuttle | 2024-05-19 00:44:40 UTC | #16

[quote="MattCorallo, post:14, topic:870"]
Certainly transparency is important, and we can do things like share log transparency (eg [Share Accounting + Accouintability Protocol Extension · stratum-mining/sv2-spec · Discussion #76 · GitHub](https://github.com/stratum-mining/sv2-spec/discussions/76#discussioncomment-9472619)) to aide users in deciding if they’ve been paid out appropriately and let them decide to switch pools on that basis.
[/quote]

You don't see cryptographic attestation as a benefit to share log transparency? This proposal has a transparent share log, it just adds an additional attestation from the pool side that miners can verify.

-------------------------

EthnTuttle | 2024-05-19 00:53:41 UTC | #17

I understand this makes complexity and trade off decisions that vary from other ways to do similar things, but please be explicit if you see any technical reason the protocol/implementation/cryptography/etc is unsound or does not work?

While I do understand and consider the trade offs, I am interested in understanding applied cryptography and this is one of my first documented attempts. How can this protocol be gamed or broken? Where does it break? Why does it break? technically speaking.

-------------------------

davidcaseria | 2024-05-19 15:43:33 UTC | #18

[quote="EthnTuttle, post:3, topic:870"]
This is the trickiest part of the proposal and it *might* be solved as follows. I do hope others will ponder this aspect for other ways to ‘reuse’ ecash.
[/quote]

I don't fully understand your statement here because I believe that ecash can't be reused by its nature.

I think I understand your idea at a high level: redeem the ehash for the reward and also receive another ehash to be redeemed later. How do you track how often an ehash has been issued for the original share? Is there a way to link an ehash to a share? How would this work if an ehash is split into representations of less difficult shares?

-------------------------

EthnTuttle | 2024-05-19 21:26:33 UTC | #19

> How do you track how often an ehash has been issued for the original share?
I don't think this is the correct way to think about it. If ehash is to be "used twice", you just have a [/swap](https://github.com/cashubtc/nuts/blob/6024402ff8bcbe511e3e689f6d85f5464ecc3982/03.md) that outputs two different units. One is the actual reward (on chain/ln/ecash) and the other is another ehash.

 > Is there a way to link an ehash to a share? 

By whom? A mint knows which share to which ehash after redemption since the mint has. But the redeemer need not be the originator, as with other ecash. 

> How would this work if an ehash is split into representations of less difficult shares?

This is already covered and uses the [keyset](https://github.com/cashubtc/nuts/blob/6024402ff8bcbe511e3e689f6d85f5464ecc3982/02.md)

-------------------------

MattCorallo | 2024-05-20 23:51:47 UTC | #20

[quote="EthnTuttle, post:16, topic:870"]
You don’t see cryptographic attestation as a benefit to share log transparency? This proposal has a transparent share log, it just adds an additional attestation from the pool side that miners can verify.
[/quote]

Maybe explain what you mean by "cryptographic attestation". The protocol I describe there allows any user of a pool to fully audit the payouts the pool is making and, if the pool cheats, publish a concise proof of the issue. It is also incredibly simple and doesn't have much overhead. The only disadvantage I see compared to this or some of the other protocols described in that thread is that it has some time lag, but as I described above, pools in practice have quite a bit of time lag before they can do payouts anyway, so I believe this point is moot.

-------------------------

