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

EthnTuttle | 2024-05-21 11:16:50 UTC | #21

By "cryptographic attestation", I mean that the pool/mint is signing the individual shares and those signatures can be checked and cannot be modified after that fact.
Since this signature happens at time of share validation, the hash provider knows that anything modified after the fact is malicious behavior.

-------------------------

calle | 2024-05-23 07:29:56 UTC | #22

[quote="MattCorallo, post:14, topic:870"]
the difficult part of this is not the “pay out quickly” part, its the “if the pool is not paying out sufficiently, migrate” part
[/quote]

IIUC, the point of this proposal is to introduce privacy for payouts. That's it. So far, I see mostly discussions about unrelated or at best mildly related issues here.

-------------------------

davidcaseria | 2024-05-23 21:57:57 UTC | #23

[quote="EthnTuttle, post:19, topic:870"]
By whom? A mint knows which share to which ehash after redemption since the mint has. But the redeemer need not be the originator, as with other ehash.
[/quote]

Ok so if an ehash is sent to another person before redemption would the pool/mint need to update the share log to keep track of the swap?

I think I understand how this would work now for an ehash to be paid multiple times: when an ehash is redeemed the share is looked up in the share log, if the share is still in the share window then a new ehash is provided. If the ehash falls out of the share window, then it can no longer be redeemed. Is my understanding correct?

-------------------------

EthnTuttle | 2024-05-24 13:07:52 UTC | #24

> ehash is sent to another person before redemption would the pool/mint need to update the share log to keep track of the swap?

No. But the receiver of the ehash would want to redeem what is received ASAP to avoid a sender double spend. This is true of ecash as well. When you send ecash to an other user, you're sharing the root secret used in the blinded signature and so the receiver is incentivized to redeem that sooner than later, although the transfer may be done offline.

> when an ehash is redeemed the share is looked up in the share log, if the share is still in the share window then a new ehash is provided. If the ehash falls out of the share window, then it can no longer be redeemed. 

Correct. There is something to be considered with the share window that the TIDES doc calls out:
> If the network difficulty goes up, the size of the window increases. Older shares end up being in the window again, despite having “slid out” before the difficulty change.

So there is a perpetually rolling list of ehash that the pool should be tracking.

-------------------------

MattCorallo | 2024-05-31 20:13:43 UTC | #25

[quote="EthnTuttle, post:21, topic:870, full:true"]
By “cryptographic attestation”, I mean that the pool/mint is signing the individual shares and those signatures can be checked and cannot be modified after that fact. Since this signature happens at time of share validation, the hash provider knows that anything modified after the fact is malicious behavior.
[/quote]

Sure, but they could just sign the shares as they're submitted (a trivial extention to Sv2 to have the pool sign all SubmitShares.Success messages would suffice)? There's no need to tie that directly to ecash in any way.

[quote="calle, post:22, topic:870"]
IIUC, the point of this proposal is to introduce privacy for payouts. That’s it. So far, I see mostly discussions about unrelated or at best mildly related issues here.
[/quote]

This proposal is very substantially overcomplicated if that is the only goal. If this is the goal, the pool simply needs to make payouts using lightning/ecash, it doesn't require tying shares to ecash or anything fancy like this.

-------------------------

davidcaseria | 2024-06-02 21:08:52 UTC | #26

[quote="MattCorallo, post:25, topic:870"]
This proposal is very substantially overcomplicated if that is the only goal. If this is the goal, the pool simply needs to make payouts using lightning/ecash, it doesn’t require tying shares to ecash or anything fancy like this.
[/quote]

If the goal was expanded to include making pool shares into a tradable asset, would you see benefits in using e-cash?

-------------------------

MattCorallo | 2024-06-03 17:16:36 UTC | #27

[quote="davidcaseria, post:26, topic:870"]
If the goal was expanded to include making pool shares into a tradable asset, would you see benefits in using e-cash?
[/quote]

