# Post Quantum Recovery Framework

adiabat | 2026-11-20 19:16:00 UTC | #1

Hopefully, a Cryptographically Relevant Quantum Computer (CRQC) will never be built.  If one does get built, hopefully by that point Bitcoin will have activated PQ output types and everyone will have migrated all their coins to those types.  If people haven't moved their coins though, they would benefit from a Post-Quantum Recovery Protocol.

Hopefully, maybe even likely, PQ recovery won't be needed.  But this is Bitcoin, where nodes store 100GB of rev00000.dat files just in case we have to reorg back to 2009.  Expending considerable effort on mitigating unlikely threats is nothing new.

At recent Bitcoin discussions there was a good amount of interest and different perspectives on this topic and I'd like to record some of the ideas here for reference, feedback & discussion.

(I've made up a bunch of terms here and if people can suggest better names or definitions for things please do!)

## What is PQ Recovery? Prepared and Unprepared

I'd define a PQ recovery scheme as any consensus change that adds additional spending conditions to already existing UTXOs with the goal of protecting those UTXOs against theft from an break in elliptic curve security.

We can also distinguish between **Prepared** and **Unprepared** PQ Recovery, where Prepared Recovery schemes require the user to perform some action or commit some data before Q-day (the day that EC security is broken).  Unprepared PQ Recovery schemes are those which assume the users has taken no actions before Q-day.  I'm mostly interested on Unprepared schemes here.  Prepared schemes overlap with the design of new output types, and probably should be discussed alongside those.  Unprepared recovery seems independent of the PQ output type chosen and is what I'm focused on here.

## What and How

It's helpful to look at two things separately: What is used to perform the recovery, and how the recovery is performed.  The "what" is the recovery secret: the information held by the user, such as a public key hidden behind a hash.

The "how" is what technique is used to leverage that secret into a transaction that moves the coins to a new output type.  People have proposed different methods such as are commit / reveal protocols such as dropkick and lifeboat, as well as ZKP protocols that do not reveal the recovery secret.

In many cases the "what" and "how" are independent and can be mixed and matched.  There is some interaction between them though; for example if a PKH pubkey is used as the recovery secret, the hiding property of the ZKP method is no longer relevant.  (Unless the update is done as a hard-fork, and *only* the ZKP is used to recover, without a normal transaction.  I mostly think about PQ recovery as a soft fork, but there could be a case for doing it as a hard fork.)

## What: Recovery Secrets

Recovery secrets can be separated into 2 categories: **Accidental Secrets** and **Essential Secrets**.  Accidental secrets are data that, while EC is secure, can be made public without loss of control of the coins, but which are often not made public.  Essential secrets are those which, on their own, currently give the bearer of that information control and ownership of the coins.

The primary examples of an accidental secret are pubkeys and scripts hidden in hashed output types like P2PKH, P2WPKH, P2SH and P2WSH.  Some other accidental secrets could be BIP32 xpubs, as well as internal keys or scripts of P2TR outputs.

While the secret most often used today is the EC private key, those would be susceptible to a CRQC.  The most common essential secret for PQ recovery is the BIP32 seed (the "S" value defined in BIP32) and derivation path from there to a private key.  Other essential secrets would be a brainwallet passphrases or other preimages of private scalars.

BIP39 seed phrases could also be used as essential secrets, but since they always (?) use the output of the BIP39 hashing to generate a BIP32 master key we're already covered.  (If anyone knows of wallets that use BIP39 seed phrases but then do not use BIP32 derivation please say so!)

Only going one "step" up the BIP32 derivation tree would count as essential if hardened.  Unhardened parents would be instead be accidental because users may have shared Xpubs, and they do not currently give control of coins.

## How: Commit / Reveal and ZKP

This could be a whole different thread, and I'll just link to some things for now.  I haven't posted or talked much about shrinc-ray, but people might be interested in it as it's a new way to do commit / reveal that solves some of the problems of previous designs.

[Dropkick](https://conduition.io/bitcoin/dropkick/)

[Lifeboat](https://github.com/adiabat/lifeboat/blob/0ac1d0e1999530cbbf291a6513ab16144624432e/lifeboat.md)

[Shrinc-ray](https://github.com/adiabat/lifeboat/blob/0ac1d0e1999530cbbf291a6513ab16144624432e/shrincray.md)

[bip32-pq-zkp](https://github.com/roasbeef/bip32-pq-zkp)

[Post Quantum Recovery Of Hashed Addresses With No Confiscatory Risk](https://delvingbitcoin.org/t/post-quantum-recovery-of-hashed-addresses-with-no-confiscatory-risk/2714)

## Theft / Loss Dillema

There are of course some big problems with PQ recovery.  One is all the early coins that are in P2PK outputs.  Since BIP32 was written in 2012, requiring a BIP32 derivation for any UTXO from 2009-2011 is certain to render the UTXO unspendable.

But even restricting PQ recovery to more recent coins, and those using P2WPKH, there are problems.  While BIP32 is widely deployed, it's not a requirement.  And there's no way to tell if an output was made with BIP32, or any other provable derivation.  Because of this, adding a requirement for a BIP32 derivation path, or any other essential secret, would render some coins unspendable.

On the other hand, every user must know the pubkey of what they're spending.  Even if they don't have the private key, and only have a pre-signed transaction that they've been holding on to for 10 years, that transaction contains the pubkey.  So relying on pubkeys or scripts hidden behind PKH / SH outputs cannot cause coins to become unspendable.  It can, however, leave those coins vulnerable to theft.  Many pubkeys and scripts have been shared without being recorded in the blockchain: for example, a client sending a server an xpub in order to learn about new incoming transactions.  These are called "public keys", after all, and it certainly seems unfair to tell people that their coins are only safe as long as they kept their public keys private!

We can know for certain when UTXOs don't have accidental secrets, because the address has been re-used and the public key is visible in the blockchain.  For these outputs, it doesn't make sense to require accidental secrets for recovery since we know anyone can provide them.  Multisig outputs present complications to this which I'll describe later.

Before we even get to the dilemma, there are the unrecoverable coins, where none of our recovery techniques work.  Say that's out of scope for recovery and move on.

Then with the larger set of recoverable outputs, the dilemma is whether it's preferable to protect all coins from theft at the cost of destroying some of them, or to protect all coins from destruction at the cost of allowing some of them to be stolen.

Trying to get better estimates of what portion of coins / users would be in which category: \[no essential secret / destroyed\] vs \[no accidental secret / stolen\] could help to evaluate the tradeoffs, but it's going to be an estimate, and unfortunately even with good data on the proportions, this is not really an engineering problem, but more of a philosophical problem.  Despite the recent advances in AI assisted mathematical proofs, I don't think there's going to be any model that can show an optimal ratio would be 1.4121 stolen coins for every destroyed coin.

## Mitigating the Dillema: Recovery Eras

So a hard problem Bitcoin PQ recovery faces is this: If we use essential recovery secrets, some coins will be destroyed.  If we use accidental recovery secrets, some coins will be stolen.  We can't just say "use whichever works".  If it's a logical OR, then it's open to theft and equivalent to accidental only, if it's a logical AND, then it's open to destruction and equivalent to essential only.

What might help though, is to separate the two recovery secret types into **eras** (or epochs or some other name, but I think the term epoch is used for halvings?).  It would have to be essential first, then accidental.  So we could say, from Q-day to Q-day + 2 years, we require essential secret proofs.  Then from Q-day + 2 years on, we require accidental secrets.

This would allow users who have essential secrets, but have leaked their accidental secrets, a 2 year window to safely move their coins, after which their coins could be stolen.  For users with only accidental secrets, their coins would be inaccessible for 2 years, after which they would be able to recover them.

The alternatively, the eras don't have to be fixed, and spending conditions could be a function of UTXO age: eg the essential secret requirement lasting from Q-day until the UTXO is twice as old as it was at Q-day.  The idea being that UTXOs which haven't moved for a long time would be both less inconvenienced by being frozen for a longer time, and also more likely to be owned by a user who is not paying attention to Bitcoin, and has a higher risk of missing a shorter recovery window.

## Multisig

Multisig outputs present further complications.  Multisig, where M of N keys need to sign, is the most widely used P2SH / P2WSH script.  (P2TR mutisigs like FROST / Musig2 are more complex and I'll try to post about it later.) These present another fun question: Is theft from a multisig co-signer less bad than theft from an unrelated 3rd party?

Unlike P2(W)PKH outputs, where accidental secrets could allow secure recovery, we know that with multisig the accidental secret must have been shared, and it cannot be used to provide the original security the script intended.

Assuming all multisig signers have CRQCs, requiring accidental secrets for a 2 of 3 P2WSH script effectively turns it into 1 of 3 since every signer knows the script.  Requiring essential secrets for the 2 signing keys increases destruction risk of destroying the coins.  It seems reasonable to say that there's already some level of trust between multi-sig co-signers, and defending co-signers from each other is less important than defending against theft from unrelated parties.  That would suggest that using accidental recovery secrets for multisig is OK.  But here again it's more of a philosophical problem, and one could argue for the indefinite requirement of essential secrets for multisig scripts, to prevent theft by co-signers.

## TLDR

PQ recovery is possible for most coins, but these protocols may not just be **quantum-resistant** but also **consensus-resistant**.  Hopefully there are techniques like the ones mentioned here that can help minimize the disagreements, and help to break consensus-resistance long before any breaks in any preimage-resistance or collision-resistance, and give people confidence in Bitcoin's resilience against a CRQC.

And then hopefully we never use any of this :slight_smile:

-------------------------

