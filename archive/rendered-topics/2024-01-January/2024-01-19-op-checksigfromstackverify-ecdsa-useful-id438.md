# OP_CHECKSIGFROMSTACKVERIFY ECDSA useful?

reardencode | 2024-01-19 05:50:24 UTC | #1

@ajtowns brought up the [question](https://github.com/bitcoin-inquisition/binana/pull/1#discussion_r1456957199) of whether it's worth including ECDSA for OP_CHECKSIGFROMSTACKVERIFY in legacy scripts, and then we discussed it briefly on the [optech review](https://twitter.com/bitcoinoptech/status/1748048222705103231) today.

So delving: Should OP_CHECKSIGFROMSTACKVERIFY support only BIP340 Schnorr signing with 32-bytes keys or also ECDSA signing with 33-byte keys having leading bytes `0x02` or `0x03`?

If it should support ECDSA signing, should it do so on legacy/witnessv0 scripts only or also on tapscript `0xc0`?

@murch leaned toward no, as has everyone else I've talked to about this (except @JeremyRubin, but even he didn't have a firm justification for including ECDSA).

-------------------------

