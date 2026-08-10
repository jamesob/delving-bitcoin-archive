# Universal opt-in replay protection?

moonsettler | 2026-08-09 22:43:19 UTC | #1

I would like to propose for consideration adding opt-in replay protection in case a future fork war<sup>1</sup> requires it for resolution!

A hostile soft or hard fork could make stakeholders fighting against it difficult by not providing replay protection. This was an interesting lesson learned from recent events where people tried to ancestrally split their coins, but it has proven to be fairly difficult.

One way to do it for example is standardizing a 34 byte Annex payload as follows:

`<0xFAF0><32-byte-prior-block-hash>`

Transaction validation would work as before, except when it comes to block validation or block template construction, in which case the current (valid header chain with most work) must contain the hash.

Since the Taproot signatures commit to the annex, the signatures could therefore commit to a specific chain in case of a chain split condition. Obviously empty Annex remains valid and such signatures are expected to be valid on all chains.

This allows for bribing miners to work on a specific block or splitting coins to dump on a specific chain. Even if only one of the chains honors this rule, the coins can still be safely split.

Thoughts?

*<sup>1</sup> One example that was brought up is a situation where the miners might try to hard fork more inflation to bitcoin. Majority hash leaving the legacy chain and no replay protection would hinder the UTXO holders ability to express their economic preference safely and effectively.*

-------------------------

ajtowns | 2026-08-09 23:15:57 UTC | #2

I think a block height and a hash suffix would probably be superior; if there's already been a split, you can specify which side of the fork with perhaps 5 or 6 bytes that way instead of 32 just by picking a height where the two block hashes end in different digits.

Some prior references:

https://gnusha.org/pi/bitcoindev/20190508044928.z52oaxevwcppkvna@erisian.com.au/

https://github.com/ariard/bips/blob/9dc3f74b384f143b7f1bdad30dc0fe2529c8e63f/bip-annex.mediawiki

-------------------------

