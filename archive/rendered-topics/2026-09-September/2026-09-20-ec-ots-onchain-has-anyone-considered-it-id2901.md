# EC OTS (onchain). Has anyone considered it?

AdamISZ | 2026-09-20 17:57:00 UTC | #1

Motivation: I was thinking: could you validate a threshold signature from some kind of attesting entity, in Script? (obviously assuming CSFS does not exist, as now).

We know the trick of Lamport or Witnernitz OTS being verified in Script. A preimage per bit value (or per digit value in other bases). If more than one digit value is revealed, that's equivocation and so can be trivially punished in a clause.

We know that it's rather large and scales with message size. So typically 256 bit messages needing kBs worth of script in terms of pubkeys and the corresponding signatures. Workable for some simple use cases. See BitVM etc. etc.

So what I was thinking is, I want to be able to verify a signature on a message (and put the message on the stack) to validate some rule based on 'an external attestor attests to message m, and cannot equivocate without being punished', but have that external attestor be a threshold (let's say FROST specifically because we're in BIP340 land) signer. I don't think that can be done with hash based constructions (except in the very ugly way of a tuple of signatures and some logic, I guess; doesn't scale to large quora, very clearly). So what if you replaced the one-way hash function with a one-way EC scalar mult.

The idea would be: message is a series of bits (stick with binary here for now, for simplicity). Signer (the FROST aggregated entity) has some long-lived signing key call it X with dlog x. Now we use the Dryja "R-committed key" trick: for each bit (or digit, in general) we have a single published R value $R_i$. The attestor publishes $S_{i,0},S_{i,1}$; two curve points such that:

$s_{i,j} = k_i + H(j, R_i, X, \textrm{attestation context etc.})x$

Verifiable that $S_{i,j}$ is constructed correctly; if either of those two $s_{i,0}, s_{i,1}$ is published, it's verifiable as the signature, but it doesn't leak $x$ unless you equivocate and publish both. Then the trick would be that the Bitcoin Script tapleaf (say) has a set of checks like <$S_{i,j}$> OP_CHECKSIG. To satisfy (that clause in) the script you have to prove knowledge of $s_{i,j}$ by doing a signature over the transaction hash, just as normal. The result though is that you can feed in, with Script, all of the bits of the message, by these signature operations validating. And equivocation "prevented" in the discreet log contracts way.

I would guess that in typical cases outside of 'we want a quorum's aggregated signature on the message', using EC instead of hashes for this construct would make little sense. But for this specific case, it looks like the obvious way to do it.

Now, considering Winternitz instead of Lamport: the checksum does not really apply to this use-case where we only use a key once. But it doesn't matter, because you can't do the Winternitz trick here; you can't do a hash chain. we're mapping scalars into points and there's no EC_MUL in Script (if there was you should be able to do this I think; but then again you could probably also do much cleverer things, too).

But the idea of chunking and using not binary digits but say, base 16 or similar, does apply, I think.

-------------------------

