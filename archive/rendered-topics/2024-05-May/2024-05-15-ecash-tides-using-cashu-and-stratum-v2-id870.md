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

1440000bytes | 2024-05-16 01:41:35 UTC | #6

[quote="davidcaseria, post:5, topic:870"]
Mining pools are already custodial and vulnerable to state action, such as enforcing KYC requirements (which some do).

The possibility of immediate liquidity of mining rewards is self-evidently beneficial.
[/quote]

[TIDES](https://bitcoin.stackexchange.com/questions/120719/how-does-ocean-s-tides-payout-scheme-work) is only used by Ocean mining pool. Based on a private conversation with CTO of Ocean (cannot share), they plan to stay non-KYC hence doing payouts in the coinbase transaction.

Mining rewards are essentially an IOU in this case until they are redeemed for bitcoin.

-------------------------

davidcaseria | 2024-05-16 10:07:06 UTC | #7

I understand your point about Ocean’s current KYC policy, but I still believe that your argument is weak. Not all of Ocean’s payouts are in the coinbase transaction, and with a minimum payment threshold, shares in the share log already act as IOUs.

I don't think it's an accurate characterization to say there are no benefits and it introduces new trade-offs, because in reality, these legal considerations already exist for a pool using TIDES and are not as clear as you suggest.

I believe this idea is worth discussing on its own merit, even though implementing it seems like a long shot due to the need for custom stratum protocol implementation regardless of potential legal issues.

-------------------------

