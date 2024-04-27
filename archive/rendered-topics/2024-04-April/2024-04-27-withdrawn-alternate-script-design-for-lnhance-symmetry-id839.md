# [WITHDRAWN] Alternate script design for LNHANCE-Symmetry

reardencode | 2024-04-27 14:14:25 UTC | #1

# WITHDRAWN

> * Let *t = hashTapTweak(p || km)*.

You can't back-calculate the internal key from the SPK and the script tree - would make taproot all in secure if that worked.

[details=Original post]
I blame Lisa.

I'm writing my deck for BTC++ and I think that there's an alternate design for Lightning Symmetry channel scripts when specifically designed for use with LNHANCE instead of APO.

Basic background reading is Greg's [thread](https://delvingbitcoin.org/t/ln-symmetry-project-recap/359).

## Scripts

For LNHANCE-Symmetry, we have the following script structures:

### Channel outpoint

LNHANCE: `tr(musig2(a, b), CTV INTERNALKEY CSFS)`

APO: `tr(musig(a, b), 1 CHECKSIG)`

### Update (tapleaves)

LNHANCE: `tr(settlement-hash, {<n+1> CLTV DROP CTV musig2(a, b) CSFS, INTERNALKEY CTV})`

APO: `tr(musig(a, b), {<n+1> CLTV DROP 1 CHECKSIG, <settlement-sig> <G> CHECKSIG})`

### Update (OP_IF)

LNHANCE: `tr(settlement-hash, DEPTH NOTIF INTERNALKEY CTV ELSE <n+1> CLTV DROP CTV musig2(a, b) CSFS ENDIF)`

APO: `tr(musig(a, b), DEPTH NOTIF <settlement-sig> <G> CHECKSIG ELSE <n+1> CLTV DROP 1 CHECKSIG ENDIF)`

## Discussion

In Greg's APO-Symmetry design, the taproot annex is used to store the hash of the settlement transaction which was used to generate the settlement sig for the settlement transaction corresponding to the update transaction being signed. This ensures that if Alice publishes an update and Bob has a later update, he can recover the taproot sibling hash needed to spend Alice's update.

In LNHANCE-Symmetry, the settlement transaction's CTV hash can be ground to a valid X on the curve and then used as the internal key for the taproot output. This makes the tapscripts identical from update to update ensuring that either party can store O(1) data and be able to spend any prior update.

## Witness sizes

### Summary

Variant                      | Initial update | Subsequent update | Settlement
:--------------------------- | :------------- | :---------------- | :---------
APO-Symmetry (tapleaves)     | 137WU          | 176WU             | 168WU
APO-Symmetry (OP_IF)         | 137WU          | 249WU             | 149WU
LNHANCE-Symmetry (tapleaves) | 136WU          | 207WU             | 69WU
LNHANCE-Symmetry (OP_IF)     | 136WU          | 149WU             | 83WU

[details=weight calculations]

### APO-Symmetry:

#### Initial update

* 33-byte annex
* 33-byte control block
* 2-byte script
* 65-byte signature
* Total: 4+33+33+2+65=137WU

#### Subsequent update (tapleaves)

* 33-byte annex
* 65-byte control block
* 9-byte script
* 65-byte signature
* Total: 4+33+65+9+65=176WU

#### Settlement (tapleaves)

* 65-byte control block
* 66+33+1-byte script
* Total: 2+65+(65+1+33+1+1)=168WU

#### Subsequent update (OP_IF)

* 33-byte annex
* 33-byte control block
* 114-byte script
* 65-byte signature
* Total: 4+33+33+114+65=249WU

#### Settlement (OP_IF)

* 65-byte control block
* 114-byte script
* Total: 2+33+114=149WU

### LNHANCE-Symmetry

#### Initial update

* 33-byte control block
* 3-byte script
* 32-byte hash
* 64-byte signature
* Total: 4+33+3+64+32=136WU

#### Subsequent update (tapleaves)

