# Alternate script design for LNHANCE-Symmetry

reardencode | 2024-04-27 06:35:35 UTC | #1

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
* 32-byte hash
* 64-byte signature
* Total: 4+33+3+64+32=136WU

#### Subsequent update

* 65-byte control block
* 35-byte script
* 32-byte hash
* 64-byte signature
* Total: 4+65+35+64=200WU

#### Settlement

* 65-byte control block
* 2-byte script
* Total: 2+65+2=69WU

## Conclusion?

Where a 32-byte hash can be used for settlement, there is a significant weight saving for Lightning Symmetry channels from overloading the taproot internal key with the settlement hash.

Edit: not as great a savings when i properly include the ctv hash in the update witness stack. Sigh. OP_TEMPLATEHASH in tapscript only would bring back the sick gains.

-------------------------

ajtowns | 2024-04-27 06:54:29 UTC | #2

[quote="reardencode, post:1, topic:839"]
LNHANCE: `tr(settlement-hash, {CTV musig2(a, b) CSFS, INTERNALKEY CTV})`

APO: `tr(musig(a, b), {1 CHECKSIG, <settlement-sig> <G> CHECKSIG})`
[/quote]

Two things:
 * This is missing the `<n> CLTV` step that makes eltoo work
 * `<settlement-sig> <G> CHECKSIG` is just a more expensive way of doing `<h> CTV`; it's not surprising that changing this to CTV makes it cheaper. I believe you can make it cheaper still by using [scriptless scripts to reveal a signature](https://github.com/instagibbs/bolts/issues/5) instead of doing the CTV-ish approach.

[quote="reardencode, post:1, topic:839"]
In LNHANCE-Symmetry, the settlement transaction’s CTV hash can be ground to a valid X on the curve and then used as the internal key for the taproot output. This makes the tapscripts identical from update to update ensuring that either party can store O(1) data and be able to spend any prior update.
[/quote]

I don't think this works at all? If you grind the IPK, the scripts you end up with are something like:

 * <n+1> CLTV DROP musig(A,B) CHECKSIG
 * INTERNALKEY CTV

The scriptPubKey is then calculated as `IPK + Hash(IPK, Hash(Hash(S1), Hash(S2)))`. 

But to spend using the first path, you need the witness stack to contain `IPK, Hash(S2), S1, signature`. But to know what `IPK` is, you need to know what the settlement tx corresponding to this update tx is, and to redo the grinding. The ln-symmetry approach to solving that is just to stick it into the annex of the update tx that you're looking to spend.

I think lnhance would be more efficient with an IF than two leafs, namely:

 * `IFDUP IF <n> CLTV DROP CTV [musig2(a,b)] CSFS ELSE INTERNALKEY CTV ENDIF`

which is always 48 bytes, versus 32+42 bytes (74 bytes) or 32+2 bytes (34 bytes).

I think that makes the comparison:

 * update tx spending funding tx:
    * ln-symmetry: 33B annex, 33B control block, 2B script, 65B signature (133B)
    * lnhance: 33B annex, 33B control block, 3B script, 64B signature, 32B ctv hash (165B)

 * update tx spending prior update tx:
    * ln-symmetry: 33B annex, 65B control block, 8B script, 65B signature (171B)
    * lnhance: 33B annex, 33B control block, 48B script, 64B signature, 32B ctv hash (210B)

 * settlement tx spending update tx:
    * ln-symmetry: 65B control block, 100B script (165B)
    * lnhance: 33B control block, 48B script (81B)

In normal use that's 298B versus 246B, so a 52B advantage to lnhance. lnhance would be improved by another 32B if TXHASH was available rather than CTV for the "CTV CSFS" construct, equally, just using APO there would also save that much. ln-symmetry would be improved by 64B just by switching to CTV. Switching ln-symmetry to the scriptless script approach would save ~128B I think.

-------------------------

