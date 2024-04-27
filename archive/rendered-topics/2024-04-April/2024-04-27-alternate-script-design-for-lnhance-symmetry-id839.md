# Alternate script design for LNHANCE-Symmetry

reardencode | 2024-04-27 04:46:19 UTC | #1

I blame Lisa.

I'm writing my deck for BTC++ and I think that there's an alternate design for Lightning Symmetry channel scripts when specifically designed for use with LNHANCE instead of APO.

Basic background reading is Greg's [thread](https://delvingbitcoin.org/t/ln-symmetry-project-recap/359).

## Scripts

For LNHANCE-Symmetry, we have the following script structures:

### Channel outpoint

LNHANCE: `tr(musig2(a, b), CTV INTERNALKEY CSFS)`

APO: `tr(musig(a, b), 1 CHECKSIG)`

### Update

LNHANCE: `tr(settlement-hash, {CTV musig2(a, b) CSFS, INTERNALKEY CTV})`

APO: `tr(musig(a, b), {1 CHECKSIG, <settlement-sig> <G> CHECKSIG})`

## Discussion

In Greg's APO-Symmetry design, the taproot annex is used to store the hash of the settlement transaction which was used to generate the settlement sig for the settlement transaction corresponding to the update transaction being signed. This ensures that if Alice publishes an update and Bob has a later update, he can recover the taproot sibling hash needed to spend Alice's update.

In LNHANCE-Symmetry, the settlement transaction's CTV hash can be ground to a valid X on the curve and then used as the internal key for the taproot output. This makes the tapscripts identical from update to update ensuring that either party can store O(1) data and be able to spend any prior update.

## Witness sizes

### APO-Symmetry:

#### Initial update

* 33-byte annex
* 33-byte control block
* 2-byte script
* 65-byte signature
* Total: 4+33+33+2+65=137WU

#### Subsequent update

* 33-byte annex
* 65-byte control block
* 2-byte script
* 65-byte signature
* Total: 4+33+65+2+65=169WU

#### Settlement

* 65-byte control block
* 66+33+1-byte script
* Total: 2+65+(65+1+33+1+1)=168WU

### LNHANCE-Symmetry

#### Initial update

* 33-byte control block
* 3-byte script
* 64-byte signature
* Total: 3+33+3+64=103WU

#### Subsequent update

* 65-byte control block
* 3-byte script
* 64-byte signature
* Total: 3+65+3+64=135WU

#### Settlement

* 65-byte control block
* 2-byte script
* Total: 2+65+2=69WU

## Conclusion?

Where a 32-byte hash can be used for settlement, there is a significant weight saving for Lightning Symmetry channels from overloading the taproot internal key with the settlement hash.

-------------------------

