# BIP352 private key formats

junderw | 2025-10-25 00:43:40 UTC | #1

I was mulling over SP support for things like BTCPayServer (which normally takes an xpub).

This would require some sort of encoding of the scan private + spend public combo.

I was thinking of all sorts of hacks, like using the xpub chaincode as the scan private (LOL) but obviously this breaks a lot of stuff with BIP352 so I binned the idea.

I think it would be a good idea to define a string format for:

1. B_scan + B_m (currently defined in BIP352)
2. b_scan + B_m (for delegated scanning)
3. b_scan + b_spend (for spending and generating new B_m SP addresses)

I was thinking we could just modify the HRP of `sp` and `tsp`.

Delegated scan should be `spscan` and `tspscan`

All private should be `sppriv` and `tsppriv`

Thoughts? I apologize if this has been brought up.

-------------------------

