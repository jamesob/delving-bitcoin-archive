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

junderw | 2025-10-25 02:16:33 UTC | #2

(post deleted by author)

-------------------------