For a centralized pool, I'm not quite sure why you'd want that (at best they'd trade at roughly par?), unless the pool didn't support withdraws via echas/lightning natively (but they should just do that instead!). That said, tradable shares is one of the two key insights to braidpool, but the reason they make the shares tradable is because they can only do payouts for the top N (probably, like, 10) miners owed the most in any given block, so you need to let users with lower hashpower sell their shares to larger miners for lightning/ecash.

-------------------------

mcelrath | 2024-06-04 13:47:41 UTC | #28

[quote="MattCorallo, post:27, topic:870"]
That said, tradable shares is one of the two key insights to braidpool, but the reason they make the shares tradable is because they can only do payouts for the top N (probably, like, 10) miners owed the most in any given block, so you need to let users with lower hashpower sell their shares to larger miners for lightning/ecash.
[/quote]

Braidpool chooses to pay every difficulty adjustment epoch (2016 blocks) because in this window the hash price is constant, and tradable shares are a natural forward contract. We could just as easily do PPLNS and pay in every block. It's a trade-off (as you indicate) between wasting block space with miner payouts (OCEAN/PPLNS pays up to 8x in multiple blocks for the same share) and paying smaller miners as you indicate. Tradable shares will be a V2 feature for Braidpool, we won't launch with it. Anyway we have a new idea here that I think is worth trying. It's always possible to fork Braidpool using a different payout mechanism if our current proposal turns out to be undesirable for some reason. And criticism is welcome.

Forward/futures contracts are a fundamental risk management tool for commodities producers. This is one way we want to make Braidpool **more** profitable/desirable than other pools, and other design considerations like the 2016 block window are chosen to enable this. While I encourage the development of other ideas, we're going to end up fragmenting the market with different kinds of forward/futures contracts (braidpool shares, eCash, Luxor futures, indexed futures, etc) that are not the same and not tradable amongst each other. This is the difference between a futures contract and a forward contract. Any private bespoke contract which pays in the future is a forward, whereas futures contracts are standardized to make them tradable and increase market liquidity. Anyway I look forward to further development on the topic and I'd like to see what eCash can come up with. Centralized pools running on top of Braidpool can provide unique payout mechanisms including eCash and Lightning.

Cheers,
-- Bob

-------------------------

davidcaseria | 2024-06-04 20:21:55 UTC | #29

[quote="MattCorallo, post:27, topic:870"]
For a centralized pool, I’m not quite sure why you’d want that (at best they’d trade at roughly par?), unless the pool didn’t support withdraws via echas/lightning natively (but they should just do that instead!).
[/quote]

I think maybe this is where confusion is being generated. As I understand the proposal, a share represented by an e-cash token is not equivalent to a payout, so I'm not sure what you mean by your comment.

IIUC, hypothetically, a miner would receive a share for proof of work in the form of an e-cash token. In a TIDES payout scheme, shares represented by an e-cash token must be redeemed multiple times (i.e., 8 in the average case) to collect the appropriate payouts when the pool finds a block. Thus, the withdrawal mechanism of the pool doesn't matter because the e-cash shares aren't payouts.

When I read the proposal, I saw this distinction and immediately realized the possibility of a market (which I think will be crucial because miners want FPPS, but the pools will have trouble supporting that as TX fees outpay the block subsidy). Apologies if I distracted us from the technical aspects of the proposal with the economics of it.

-------------------------

EthnTuttle | 2024-07-21 19:48:40 UTC | #30

https://github.com/stratum-mining/stratum/discussions/1052

-------------------------

marathon-gary | 2024-10-15 21:03:35 UTC | #31

https://www.youtube.com/watch?v=uCyRffPdsaU
The first presentation in this video is Hashpools by @vnprc

-------------------------

vnprc | 2025-01-08 01:56:55 UTC | #32

Hey! I've been working on this idea for some time in isolation. I'm in the process of [building it](https://github.com/vnprc/hashpool). Apologies for not engaging in this discussion earlier. I didn't realize there was such a robust discussion until I read the Optech year-in-review.

I believe I have solved many (all?) of the issues raised here and discovered several new insights. But in the process I have raised more questions that need answers. Let's get into it:

**You can't redeem ecash multiple times**

I tried this approach as well and decided that it was impossible to reissue an ecash token after the first payout because these tokens are inherently unlinkable to the underlying collateral (in this case a PoW mining share). You can do multiple redemptions by linking the tokens to the mining share, but in the process you destroy the privacy properties of ecash.

I think the correct solution is to only allow one redemption per eHash token. If miners hold the token until the share window expires they get the maximum payout. If they choose to redeem early they donate potential future rewards in that share window to all the other owners of shares in that window. Kind of like how destroying bitcoin only makes the remaining bitcoin more valuable.

If miners want immediate liquidity they can sell their eHash tokens. This closely emulates FPPS payouts without adding trust assumptions (more on this in the auditing section). This will create a new and highly efficient futures market.

To Bob McElrath's point differentiating futures from forwards, eHash will be a futures instrument because cashu will be the standard. Users can buy and sell tokens from different mints through any exchange medium. Personally, I think decentralized and private nostr marketplaces are the right way to go here but that's external to this proposal.

**Payout Calculations**

I go into detail on this in [my talk](https://www.youtube.com/watch?v=uCyRffPdsaU). The key insight is that you can use Calle's [Proof of Liabilities](https://gist.github.com/callebtc/ed5228d1d8cbaade0104db5d1cf63939) protocol to enforce an arrow of time on mint operations. When a share is accepted, an eHash token will be issued within the current ecash epoch. Epochs are defined by the keyset used by the mint to sign blinded secrets. Epochs will be rotated at a regular time interval. The mint transitions to a new epoch by announcing a new keyset. This essentially 'buckets' all shares into time windows that can be used to calculate payouts and enforce a TTL for each token the mint issues.

I find it useful to think of this construction as a series of rolling contracts. Each ecash epoch is a different contract and eHash tokens issued in that epoch are a claim on the block rewards the pool wins within the share window of the contract. Coinbase transactions are peg-ins to the contract and eHash redemptions are peg-outs.

The *really* cool part is that with this arrow-of-time scheme you also get ecash mint auditing for free.

**Auditing**

An ecash mint is essentially a new kind of bank. In banking, there are two facets to auditing: assets and liabilities. Ehash tokens are liabilities and can be audited in a privacy-preserving fashion as described in the PoL protocol (linked above).

In the mining pool scenario each PoW mining share represents an asset, it is proof that some miner performed valuable work for the pool and should be compensated accordingly. I haven't designed this protocol yet but I am keen to get there. This will also be very useful for existing pools since proving share validity is an unsolved problem for the industry at large.

In order to prove the validity of a mining share you need 3 pieces of information: block template, block header, and nonce(s). The goal is to prove that each share accepted by the pool corresponds to a valid block.[^*]

I think all you need is a lookup table of block template, header, and nonce data and a merkle sum tree of block header hashes. In order to prove an individual share's validity you look up the block template, header, and nonce data, combine them into a full bitcoin block (with insufficient proof of work, probably), hash the header and provide a merkle proof for that header. The merkle sum tree is used to attest to the sum of difficulty of accepted shares. Ehash tokens are denominated in difficulty so this value can be directly compared to the sum of liabilities (mint proofs minus burn proofs (not including ecash redemptions)) from the Proof of Liabilities report. I haven't fully scoped out the idea but it seems like a fairly straight-forward problem. Am I missing anything?

A really cool aspect of this proposal is that it actually closes all of the gaps that Calle describes in the PoL protocol. By comparing a proof of the collateral used to issue liabilities against the proof of liabilities we can fully audit all mint operations, leaving no possibility of hidden fraud on the part of the mint. (Please prove me wrong!)

I was surprised and encouraged by this realization. By exploring the architecture that would enable ecash mining shares I found solutions to problems that I wasn't even trying to solve. It's pretty fucking rad IMO. Keep reading for more solutions to seemingly unrelated problems. :)

**Goals of the Proposal**

This was repeatedly brought up in this thread and I agree that unclear or unstated goals lead to confusion and dead end arguments, so let's clear the air. I have three major goals in mind, in order of importance:

1. distribute block template production
2. create a new FOSS self-hostable KYC-free bitcoin onramp
3. reduce the minimum threshold of mining withdrawals (enable pleb mining)

I originally started working on this problem with only last two goals in mind but then I had an insight that blew open the doors. Pseudonymous and transparent PPLNS payouts enable a layered approach to bitcoin mining. With PPLNS, you can run a small pool that mines upstream to a larger pool. This lets you skip over the 0-to-1 problem of launching a new pool. You don't need to launch with enough hashrate to regularly find blocks. A small pool can mine to an account with a larger pool, receive payouts when the upstream pool finds a block, and issue payouts to downstream miners from those funds. This is a fundamentally more scalable arrangement than the monolithic mining pool model that everyone seems to carry around in their head.

It only makes economic sense with PPLNS pools because FPPS pools assume the role of pricing hashrate; PPLNS pools push the 'luck risk' or payout variability (and thus the pricing of hashrate) onto the miners. We're building a decentralized ecash market for pricing hashrate, so mining to an FPPS pool account pays the upstream pool to provide a redundant service that free markets can do a better job at providing. This fee is wasted on an FPPS pool and only increases the cost to downstream miners.

With a diversity of hashpools mining upstream to a few large top-level mining pools we can finally realize the ideal of a big dumb mining pool that does nothing more than aggregate hashrate and distribute block rewards. The second layer pools can specialize in payout mechanisms, authentication (or lack thereof), block template production, and more.

I believe the biggest wins come later when hashpools can start experimenting with template selection and coinbase payouts. Paid template selection would enable hashpools to offer transaction accelerator and non-standard transaction services. Coinbase output markets enable the mining pool to be used as a coin mixer in a fundamentally more private manner than coinjoin services because coinbase outputs have no on-chain transaction history. 🤯

I've written a lot on these topics elsewhere and this post is long enough already so I'll stop here. But ponder this: what would bitcoin look like with a thousand small mining pools innovating in coinbase and block template future (or forward) instruments? The state can stop some of us some of the time but they can't stop all of us all the time. Let's fucking go.

[*] Valid within the bitcoin consensus rules at the current block height. The pool also needs to validate that the coinbase transaction includes the cashu `keyset_id` somewhere and pays the right amounts to the to the right outputs. The `keyset_id` commitment prevents miners from submitting shares to multiple pools.

-------------------------

vnprc | 2025-01-07 22:41:04 UTC | #33

**Open Questions**

1. What should the Proof of Assets a.k.a. hashrate validation protocol look like?

Can we build something that works for hashpool and traditional mining pools? Can we get some help from existing pools? 🙏

2. What payout formula should we use? Basic PPLNS, TIDES, or some other tweaked PPLNS algorithm?

If the pool uses authentication I think we could even go back to proportional payouts. PPLNS was invented to prevent pool hopping but it would seem that an eHash free market could also solve this problem. So can we go back to proportional payouts in this scenario? What are the benefits? Is the juice worth the squeeze?

**Post 1.0 Questions**
1. How to build a block template selection market?

The solution probably involves mining a share and selling it to the 'block template purchaser' (or transaction accelerator customer if you like to think in those terms) along with a merkle proof of the mining share.

2. How do we minimize MEV or MEVil risk?

You can't solve a problem that is poorly defined. I think we first need to do some fundamental research into the causes of MEV (or MEVil). I am confident that with the right foundational research we can put some basic restrictions in place to limit this possibility.

`<soapbox>` I *do not* believe the solution is to avoid building any software that could possibly enable spooky, undefined behavior. Bitcoin is permissionless, someone will build it (they might even get a free wizard NFT for their efforts). We are better off building it the right way first and setting the example to emulate. `</soapbox>`

3. How to build a coinbase output market?

I have concluded that you can't trustlessly pay for a coinbase output with eHash tokens because the tokens are indelibly tied to past block templates. How does one ensure that future block templates include the right coinbase output? Furthermore, how do you ensure the coinbase output has the right amount if eHash tokens have indeterminate value?

The only remaining option is for the pool to sell coinbase outputs directly for bitcoin. Which leads to my next question:

4. How do we trustlessly and transparently funnel (most of) the profits from selling coinbase output to the miners?

If the purchaser is paying with an on-chain UTXO they can craft a transaction that pays the right amount directly to mining fees. This way all miners who contribute to the pool get to share in the profits. But how to prevent other pools from mining that transaction and stealing those fees? Easy peasy, put a connector output in the coinbase.

Well...it seems easy peasy but I can imagine a consensus rule that might prevent a graph cycle where a coinbase output pays to a transaction that pays fees into the coinbase. Has anyone ever tried to build something like this? Does such a rule exist?

I haven't worked out a solution for coinbase output purchasers paying in ecash. Maybe the mint simply creates the transaction that flows entirely to fees? This is a half-baked idea. Let me know if you can think of a way to break it.

-------------------------

mcelrath | 2025-01-08 14:42:29 UTC | #34

Really cool that you're actually building this. Kudos!

How do people validate shares? A share is a full bitcoin block, including all transactions, but having a PoW less than Bitcoin's difficulty target.. This means a "share" is up to 4MB of data. See [General Considerations for Decentralized Mining Pools for Bitcoin](https://github.com/braidpool/braidpool/blob/6bc7785c7ee61ea1379ae971ecf8ebca1f976332/docs/general_considerations.md) for details on this. If block templates are truly arbitrary, this means that any share mining a tx that is not in the share-validator's mempool cannot be validated. Even if all txs are known at best I have to store and download the txid list for every share. At worst I have to download an entire 4MB graffiti block full of txs I've never seen before.

Let me suggest something. [Braidpool](https://github.com/braidpool/braidool/) will be a public blockchain (DAG) with each entry (bead) in the DAG being a share. I've proposed using using [Deterministic Block Templates](https://github.com/braidpool/braidpool/discussions/69) to mitigate this problem. The idea being that Braidpool beads carry bitcoin transactions, which for any given set of braid tips defines a committed mempool. Tx selection can then proceed using any deterministic algorithm on this committed mempool. Proof of a share is then only the share header (bead ~ 2kb or so), which implicitly commits to a tx set in a committed mempool, and a block header independently computable via the defined deterministic algorithm. Thus the block template nor its txs need to be separately stored and or transmitted for verification (as long as the share verifier is running a Braidpool node -- the data within is entirely public).

Now, this is really only possible with a consensus algorithm on that mempool. Because TIDES and Cashu don't have any such publicly known list of txs or consensus, that means that shares in your scheme require transporting or storing a tremendous amount of data.

Braidpool is limited on the size of shares it can support, by the restriction on how many beads (shares) can be mined in a given period of time and consensus decide upon them. This is limited by the latency of transmitting shares, and given the 600s block time and observed latencies, this is a bead time around 250ms - 1000ms, so for round numbers let's call it 600ms or 1000x shares per Bitcoin block. If a miner wins one share per bitcoin block (so has 0.1% of the bitcoin hashrate) he'd have a monthly revenue variance of about 1.5%. Variance reduction is the main point of pools in the first place, and this is really the floor in variance of where miners want to be, given their margins. If you're a smaller miner than that you either accept higher variance or want to find another payout solution. However 0.1% of the network at today's hashrate of 800M Th/s is about 4000 S21-class devices. This is a fairly large operation corresponding to 13MW of power. For miners smaller than 4000 devices we need to find another solution.

This is where I think an eCash mint can come in. I've proposed Braidpool sub-pools for this, which is another instance of Braidpool that takes payment from shares in the parent pool, instead of Bitcoin. Shares in this sub-pool could be eCash tokens leveraging the corresponding Braidpool sub-pool instance for share information and validation. This could get us down to ~1.5% monthly variance for miners having as few as ~4 S21-class devices. Smaller miners than this (e.g. Bitaxe) could use sub-sub-pools, but at this point the payouts are so small that it's even difficult to put them on-chain because of the dust limit, so Lightning or eCash tokens are really required because of the small denominations.

I don't know that it makes much sense to worry about the BitAxe folks though, they're really playing a lottery, not expecting steady income. Running a BitAxe on a sub-pool is still a pretty nice lottery.

Cheers,
-- Bob

-------------------------

marathon-gary | 2025-01-08 19:26:22 UTC | #35

[quote="vnprc, post:32, topic:870"]
I think all you need is a lookup table of block template, header, and nonce data and a merkle sum tree of block header hashes. In order to prove an individual share’s validity you look up the block template, header, and nonce data, combine them into a full bitcoin block (with insufficient proof of work, probably), hash the header and provide a merkle proof for that header. The merkle sum tree is used to attest to the sum of difficulty of accepted shares. Ehash tokens are denominated in difficulty so this value can be directly compared to the sum of liabilities (mint proofs minus burn proofs (not including ecash redemptions)) from the Proof of Liabilities report. I haven’t fully scoped out the idea but it seems like a fairly straight-forward problem. Am I missing anything?
[/quote]

[quote="vnprc, post:33, topic:870"]
What payout formula should we use? Basic PPLNS, TIDES, or some other tweaked PPLNS algorithm?
[/quote]

I recommend checking out the [PPLN-JD post](https://delvingbitcoin.org/t/pplns-with-job-declaration/1099) as it offers a complimentary/orthogonal solution to share accounting and auditing. I believe eHash is doable using the PPLNS-JD accounting schema.

-------------------------

