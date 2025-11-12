# BIP352 private key formats

junderw | 2025-10-25 02:19:06 UTC | #1

I was mulling over SP support for things like BTCPayServer (which normally takes an xpub).

This would require some sort of encoding of the scan private + spend public combo.

I was thinking of all sorts of hacks, like using the xpub chaincode as the scan private (LOL) but obviously this breaks a lot of stuff with BIP352 so I binned the idea.

I think it would be a good idea to define a string format for:

1. B_scan + B_m (currently defined in BIP352)
2. b_scan + B_spend (for delegated scanning and generating new B_m SP addresses)
3. b_scan + b_spend (for spending)

I was thinking we could just modify the HRP of `sp` and `tsp`. For private keys, leave the 0x00 byte in place of the 0x02/0x03 pubkey header bytes to keep the same length.

Delegated scan should be `spscan` and `tspscan`

All private should be `sppriv` and `tsppriv`

Thoughts? I apologize if this has been brought up.

-------------------------

nymius | 2025-10-27 20:20:28 UTC | #3

There is a transcript of a previous discussion related to descriptors that has similar ideas on encoding: [Bitcoin Core Dev Tech 2024: Silent Payment Descriptors](https://btctranscripts.com/bitcoin-core-dev-tech/2024-04/silent-payment-descriptors).

For (3), let me add that `b_scan` and `b_spend` are not enough to spend outputs by themselves, as they require the tweak material from the original transaction creating the output. Shouldn’t we also consider a format for “self contained” spending material for individual outputs? like `tweak` + `b_spend` , where `tweak` = `label + b_scan + shared secret` or even `b_scan + b_m + shared secret`.

For (2), it may also be useful to encode a range or limit for the number of applicable labels on the string itself.

-------------------------

junderw | 2025-11-01 03:50:00 UTC | #4

For 2 and 3 regarding labels and per-tx info (aggregate pubkey and smallest outpoint)

Per-tx info requiring a scan action is already stated in the BIP, so the assumption should be that the wallet will start a scan on import. Perhaps 2 and 3 should encode a block number for their “birthday” which would be the key asserting “no need to check below this block”

For labels, this could definitely be encoded, like an array of labels at the end. But I do think that even with no label the “0” (change) label should be checked.

If a wallet restores from BIP39 then the birthday should be assumed to be the latest block at the time which the BIP was published, and the only label should be the change label 0.

just some ideas. Might submit a PR to modify the BIP with the new encoding.

-------------------------

junderw | 2025-11-01 05:26:14 UTC | #5

I have created a BIP PR to get the ball rolling: https://github.com/bitcoin/bips/pull/2026

-------------------------

pyth | 2025-11-12 10:41:23 UTC | #6

is this format intented to be used as output descriptor?

-------------------------

