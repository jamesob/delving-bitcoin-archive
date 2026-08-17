# Towards New Self-Custody Best Practices

seed-cat | 2026-08-04 20:23:14 UTC | #1

# Self-Custody Best Practices
By now, many Bitcoiners are aware of the risk of trusting a single wallet vendor with their life savings.  The low-entropy vulnerability in the Coldcard has resulted in over [$100M in losses](https://coldcard-hack-tracker.vercel.app/), with many users only narrowly escaping losses because they added sufficient dice roll or passphrase entropy.

This is not the fault of users, but a failure of the Bitcoin technical community to create self-custody best practices that balance complexity and security.  I have been working on seed recovery tools for a while and have learned many of the ways that users fail when attempting to self-custody. Here is my attempt to draft some best practices:

# 2-of-2 Multisig

Multisig has often been discouraged for new users since the typical suggestion involves setting up 3 hardware wallets in a 2-of-3 multisig, importing the public keys into coordinating software, then importing the public keys into the hardware wallets themselves, plus backing up all the seed words separately. In addition, if users fail to properly backup all 3 public keys they could lose funds. However, keeping all 3 public keys together could also reveal the user’s entire wallet history if found.



To avoid the unnecessary complexity and risks of a 2-of-3 multisig, we should standardize on 2-of-2 multisig as the best practice (when not relying on third-party signers such as Unchained and Casa). The level of security and redundancy is the same as 2-of-3 when using distributed seed word backups (we can tolerate up to one comprised backup or device).



# Multi-vendor

It appears some multisig users relied on Coldcards for 2 of the signers, which negated the security advantages of multisig. Had 2 different vendors been chosen in a 2-of-2 multisig, these funds would still be secure.



Although hardware wallets (Trezor, BitBox, Foundation, Coldcard) store secrets differently in their secure elements, they all rely on the same [secp256k1](https://github.com/bitcoin-core/secp256k1) cryptographic library to perform critical operations such as signing. Although unlikely, if a vulnerability appeared in that library it is possible that multiple vendors could be comprised simultaneously. For these reasons, we should extend multi-vendor to include no shared cryptographic code.



While this does limit choices for our second-signer, there are at least two options.

* Sparrow Wallet, being written entirely in Java performs cryptography using the [BouncyCastle](https://www.bouncycastle.org/) libraries, making it a good multi-vendor option so long as users choose a strong password to encrypt their secret key on disk.

* Ledger Wallet is the only wallet that performs cryptography within the Secure Element via its proprietary [BOLOS](https://www.ledger.com/academy/security/the-secure-element-whistanding-security-attacks) operating system, making it arguably more secure albeit closed-source. 



# User-Generated Verifiable Entropy

Often it is said that users are bad at generating entropy compared to dedicated security chips. However, this is only the case when users are asked to come up with passwords they can remember. And as demonstrated by the Coldcard bug, when users do not create their own entropy they cannot verify the entropy of their wallet is truly random.



For this reason, we suggest users choose 11 of the 12 seed words. The best way to do this is to mix up a bag containing all 2048 BIP39 words, either cutout from paper or a product like [seed sticks](https://seedsticks.org/). However, there are many creative alternatives such as using a [shuffled deck of cards](https://github.com/jimbojw/seed-picker-solitaire). The wallet should provide choices for the 12th word to ensure the checksum is valid. Advanced users can verify the software derives public keys from seed words correctly [using third-party tools](https://iancoleman.io/bip39/) to keep the vendors honest.



We see no advantage in using 24 words since it adds no additional output entropy and takes twice as long to backup. 100 dice rolls is more difficult to generate and verify. We should not add complexity if we do not get more entropy.



For similar reasons, users should not use a passphrase. Passphrases provide no additional output entropy and often provide little input entropy. Furthermore, passphrases follow no standards for backup and are often forgotten, making it more likely a user will lose access to their funds.



# Backup Practices

As demonstrated by [Jameson Lopp’s tests](https://blog.lopp.net/a-treatise-on-bitcoin-seed-backup-device-design/), the best backups are steel / titanium plates that are punched or stamped. Punching is slightly preferable since it is more forgiving and easy to “erase” by punching all boxes.



After creating backups, users should wipe and restore the wallet using only their seed backups. This critical step ensures the backups are actually correct in case of device failure.



Backups and devices should be stored in different locations that are difficult for attackers to access. If a single location is compromised, the user should lose no funds. The locations chosen in practice will depend on the threat model of the individual user.



# Transaction Practices

After setting up a wallet, users should perform a small receive-and-send test before sending a large amount of bitcoin to the wallet. Users should always confirm the receive address, amounts, and change address using the display on their signing devices.



Whether the transport is USB, QR code, or MicroSD is mostly irrelevant so long as at least one of the devices is not comprised. General-purpose devices that act as signers such as Sparrow on desktop or Blue Wallet on mobile should be used exclusively for Bitcoin to avoid running malicious code that might steal secrets from the signing program’s memory.



# TLDR

1. Setup a 2-of-2 multsig between one of (Trezor, BitBox, Foundation) and (Sparrow, Ledger).

2. Select 11 words from a [mixed bag](https://seedsticks.org) of the 2048 BIP39 words to generate seed words for both wallets. 

3. Optionally, verify the wallet generates your keys correctly using a [third-party tool](https://iancoleman.io/bip39) (and then select another set of random words). 

4. Backup your seeds by punching them into a [steel plate](https://bitbox.swiss/steelwallet/). 

5. Wipe your devices and restore them from backups

6. Send a small test transaction, remembering to always confirm the receive address, amounts, and change address using the displays of both signing devices



If you need hand-holding, use Casa or Unchained.

**NOTE: I have no affiliation with any of the products / companies mentioned in this post**

-------------------------

optout | 2026-08-07 15:16:15 UTC | #2

I generally welcome such suggestions, though I find picking 2-of-2 multisig a bit too restrictive.
I would add a comment/suggestion regarding manual seed generation.

Fully offline and manual seed generation is plausible, and several documented methods exist -- using 12 physical coins, one or more dice, a deck of cards, or pieces of papers with seedwords (note: I would not single out one method here). Manual generation does not rely on any device, does not need to trust a supplier for software & hardware stack.

One tricky part is the checksum computation/corrections. Devices like a Seedsigner can be used to compute the correct last work (checksum). A potential beneficial effect on the industry would be if any (most) hardware devices would offer the checksum correction functionality. Receiving (address generation) and sending (signing) needs a device in any case, so it would be practical if the same device could perform the checksum correction as well.

The way I envision it:

Whenever the user is able to enter a BIP39 seedphrase into the device, the device SHALL verify the checksum of the seedphrase. In case the checksum is invalid, the device SHALL inform the user, and offer the option to discard the seedphrase, or to "Correct" it. If the user chooses "Correct", the device SHALL display a warning about the dangers of incorrect manual generation, which the user has to acknoledge. Afterwards, the device SHALL compute the correct checksum, and display the corrected last word (or full seedphrase). Next, the device SHOULD offer the options to accept or discard the corrected seedphrase.

-------------------------

seed-cat | 2026-08-07 17:21:35 UTC | #3

Agreed, user provided verifiable entropy eliminates the need to trust the device.  Although using 2 devices greatly reduces the odds both are simultaneously broken.

I think it is fine for advanced users to choose 2-of-3 or 3-of-5, etc.  These best practices are not meant to be a universal standard, but more the best way to prevent non-technical / new users from losing funds.

For instance, passphrase loss is the most common foot-gun for new users, but passphrases may make sense if an advanced user wants to separate accounts with the same seed words without changing derivation paths.

-------------------------

Anzus_GemWallet | 2026-08-14 03:14:47 UTC | #4

One addition that may help nontechnical users is a tiered model rather than one setup being presented as the default for everyone.

A small everyday balance, meaningful personal savings, and life savings may justify different levels of complexity. A 2-of-2 multi-vendor setup can reduce some risks, but it also introduces more backup material and more ways for an inexperienced user to make a recovery mistake.

Would it make sense for the guidance to define security levels and include a recovery rehearsal before substantial funds are deposited? Many users create a backup but never confirm that they can restore it or that they retained all the information required by the setup.

-------------------------

seed-cat | 2026-08-17 17:38:54 UTC | #5

Yes, I think software wallets should do a lot more hand-holding.  Regarding the multisig, wallets could encode the master xpubs on-chain (in a private manner) so you don't have to worry about backing them up in a 2-of-3 or 3-of-5.

-------------------------

