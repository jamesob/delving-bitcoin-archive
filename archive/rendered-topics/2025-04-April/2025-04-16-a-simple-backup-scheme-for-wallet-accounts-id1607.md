# A simple backup scheme for wallet accounts

salvatoshi | 2025-04-16 17:27:45 UTC | #1

For any wallet account that is not single-signature, backing up the descriptor is crucial, as its loss is likely to be catastrophic and lead to lost funds - even if the seeds that are in theory sufficient for recovery are not lost. This is true even for simple multisig wallets, as losing the knowledge of just a single xpub might make recovery impossible.

In lack of a standard, this led people to get creative with wallet backups schemes, with some even engraving the descriptor in metal.

I believe **this is a bad idea and should be discouraged**. Descriptors are not seeds, and should be treated radically differently both in theory and in practice.

In this post, I briefly introduce the context, and draft what I believe is the ideal structure for a *wallet account backup standard* that could be adopted by software wallets.

## Motivation: secrecy vs privacy

The seed is ***secret***, as it is what protects the key material that allows spending funds. Unauthorized access to the seed implies that attackers gain ownership of the funds (or at least the specific access controls that the keys are protecting). Hence, it is very valuable for an attacker to gain access to seed, and they will be willing to increase the cost and the sophistication of the attacks, because of the potential of high returns.

Therefore, for **seeds**:
- **digital copies are a high risk**: hardware signing devices have been built to keep the seeds in a secure enclave, separate from the machine the software wallet is running on.
- **redundant copies of the seed are a high risk**: the seed has to be physically protected, and multiple copies in multiple places inherently make their protection harder.

The descriptors (and their little brother *xpub*) is only ***private***: unauthorized access allows an attacker, to spy on your funds. That is bad, but not nearly as valuable as taking your funds. Attackers might use this to get information about you, and to inform further attacks, but *will lose interest once attempting an attack becomes too costly* or sophisticate.

For **descriptors**:
- **digital copies are unavoidable**: each of parties using the account *will* necessarily have a digital copy in their software wallet.
- additional **redundant copies are a very moderate risk**.

Therefore, having multiple copies of the descriptor, whether physical, digital, on your disk or on the cloud, is a valid mean to reduce the risk of loss of funds, unlike replicating the seed - which would incur a much higher risk.

