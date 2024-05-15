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

