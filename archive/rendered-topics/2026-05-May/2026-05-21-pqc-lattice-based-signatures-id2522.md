# PQC: Lattice-based signatures

nkaretnikov | 2026-05-21 04:29:15 UTC | #1

Dear all,

I hate to contribute to the recent flood of PQC posts, but I think it’s an important issue that’s worth discussing.

In particular, what I usually see is various competing proposals without a clear winner.

So I’d like to bring everyone’s attention to this new post from Blockstream:
[https://blog.blockstream.com/schnorr-but-with-vectors-lattice-based-signatures-explained/](https://blog.blockstream.com/schnorr-but-with-vectors-lattice-based-signatures-explained/)   

This post is interesting because unlike a lot of PQC discussions, it actually includes a comparison table of various approaches, where lattices seem to come out ahead.

This raises a few questions.

Since lattices are not a new topic in cryptography, why has Blockstream focused their efforts on hash-based approaches so far?
Are hashes seen as a more conservative choice?

Given the problems with hashes outlined in the post, are lattices actually the current most likely candidate for a PQC implementation?
If so, should the community effort be focused on lattices instead of other proposals?
Or is the comparison table not telling the whole story?

I’d like to hear your thoughts on the topic.

Thanks,
Nikita

-------------------------

nkaretnikov | 2026-05-21 06:54:18 UTC | #2

I also posted this to the bitcoindev mailing list, which generated some replies:

https://groups.google.com/g/bitcoindev/c/nMO4hyEm5qc

-------------------------

ion_minus | 2026-05-28 09:18:17 UTC | #3

Dear Nikita,

Thank you for bringing this up as I fully resonate with your consideration. At least in a high level perspective, lattices seem to be a clear winner for general post-quantum cryptography. It is not without some merit that 3 out of the 4 NIST finalists are based on lattices and are clearly stated as primary options, leaving the single hashed-based candidate as a back up. While I can understand that opting for hashed-based maintains the same security assumptions for Bitcoin, I find that the solid security proofs of lattices and the worst-case to average-case hardness reductions are very lucrative. But yet again, devil is in the details which makes the whole endeavor of upgrading to pqc quite interesting.

Kind regards,
ion-

-------------------------