I recommend timelocked recovery mechanisms like [liana](https://github.com/wizardsardine/liana) to effectively address the risk of loss of funds caused by mismanaging the seed.

So how do we backup descriptors and wallet policies, *the right way*?

Physical copies are easy - all you need is a printer. *Paper is fine when you have redundancy*.
Here I'm concerned with digital copies only.

## Desirable properties of a digital backup

- **Encrypted**: this allows to outsource its storage to untrusted parties, for example, cloud providers, or even public forums.
- Has **access control**: decrypting it should only be available to the desired parties (typically, a subset of the cosigners)
- **Easy to implement**: it should not require any sophisticated tools
- **Vendor-independent**: ideally, it should be easy to implement using any hardware signing device.
- **Deterministic**: the result of the backup is the same for the same payload.  Not crucial, but a nice-to-have.

## A simple deterministic encrypted backup scheme (draft)

We could just encrypt the payload with each of the xpubs of the parties that we want to be able to decrypt it.

*Idea 1*:
We can do better: we generate a random 32-byte symmetric secret $s$, encrypt $s$ with each of the public keys, and encrypt the payload with $s$. This reduces the backup size from $O(n \cdot |data|)$ to $O(n + |data|)$ for $n$ keys.

*Idea 2*:
For any party that knows the descriptor, there is nothing to protect. Therefore, *secrecy* is reduced to *secrecy for anyone who doesn't already have the descriptor*. We already have a key (the xpub) for each involved cosigner, so we can re-use the same key as the encryption key. However, using asymmetric encryption would require the private key for decryption. This is undesirable, as private keys might be kept in secure enclaves that might not be easily programmed with customized decryption logic. Instead, we reuse the entropy of the public key itself to generate a symmetric secret key, and use it to 'encrypt' the shared secret $s$. Therefore, for the party $i$ with public key $p_i$, we derive its symmetric secret $s_i = \operatorname{sha256}(``\textrm{BACKUP_INDIVIDUAL_SECRET}" \| p_i)$. This avoids asymmetric encryption, and only requires access to the public key from the secure enclave - a functionality that all the signing devices for bitcoin already provide.

*Idea 3*:
The only randomness in the process is the shared secret $s$. In order to make it fully deterministic, we can use the combined entropy of the descriptor to derive a deterministic, shared secret known to anyone who knows the descriptor. Assuming that the different xpubs involved in the descriptor/wallet policy are $p_1, p_2, \dots, p_n$ (in lexicographical order), a simple choice is: $s = \operatorname{sha256}(``\textrm{BACKUP_DECRYPTION_SECRET}" \| p_1 \| p_2 \| \dots \| p_n)$.

The next section puts these ideas together.

### The scheme

In the following, the payload $data$ that is being backed up is left unspecified, but it will include (at least) the descriptor or the [BIP388](https://github.com/bitcoin/bips/blob/c5220f8c3b43821efa3841a6c5e90af9ce5519e8/bip-0388.mediawiki) wallet policy. The operator $\oplus$ refers to the bitwise XOR.

- Let $p_1, p_2, \dots, p_n$, be the public keys in the descriptor/wallet policy, in increasing lexicographical order
- Let $s = \operatorname{sha256}(``\textrm{BACKUP_DECRYPTION_SECRET}" \| p_1 \| p_2 \| \dots \| p_n)$
- Let $s_i = \operatorname{sha256}(``\textrm{BACKUP_INDIVIDUAL_SECRET}" \| p_i)$
- Let $c_i = s \oplus s_i $
- encrypt the payload $data$ using the symmetric key $s$ using AES-GCM.

The backup is the list of $c_i$, followed by the encryption of $data$.

<br>
**Note**: this scheme should not be used with descriptors containing private keys (*xprv*). Most software wallets only include *xpubs*.

### Decryption

In order to decrypt the payload of a backup, the owner of a certain public key $p$ computes $s =  \operatorname{sha256}(``\textrm{BACKUP_INDIVIDUAL_SECRET}" \| p)$, and attempt the decryption of the payload with the key $c_i \oplus s$ for each of the provided $c_i$.

Decryption will succeed if and only if $p$ was one of the keys in the descriptor/wallet policy.


### Security considerations

A deterministic encryption, by definition, cannot satisfy the standard *[semantic security](https://en.wikipedia.org/wiki/Semantic_security)* property commonly used in cryptography; however, in our context, it is safe to assume that the adversary does not have access to plaintexts, and no other plaintext will be encrypted with the same secret $s$.

## Further work

I hope this serves as an inspiration for a more formal specification and implementation that software wallets can adopt.

-------------------------

reardencode | 2025-04-16 13:52:53 UTC | #2

@josh this has a lot in common with the method you were describing to me, except for the inscription and location features of yours. Perhaps getting Salvatore's standardized and then layering those parts on would help everyone :)

-------------------------

josh | 2025-04-16 15:57:46 UTC | #3

@reardencode Thanks for the shoutout!

@salvatoshi I'd love to share a tool I built that does something similar and perhaps collaborate on getting a multisig backup scheme standardized. I agree that this is a problem that needs to be solved.

The scheme you propose is simple and would appear to work, assuming SHA256 can be used as a secure KDF where the key is derived from a large subset of the data it is encrypting. There is at least one drawback, though:

If the encrypted descriptor is stored publicly or on a compromised server, an attacker who gains access to one secret gains knowledge of the existence of the multisig. This is not ideal if a user wants to protect themselves with a decoy single-sig wallet.

The scheme I'm using makes one significant change. In a $k$-of-$n$ multisig descriptor, the secret $s$ is split into $n$ shares using shamir secret sharing, where $k$ shares are needed to recover. Each share is then encrypted with one xpub, so that $k$ xpubs are needed to decrypt.

The other minor difference is that I leave the derivation paths in plaintext, so that a user knows how to derive their xpubs. Only the sensitive data is encrypted (the xpubs and master fingerprints).

As of now, the scheme only supports standard (non-taproot) multisig descriptors. In the future, I hope to generalize it to support decaying and non-decaying P2TR multisigs.

Here's the [GitHub repo](https://github.com/joshdoman/multisig-backup) and the corresponding [Delving post](https://delvingbitcoin.org/t/multisigbackup-com-backup-and-recover-a-k-of-n-descriptor-using-only-n-seeds/1430/4). The slides I presented at BitDevs ATL can be found [here](https://docs.google.com/presentation/d/1S6XN2jyeQsAkPeT4zvOaFIrOkGyzBnO8fbvcqg2Jlwk/edit?usp=sharing).

Let me know if you'd like to discuss!

-------------------------

1440000bytes | 2025-04-16 16:00:46 UTC | #4

[quote="salvatoshi, post:1, topic:1607"]
The descriptors (and their little brother *xpub*) is only ***private***: unauthorized access allows an attacker, to spy on your funds. That is bad, but not nearly as valuable as taking your funds.
[/quote]

Descriptors could include private keys: https://github.com/bitcoin/bitcoin/blob/cdc32994feadf3f15df3cfac5baae36b4b011462/doc/descriptors.md#including-private-keys

-------------------------

salvatoshi | 2025-04-16 17:28:51 UTC | #5

Good point, I added a note to mention to not use it with such descriptors.

-------------------------

salvatoshi | 2025-04-16 17:47:43 UTC | #6

Hi @josh, thanks for the comments! Somehow I missed your previous post, my bad.

[quote="josh, post:3, topic:1607"]
If the encrypted descriptor is stored publicly or on a compromised server, an attacker who gains access to one secret gains knowledge of the existence of the multisig. This is not ideal if a user wants to protect themselves with a decoy single-sig wallet.
[/quote]

I would say in my scheme there is no distinction between an attacker and 'someone who knows a secret', as it's designed to give knowledge of the descriptor precisely to the people who know at least one of the xpubs (or a subset of them, if desired). So if someone has the backup _and_ knows an xpub, they are expected to be able to decrypt.

[quote="josh, post:3, topic:1607"]
The scheme I’m using makes one significant change. In a $k$-of-$n$ multisig descriptor, the secret $s$ is split into $n$ shares using shamir secret sharing, where $k$ shares are needed to recover. Each share is then encrypted with one xpub, so that $k$ xpubs are needed to decrypt.
[/quote]

Shamir secret sharing, apart from adding at least *some* (arguably manageable) complexity, does not generalize well to wallet setups more complex than multisig. For example, in a setup where there is a time-locked recovery partner that can help retrieve the funds if the primary spending path became inaccessible, you want them to be able to decrypt the backup even with the single xpub.

If you don't want to enable some party to decode the backup, I think what will work better in practice is to have redundant copies of the backup, but do not give access to the backup to this third party (therefore, not posting it in a public place). Only if the primary spending path becomes lost, then they will be sent the encrypted backup.

[quote="josh, post:3, topic:1607"]
The other minor difference is that I leave the derivation paths in plaintext, so that a user knows how to derive their xpubs. Only the sensitive data is encrypted (the xpubs and master fingerprints).
[/quote]

This is a great idea; even just a list of all the derivation paths that appear in the key-origin information (without attribution to specific keys) would reduce the search space to at most $n$ xpubs when attempting decryption.

-------------------------

kloaec | 2025-04-23 19:29:50 UTC | #7

Great work, thanks for doing this salvatoshi.

Assuming we want all keys to form the secret, one way to "prevent" someone to be able to access it would be to simply to not generate their *c*<sub>i</sub>. Might be useful for some use-cases.

I'm also pondering if the *c*<sub>i</sub> should not use a different entropy, maybe a different path (standard, this time), from the same device. The major drawback is that all devices need to provide their second key for the backup to be performed, instead of just any person in the setup being able to create the encrypted backup.
The advantage is to not have to deal with unknown paths (let's not create a descriptor of the descriptor backup?), possibly even allowing hardware manufacturers to later on add security features to this specific path (confirm on screen to share it?), without breaking compatibility now.

Lastly, I feel like these files would benefit strongly from an **error correction mechanism.**
I obviously don't like the idea of sending it to the chain, so I assume most users won't have large number of replications.
In the case of Liana, assuming it's for disaster recovery or inheritance, it might be just one copy easily accessible. You want that one to be correct.

-------------------------

salvatoshi | 2025-04-24 08:31:45 UTC | #8

[quote="kloaec, post:7, topic:1607, full:true"]
Assuming we want all keys to form the secret, one way to "prevent" someone to be able to access it would be to simply to not generate their *c*<sub>i</sub>. Might be useful for some use-cases.
[/quote]

Indeed this is what I meant when I wrote above that the scheme has *access control*, but I could elaborate a bit more.

[quote="kloaec, post:7, topic:1607"]
I’m also pondering if the *c* i should not use a different entropy, maybe a different path (standard, this time), from the same device. The major drawback is that all devices need to provide their second key for the backup to be performed, instead of just any person in the setup being able to create the encrypted backup.
[/quote]

I agree with the advantage of reducing/eliminating the number of search paths for recovery. However, my main concern is that in practice, this adds a big dependency: the backup scheme now needs access to the necessary tech stack to access the hardware signers (notoriously, a non-trivial one), and the physical device needs to be available when the backup is created - so for example, a watch-only wallet that only receives the descriptor can't create the encrypted backup.

Instead, the more trivial scheme above is a pure function `f(descriptor) -> backup`, which I think is a big practical advantage.

[quote="kloaec, post:7, topic:1607"]
Lastly, I feel like these files would benefit strongly from an **error correction mechanism.** I obviously don’t like the idea of sending it to the chain, so I assume most users won’t have large number of replications. In the case of Liana, assuming it’s for disaster recovery or inheritance, it might be just one copy easily accessible. You want that one to be correct.
[/quote]

Can you elaborate on this? I can't think of situations where error correction would save the day.

I'd rather suggest implementations to get creative in how to make sure that there are multiple replicas of the backup. Save to google drive? Send via e-mail or DM to someone else (that's two copies)? Post on nostr/twitter/facebook?...
All of these options could be just a few clicks away with a good implementation in software wallets.

Many services doing any form of collaborative custody (or providing services for self-custody) could also consider storing the encrypted backup for their customer, so backup would be entirely transparent and add no UX cost at all.

-------------------------

kloaec | 2025-04-24 11:04:04 UTC | #9

[quote="salvatoshi, post:8, topic:1607"]
Can you elaborate on this? I can’t think of situations where error correction would save the day.
[/quote]
Over time, data storage decay. As we can't assume it will always be stored in a self healing manner, I would put such a self healing mechanism in the backup itself.
That being said, it might be overkill in the sense that most data storage media already implement an error correction mechanism (SD, HDD, SSD, optical disks).
Still, each mechanism has his own tolerance for failure, and sometimes the error correction assumes regular use.
To answer your questions: backups on an offline Flash, Magnetic or Optical storage, let to data rot for 10 years.
Sure, extra ECC won't prevent failure ultimately, but could prolong the viability of the data long for a negligible cost.

While having more backups is the better option, it's still not certain the user will easily be able to store and recover data in a large number of places.
Another point would be that a user might not want to leave copies everywhere (or in public, even if encrypted).

To conclude, I'd say that this scheme should still "strongly recommend" having multiple backups, in multiple places. But any one copy should also be pretty resilient against decay, more than your average data.

-------------------------

sjors | 2025-05-14 12:18:33 UTC | #10

[quote="salvatoshi, post:1, topic:1607"]
Therefore, *secrecy* is reduced to *secrecy for anyone who doesn’t already have the descriptor*.
[/quote]

I like this approach. Conversely, if you lose one of the signer keys (and the software wallet that goes with it), having _any_ of the other keys lets you recover this information. E.g. your house burns down along with one signing device, your Bitcoin Core node and its wallet, but you still have a signing device in a vault.

[quote="salvatoshi, post:1, topic:1607"]
In order to decrypt the payload of a backup, the owner of a certain public key $p$ computes $s = \operatorname{sha256}(``\textrm{BACKUP_INDIVIDUAL_SECRET}" \| p)$, and attempt the decryption of the payload with the key $c_i \oplus s$ for each of the provided $c_i$.
[/quote]

Calling this $s$ had me confused. But IIUC what you're actually doing here is to reconstruct your $s_i$, without knowing which $i$ is you. You try it against each $c_i$ to see if you found $s$. The way you know if that worked is if $s$ decrypts something sane.

You still need to figure out which xpub to use for $p$.

The [BIP87](https://github.com/bitcoin/bips/blob/60ac0e8feccb07f891fd984e4ed76105d2898609/bip-0087.mediawiki) account level xpub seems like a good candidate to recommended for this, where you may have to try multiple accounts for decryption.

Or you just add a (plain text) derivation hint to the backup.

Another approach is to pick a standard derivation path for these backups, but then you lose the nice property of being able to derive the backup key from a descriptor:

[quote="salvatoshi, post:8, topic:1607"]
Instead, the more trivial scheme above is a pure function `f(descriptor) -> backup`, which I think is a big practical advantage.
[/quote]

One such practical advantage:

[quote="kloaec, post:9, topic:1607"]
Over time, data storage decay. As we can’t assume it will always be stored in a self healing manner, I would put such a self healing mechanism in the backup itself
[/quote]

Backups can easily be verified against data corruption by any software that has the descriptor.

If you want to go one level fancier, you can even use this backup format for cloud sync.

[quote="salvatoshi, post:1, topic:1607"]
the payload $data$ that is being backed up is left unspecified, but it will include (at least) the descriptor or the [BIP388](https://github.com/bitcoin/bips/blob/c5220f8c3b43821efa3841a6c5e90af9ce5519e8/bip-0388.mediawiki) wallet policy
[/quote]

I would suggest a simple JSON blob with a `"descriptor"` field. That can arbitrarily be expanded to include whatever else people want to (occasionally) backup, such as [BIP329](https://github.com/bitcoin/bips/blob/60ac0e8feccb07f891fd984e4ed76105d2898609/bip-0329.mediawiki) labels.

[quote="josh, post:3, topic:1607"]
If the encrypted descriptor is stored publicly or on a compromised server, an attacker who gains access to one secret gains knowledge of the existence of the multisig. This is not ideal if a user wants to protect themselves with a decoy single-sig wallet.
[/quote]

But this is an essential property for recovery in my opinion, see above.

[quote="josh, post:3, topic:1607"]
In a $k$-of-$n$ multisig descriptor, the secret $s$ is split into $n$ shares using shamir secret sharing, where $k$ shares are needed to recover. Each share is then encrypted with one xpub, so that $k$ xpubs are needed to decrypt.
[/quote]

This adds complexity to recovery, since the (miniscript) descriptor might define a completely different policy than the the information access control. You'd also have to remember the threshold value.

In any case this scheme still reveals the _presence_ of a more sophisticated wallet even if not its contents. It makes it unlikely an attacker falls for the decoy.

Stenographic storage of the backup seems like a better way to deal with this issue.

-------------------------

