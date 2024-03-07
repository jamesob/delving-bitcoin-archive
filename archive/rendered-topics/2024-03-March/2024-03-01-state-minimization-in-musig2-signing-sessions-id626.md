# State minimization in MuSig2 signing sessions

salvatoshi | 2024-03-01 15:24:01 UTC | #1

[BIP-0327](https://github.com/bitcoin/bips/blob/b3701faef2bdb98a0d7ace4eedbeefa2da4c89ed/bip-0327.mediawiki) discusses at length the necessity to keep some state during a signing session. However, a "signing session" in BIP-0327 only refers to the production of a single signature.

In the typical signing flow of a wallet, it's more logical to consider a _session_ at the level of an entire transaction. All transaction inputs are likely obtained from the same [descriptor containing musig()](https://github.com/bitcoin/bips/pull/1540), with the signer producing the pubnonce/signature for all the inputs at once.

Therefore, in the flow of BIP-0327, you would expect at least _one MuSig2 signing session per input_ to be active at the same time. In the context of hardware signing device support, that's somewhat problematic: it would require to persist state for an unbounded number of signing sessions, for example for a wallet that received a large number of small UTXOs. Persistent storage is often a scarce resource in embedded signing devices, and a naive approach would likely impose a maximum limit on the number of inputs of the transactions, depending on the hardware limitations.

In this post, I draft an approach that is compatible with, and builds on top of BIP-0327 in order to define a _psbt-level session_ with only a small amount of state persisted on the device. This approach enables the completion of a full signing flow for a PSBT spending from a musig-based wallet by synthetically generating the necessary state for each individual MuSig2 session.

## Signing flow with synthetic randomness

### Synthetic generation of BIP-0327 state

This section presents the core of the idea, while the next section makes it more precise in the context of signing devices.

In BIP-0327, the internal state that is kept by the signing device is essentially the *secnonce*, which in turn is computed from a random number _rand'_, and optionally from other parameters of _NonceGen_ which depend on the transaction being signed.

The core idea in this post is to compute a global random `rand_root`; then, for the *i*-th input and for the *j*-th `musig()`  key that the device is signing for in the [wallet policy](https://github.com/bitcoin/bips/pull/1389), one defines the *rand'* in _NonceGen_ as:

$\qquad rand_{i,j} = SHA256(rand\_root || i || j)$

In the concatenation, a fixed-length encoding of $i$ and $j$ is used in order to avoid collisions. That is used as the *rand'* value in the *NonceGen* algorithm for that input/KEY pair.

The *j* parameter allows to handle wallet policies that contain more than one `musig()` key expression involving the signing device.

### Signing flow in detail

This section describes the handling of the psbt-level sessions, plugging on top of the default signing flow of BIP-0327.

We assume that the signing device handles a single psbt-level session; this can be generalized to multiple parallel psbt-level sessions, where each session computes and stores a different `rand_root`.

In the following, a _session_ always refers to the psbt-level signing session; it contains `rand_root`, and possibly any other auxiliary data that the device wishes to save while signing is in progress.

**Phase 1: pubnonce generation:** A PSBT is sent to the signing device, and it does not contain any pubnonce.
- If a session already exists, it is deleted from the persistent memory.
- A new session is created in volatile memory.
- The device produces a fresh random number $rand\_root$, and saves it in the current session.
- The device generates the randomness for the $i$-th input and for the $j$-th key as: $rand_{i,j} = SHA256(rand\_root || i || j)$.
- Compute each *(secnonce, pubnonce)* as per the `NonceGen` algorithm.
- At completion (after all the pubnonces are returned), the session secret $rand\_root$ is copied into the persistent memory.

**Phase 2: partial signature generation:** A PSBT containing all the pubnonces is sent to the device.
- *A copy of the session is stored in the volatile memory, and the session is deleted from the persistent memory*.
- For each input/musig-key pair $(i, j)$:
  - Recompute the pubnonce/secnonce pair using `NonceGen` with the synthetic randomness $rand_{i,j}$ as above.
  - Verify that the pubnonce contained in the PSBT matches the one synthetically recomputed.
  - Continue the signing flow as per BIP-0327, generating the partial signature.

## Security considerations
### State reuse avoidance
Storing the session in persistent memory only at the end of Phase 1, and deleting it before beginning Phase 2 simplifies auditing and making sure that there is no reuse of state across signing sessions.

### Security of synthetic randomness

Generating $rand_{i, j}$ synthetically is not a problem, since the $rand\_root$ value is kept secret and never leaves the device. This ensures that all the values produced for different $i$ and $j$ not predictable for an attacker.

### Malleability of the PSBT
If the optional parameters are passed to the _NonceGen_ function, they will depend on the transaction data present in the PSBT. Therefore, there is no guarantee that they will be unchanged the next time the PSBT is provided.

However, that does not constitute a security risk, as those parameters are only used as additional sources of entropy in _NonceGen_. A malicious software wallet can't affect the _secnonce_/_pubnonce_ pairs in any predictable way. Changing any of the parameters used in _NonceGen_ would cause a failure during Phase 2, as the recomputed _pubnonce_ would not match the one in the psbt.

## Generalization to multiple PSBT signing sessions

The approach described above assumes that no attempt to sign a PSBT containing for a wallet policy containing `musig()` keys is initiated while a session is already in progress.

It is possible to generalize this to an arbitrary number of parallel signing sessions. Each session could be identified by a `session_id` computed by hashing enough information to (practically) uniquely identify the transaction being signed (making sure that the updated psbt presented in Phase 2 is unchanged); for example, it could be a commitment to the `txid` of the unsigned transaction contained in the PSBT, and the wallet policy used for signing.

## Acknowledgments

I would like to thank Yannick Seurin for numerous discussions on the topic, and for reviewing an earlier draft of this post.

Mistakes are my own.

-------------------------

cmd | 2024-03-02 22:11:50 UTC | #2

FYI BitEscrow already does this for parallel musig2 signing sessions. We even use the terms "root_nonce" and "session_id".

https://github.com/BitEscrow/escrow-core

You can even compute branching paths of the initial nonce values in order to run a VM using DLCs. Though it's not as sexy as it sounds in practice.

-------------------------

real-or-random | 2024-03-06 17:23:09 UTC | #3

Have you considered CounterNonceGen from the BIP? Is the problem that it needs the secret key, but you may not want to access it at nonce generation time?

-------------------------

salvatoshi | 2024-03-06 18:17:29 UTC | #4

Yes, @LLFourn suggested the same on Twitter.
Since I'm working towards an implementation on Ledger devices, I already have access to the TRNG from the Secure Element, while I would have to implement a secure atomic counter myself if I wanted to use CounterNonceGen.

Either way, I think the same approach above could be used with CounterNonceGen, just replacing *rand_root* with the atomic counter.

Perhaps there are simpler approaches to handle the psbt-level sessions with CounterNonceGen, instead of using the *(i,j)* hierarchy above; but I think that in order to handle psbt-level signing sessions securely you'd still want to commit to the initial counter and the number of signatures you'll produce for that psbt as part of the session state indexed by `session_id`, particularly if you allow multiple psbt signing flows in parallel.
Overall, I suspect this might be harder to audit in terms of no-nonce-reuse.

-------------------------

real-or-random | 2024-03-07 07:04:09 UTC | #5

Oh, I think what I had in mind is to pass the $(i,j)$ pair as `extra_in` to NonceGen, and use `rand' := rand_root`. But yeah, that's i) not exactly CounterNonceGen, and ii) not clearly better.

Note that the most natural cryptographic tool to generate $rand_{i,j}$ from `rand_root` and $(i,j)$, at least from a theory point of view, is an RNG (e.g., ChaCha20) instead of a full-blown hash function. But a hash function is totally fine, it serves as a good RNG, it's just computationally more expensive. In some sense, the same applies to the internals of nonce generation in BIP327 and even BIP340. We simply picked SHA256 since implementations need it anyway for the challenge hash of the signature, and it's a bit perhaps a bit more conservative (or overkill, in other words). 

nit: I'd call it `seed` or `psbt_seed` or `rand_seed` instead of `rand_root`. I think that's the most common word for such a thing.

[quote="salvatoshi, post:1, topic:626"]
It is possible to generalize this to an arbitrary number of parallel signing sessions. Each session could be identified by a `session_id` computed by hashing enough information to (practically) uniquely identify the transaction being signed (making sure that the updated psbt presented in Phase 2 is unchanged); for example, it could be a commitment to the `txid` of the unsigned transaction contained in the PSBT, and the wallet policy used for signing.
[/quote]

Hashing the commitment to the `txid` and the wallet policy sounds dangerous to me. What if you get a second PSBT for the same transaction? (It may very well be the case that I'm misunderstanding ...)

-------------------------

salvatoshi | 2024-03-07 09:26:58 UTC | #6


[quote="real-or-random, post:5, topic:626"]
Hashing the commitment to the `txid` and the wallet policy sounds dangerous to me. What if you get a second PSBT for the same transaction? (It may very well be the case that I’m misunderstanding …)
[/quote]

The idea with the `session_id` is that it should make `id` collisions unlikely in practice, but a collision should not pose a security risk, and at most cause a signing failure.

If the second time a colliding psbt is presented with mutated parameters that affect NonceGen (any of the extra args), then for at least one $(i, j)$ pair the recomputed secnonce/pubnonce would be different, signing is aborted and the session destroyed.
If the colliding psbt is presented with mutated members that do *not* affect the output of NonceGen, then the changes in the second PSBT vs the first are irrelevant, as NonceGen would have been executed in with the same exact parameters if there was no mutation.

So unless I missed something, that should still be safe.

-------------------------

real-or-random | 2024-03-07 10:44:57 UTC | #7

[quote="salvatoshi, post:6, topic:626"]
If the colliding psbt is presented with mutated members that do *not* affect the output of NonceGen, then the changes in the second PSBT vs the first are irrelevant, as NonceGen would have been executed in with the same exact parameters if there was no mutation.
[/quote]

It seems to me that this is the unsafe case.

 - The attacker sends you a PSBT
 - You generate a `secnonce` and sign with it 
 - The attacker tells you that something went wrong, and sends you the same PSBT again with the same `session_id`
 - So you generate the exact same `secnonce` and sign again with it

This is precisely what should not happen. Or why would this attack be prevented?

-------------------------

salvatoshi | 2024-03-07 11:09:06 UTC | #8

When you complete a signature (Phase 2 above), that `session_id` has already been destroyed (beginning of Phase 2). A new PSBT with the same `session_id` would have to start again from Phase 1, and a new session with fresh randomness (`rand_root` in my post above) would be created (even if it has the same `session_id`).

The only malleability in the PSBT while a session is "active" is after the session is created in Phase 1, and before signatures are produced in Phase 2, which is what I was commenting about in the last post.

Perhaps I should make it more explicit in the description that Phase 2 fails immediately if a corresponding active `session_id` is not found.

-------------------------

real-or-random | 2024-03-07 12:26:19 UTC | #9

[quote="salvatoshi, post:8, topic:626"]
and a new session with fresh randomness (`rand_root` in my post above) would be created (even if it has the same `session_id`).
[/quote]

Oh, sure. If you draw a fresh `rand_root`, everything is alright. Sorry, I got confused over `session_id` vs `rand_root`. I was under the assumption that `session_id` from `rand_root` (or set it to the same value).

I think my confusion partly stems from the fact that we use the term `session_id` in the C implementation of MuSig2 (instead of `rand'`). This has also confused others in the past. (I've just commented on the PR: https://github.com/bitcoin-core/secp256k1/pull/1479/files#r1516062204)

-------------------------

salvatoshi | 2024-03-07 12:52:25 UTC | #10

Indeed, I will rename it to `psbt_session_id` to be more explicit. Sorry for being too handwavy on the 'Generalization' section - I added it for completeness as I think it's practically useful, but it wasn't the main focus, nor the part that worried me.

Thanks a lot for the comments, that's very helpful!

<br>
EDIT: oh well, it looks like I'm out of time to edit the original post. But I will take into account the comments on the naming when I write the code!

-------------------------

