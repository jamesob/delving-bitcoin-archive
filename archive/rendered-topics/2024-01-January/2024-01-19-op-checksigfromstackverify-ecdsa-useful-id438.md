# OP_CHECKSIGFROMSTACKVERIFY ECDSA useful?

reardencode | 2024-01-19 05:50:24 UTC | #1

@ajtowns brought up the [question](https://github.com/bitcoin-inquisition/binana/pull/1#discussion_r1456957199) of whether it's worth including ECDSA for OP_CHECKSIGFROMSTACKVERIFY in legacy scripts, and then we discussed it briefly on the [optech review](https://twitter.com/bitcoinoptech/status/1748048222705103231) today.

So delving: Should OP_CHECKSIGFROMSTACKVERIFY support only BIP340 Schnorr signing with 32-bytes keys or also ECDSA signing with 33-byte keys having leading bytes `0x02` or `0x03`?

If it should support ECDSA signing, should it do so on legacy/witnessv0 scripts only or also on tapscript `0xc0`?

@murch leaned toward no, as has everyone else I've talked to about this (except @JeremyRubin, but even he didn't have a firm justification for including ECDSA).

-------------------------

moonsettler | 2024-01-19 13:45:27 UTC | #2

one reason would be to support it for any organization that relies on ECDSA MPC as a vetted established practice. if you have an audited process, libraries and tools that work, for an entity like Coinbase or other large custodians, you won't just switch to a "brand new" cryptography and libraries and tools.

even tho Schnorr has cryptographic proof of being secure construction, and generally makes sense to switch to it long term, the MPC stuff like MuSig and FROST are still barely out of development. i can see how certain entities would be in no rush to switch.

as i remember it was even hard and excruciatingly slow to get exchanges to properly support P2TR outputs for withdrawals. and that's basically nothing in comparison.

-------------------------

reardencode | 2024-01-19 14:30:14 UTC | #3

Thanks @moonsettler ! So far that's the only argument I've actually heard for ECDSA, and it does only apply to legacy scripts (as such entrenched systems would need to be upgraded for any signing operation on Tapscript). So this would be an argument for ECDSA in legacy/witnessv0 only.

Keeping my ears out for any other reasons we may be missing for ECDSA in new sigops, but not expecting much :slight_smile: 

So, the strongest form of this argument would be that there may be existing custody operations that have a ECDSA TSS implementation they trust and want to take advantage of some protocol like a post-signed vaults or delegation that use CSFSV. They have the resources to implement the new protocol, but not to audit FROST (or MuSig2 depending on their specifics) and move to Tapscript for that application.

I think this remains weak. Leaning toward making CSFS(V) BIP340 Schnorr only (but upgradeable with non-32-byte keys at a later time if I'm wrong).

-------------------------