* 65-byte control block
* 42-byte script
* 32-byte hash
* 64-byte signature
* Total: 4+65+42+64=207WU

#### Settlement (tapleaves)

* 65-byte control block
* 2-byte script
* Total: 2+65+2=69WU

#### Subsequent update (OP_IF)

* 33-byte control block
* 48-byte script
* 32-byte hash
* 64-byte signature
* Total: 4+33+48+64=149WU

#### Settlement (OP_IF)

* 33-byte control block
* 48-byte script
* Total: 2+33+48=83WU
[/details]

## Conclusion?

Where a 32-byte hash can be used for settlement, there is a significant weight saving for Lightning Symmetry channels from overloading the taproot internal key with the settlement hash.

### Edit: Added OP_IF variants

The APO-Symmetry OP_IF variant favors force closes that are not challenged significantly. It moderately reduces overall weight when a close is not challenged and significantly increases overall weight when it is challenged.

LNHANCE-Symmetry already favors non-challenged closes significantly, and the OP_IF variant reduces that. It slightly increases overall weight when a close is not challenged, and significantly reduces overall weight when it is challenged.

Which of these variants might be preferred when building production Lightning Symmetry depends on the relative importance they give to unchallenged force close weight vs. the cost of challenge when necessary.

Edit: not as great a savings when i properly include the ctv hash in the update witness stack. Sigh. OP_TEMPLATEHASH (or TXHASH) in tapscript only would improve the gains.
[/details]

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

reardencode | 2024-04-27 14:09:39 UTC | #3

Hey AJ, thanks for your response!

[quote="ajtowns, post:2, topic:839"]
* This is missing the `<n> CLTV` step that makes eltoo work
[/quote]
Yep, left it out of both, so doesn't change the comparison really.

[quote="ajtowns, post:2, topic:839"]
* `<settlement-sig> <G> CHECKSIG` is just a more expensive way of doing `<h> CTV`; it’s not surprising that changing this to CTV makes it cheaper. I believe you can make it cheaper still by using [scriptless scripts to reveal a signature](https://github.com/instagibbs/bolts/issues/5) instead of doing the CTV-ish approach.
[/quote]
Yeah, CTV+APO is even more efficient, I'm just comparing an APO-only or CTV+CSFS-only world.

[quote="ajtowns, post:2, topic:839"]
I don’t think this works at all? If you grind the IPK, the scripts you end up with are something like:

* <n+1> CLTV DROP musig(A,B) CHECKSIG
* INTERNALKEY CTV

The scriptPubKey is then calculated as `IPK + Hash(IPK, Hash(Hash(S1), Hash(S2)))`.

But to spend using the first path, you need the witness stack to contain `IPK, Hash(S2), S1, signature`. But to know what `IPK` is, you need to know what the settlement tx corresponding to this update tx is, and to redo the grinding. The ln-symmetry approach to solving that is just to stick it into the annex of the update tx that you’re looking to spend.
[/quote]

~~Ah, but because the full script content is deterministic, you can compute the settlement hash / IPK by calculating the the taptweak (`t`) and then `IPK=SPK-t*G`. By moving the pieces around we solve the data availability problem.~~

Goddamnit, this doesn't work at all. My cleverness defeated. You need to know the IPK to calculate the tweak - otherwise the whole taproot construction would be insecure (any script tree could spend any outpoint).

[quote="ajtowns, post:2, topic:839"]
I think lnhance would be more efficient with an IF than two leafs, namely:
[/quote]

~~Yeah. That applies equally to both APO-Symmetry and LNHANCE-Symmetry, but in a real implementation it would indeed be advisable to use an `IF` instead of two tapleaves.~~

Ohhh!!! It doesn't apply to both, because the APO settlement leaf is large enough that it makes updates worse. Nice. I'll update OP.

[quote="ajtowns, post:2, topic:839"]
* * lnhance: 33B annex, 33B control block, 48B script, 64B signature, 32B ctv hash (210B)
[/quote]

A stray annex in this line

-------------------------

