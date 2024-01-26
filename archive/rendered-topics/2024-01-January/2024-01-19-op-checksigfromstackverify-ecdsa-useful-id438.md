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

reardencode | 2024-01-19 17:36:46 UTC | #4

Super Testnet on X mentioned DER encoded ECDSA signature length as a form of proof of work as a reason to include ECDSA signing in all script types:
https://twitter.com/super_testnet/status/1748370856814715015

-------------------------

harding | 2024-01-25 22:41:35 UTC | #5

Re: ECDSA for allowing a type of proof of work via DER encoding.  I think a naive implementation of that could suffer from several gotchas, e.g. see:

- [Half a puzzle](https://web.archive.org/web/20161101112330/https://www.blockstream.com/half-a-puzzle/)
- https://bitcointalk.org/index.php?topic=1118704

I think it could be made safe, but you'd probably want a serious cryptographer to spend a significant amount of time thinking about it before any serious money was committed to any contract depending on that feature.

If there's significant demand for PoW-based contracts in tapscript, I think it'd be better in almost every way to enable opcodes that allow verifying SHA2-based PoW.

-------------------------

reardencode | 2024-01-25 23:40:42 UTC | #6

That makes sense to me. For example, if we do reactivate CAT similar PoW proofs could be done with plain hashes.

```
stack: <nonce> <hash_prefix>
script: <required_hash_suffix> CAT SWAP <preimage_suffix> CAT SHA256 EQUALVERIFY
```

(probably missing some details in ^ by allowing any length nonce, but it's the general idea)

In short there are better ways to achieve this than ECDSA (which matches my gut reaction).

-------------------------

