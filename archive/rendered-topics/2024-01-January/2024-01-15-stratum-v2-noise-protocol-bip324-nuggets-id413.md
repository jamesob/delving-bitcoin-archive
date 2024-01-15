# Stratum v2 Noise Protocol: BIP324 nuggets?

sjors | 2024-01-15 13:54:13 UTC | #1

The Stratum v2 [specification](https://github.com/stratum-mining/sv2-spec) introduces a [protocol](https://github.com/stratum-mining/sv2-spec/blob/73ec7a6f52924b061386272707da0f692f8dea66/04-Protocol-Security.md) for its end-to-end encrypted communication based on the [Noise Protocol Framework](https://github.com/stratum-mining/sv2-spec/blob/73ec7a6f52924b061386272707da0f692f8dea66/04-Protocol-Security.md).

It uses cryptographic primitives that were picked to make integration into Bitcoin Core easier. Bitcoin Core plays the Template Provider role as part of the [Template Distribution protocol](https://github.com/stratum-mining/sv2-spec/blob/73ec7a6f52924b061386272707da0f692f8dea66/07-Template-Distribution-Protocol.md). There is a [draft PR](https://github.com/bitcoin/bitcoin/pull/28983) to Bitcoin Core to implement this role, that yours truly took over.

Although the protocol is similar to [BIP324](https://github.com/bitcoin/bips/blob/deae64bfd31f6938253c05392aa355bf6d7e7605/bip-0324.mediawiki) peer-to-peer encryption, the spec was written earlier. Stratum v2 can't switch over to BIP324 for the time being, because:

1. Sunk cost / momentum
2. Server authentication (sending work to the right pool)
3. Message routing (this part is not very clear to me, but it _may_ be useful for proxies to forward messages to and from individual mining machines based on the `channel_id` field in the header, without having to decrypt the payload)

Issue (2) may be addressed in the longer run with an authentication extension to BIP324.

Issue (3) - if it actually matters - may be more difficult to address, because BIP324 splits messages differently: the length is encrypted in 3 bytes, followed by a single blob with both header and payload.

Despite (1) I believe there's still some room to tweak the Stratum v2 spec. So my question to you all is: which little nuggets from BIP324 could be ported over?

I'm looking for potential improvements that don't require sv2 implementers to do any drastic overhaul. At the same time these changes might make a future transition to BIP324 easier.

Things that already the same (AFAIK):

1. The use of ChaCha20 and Poly1305 in AEAD mode
2. The use of sha256 to mix keys

Perhaps easy to implement:

1. Use tagged hashes

Slightly more difficult to implement:

2. Use EllSwift encoding and ECDH: https://github.com/stratum-mining/sv2-spec/pull/66 

Initially I didn't want to propose (2), but it turned about implementations were not actually following the spec when it came to ECDH, so it might be worth changing the spec to remove the ambiguity.

This rest of this list is so small it's probably not worth bothering.

-------------------------

