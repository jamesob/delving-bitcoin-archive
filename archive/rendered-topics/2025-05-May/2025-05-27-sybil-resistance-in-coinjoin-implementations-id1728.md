# Sybil resistance in coinjoin implementations

1440000bytes | 2025-05-27 22:05:11 UTC | #1

![20250528_0131_Spotless Brown Egg_remix_01jw9n7r2pff5rywp9v1ngveh5|690x460, 100%](upload://sO7WKNGuACXNa0u93tHrRWP7xy2.jpeg)

> Sybil attacks are an inherent threat to privacy in mixing schemes because a transaction between n apparent participants n − 1 of which are controlled by an attacker will fully link the victim’s coins on both sides of the coinjoin while giving the impression that the victim’s privacy has been improved.

## Wabisabi

Section 7.2.2 in [Wabisabi paper](https://eprint.iacr.org/2021/206.pdf) describes sybil attacks although the implementation does not provide resistance to these attacks. The attacker would need enough inputs in a coinjoin transactions and pay fees.

The [coordinator](https://kruw.io/) with the highest liquidity currently does not charge a coordinator fee.

Since WabiSabi is a coinjoin implementation based on centralized coordinator, a user must also trust the coordinator not to link inputs and outputs.

## Joinmarket

Joinmarket uses [fidelity bonds](https://joinmarket.sgn.space/fidelitybonds) for sybil resistance. This requires market makers to lock some bitcoin hence it increases the [cost of sybil attack](https://joinmarket.sgn.space/sybilresistance).

[Fidelity bonds](https://en.bitcoin.it/wiki/Fidelity_bonds) were proposed as a [solution](https://gist.github.com/chris-belcher/18ea0e6acdb885a2bfbdee43dcd6b5af) to sybil attacks by [Chris Belcher](https://x.com/chris_belcher_/status/1496577077751001093).

![image|505x492](upload://fDS3IyHgdYoU9zXM7o0t6hlgBMV.jpeg)

## Joinstr

Joinstr uses [aut-ct](https://github.com/AdamISZ/aut-ct) as the primary mechanism for sybil resistance, however fidelity bonds can also be used with aut-ct. There is an initiator who creates the pool and adds sybil requirements to join the pool. Everyone (maker and takers) needs to provide the proof for a successful coinjoin.

There is enough freedom for initiators to add different conditions for aut-ct. Example: Prove ownership of a UTXO with certain amount, age etc. This isn’t the UTXO which will be used for coinjoin. It isn’t possible in joinmarket with the protocol used right now. Another thing that makes it easier to provide liquidity while still resisting sybil attacks is that nobody needs to lock bitcoin.

Members need to pay a fee before joining a pool that uses paid nostr relay in joinstr. This isn’t the case with other coinjoin implementations.

Example:
- Alice creates a pool with `JM=True` because she already has a fidelity bond and using it for Joinmarket
- Bob and Carol join the pool
- Everyone shares aut-ct proof that proves they own a P2TR UTXO worth 0.1-0.2 BTC that is unspent until last block and aged more than 2016 blocks

I have shared other details in an earlier post: https://uncensoredtech.substack.com/p/market-makers-in-joinstr

## Conclusion

Wabisabi is vulnerable to sybil attacks. Sybil resistance in joinmarket is good enough as it increases the cost of the attack. However, joinstr provides the best sybil resistance among all the coinjoin implementations.

-------------------------

securitybrahh | 2025-05-28 10:28:12 UTC | #2

billions must coordinate over nostr.

-------------------------

