# MultisigBackup.com: Backup and recover a k-of-n descriptor using only n seeds

josh | 2025-02-06 19:56:35 UTC | #1

Hi all, I've been working on a new open source project, [multisigbackup.com](http://multisigbackup.com/), which is designed to make it as easy as possible to backup and recover a multisig descriptor, using nothing more than a threshold number of seeds.

**Expectations vs. reality**

Many new users aren't aware that in order to recover a 2-of-3 multisig wallet, you need at least 2 seed phrases *and* the output descriptor. Among other things, the descriptor contains the derivation paths and the extended public keys (xpubs), all of which are needed in order to spend from a standard 2-of-3 multisig.

This runs counter to most users' expectations, because 2 seed phrases are NOT enough to recover a 2-of-3 multisig wallet. Without the descriptor, a user's funds are lost if they lose one of their seeds. This scares users off from creating a multisig and pushes others to use a collaborative custody provider like Unchained, Casa, Nunchuk, etc., who walk them through the process and keep a copy of the descriptor on their behalf.

**How are descriptors typically stored?**

If you're using a collaborative multisig provider like Unchained, Casa, Nunchuk, etc., the descriptor is backed up on your behalf. This makes for a smoother user experience during onboarding and easier inheritance, but you sacrifice privacy, must pay $200+ annual fees, and must trust the provider to maintain backups. For this reason, it's recommended that users still back up the descriptor themselves.

A common, but dangerous, approach to back up the descriptor is to store it on a laptop or in the cloud. While essentially free, this approach makes inheritance extremely challenging and risks the descriptor getting lost or accidentally deleted.

A better practice is to print out multiple copies of the descriptor or put them on USB sticks, storing one alongside each seed phrase. This is more robust and better for inheritance, but the data can still get destroyed in a fire or simply degrade.

Lastly, you can purchase special equipment like a SeedHammer, which can engrave your descriptor onto steel plates. This is the most durable existing solution, but it's pricey.

**The goal: permanent, privacy-preserving, one-click backup and recovery**

For the best user experience, a new user should only need to physically back up their seed phrases. Anything else is an extra burden that discourages new users and creates risk.

With [multisigbackup.com](http://multisigbackup.com/), descriptor backup is made easy. The user simply inputs their descriptor, and the tool encrypts it and generates a taproot address, which with a single payment, will inscribe the encrypted data forever on Bitcoin.

Later, when a user wishes to recover it, they simply connect two hardware devices (Ledger and Trezor supported) and press recover. To recover manually, the user inputs the master fingerprints of two seed phrases, which are hashed and used to find the encrypted descriptor onchain via an open source indexer. The derivation paths, which are not encrypted, are then used by the user to derive two xpubs, which can decrypt and recover the original descriptor.

**How it works**

This tool encrypts the sensitive data (master fingerprints and xpubs) in a **k-of-n** descriptor in a way that can be decrypted with *any* **k** xpubs. Here's a high-level overview of how this works:

1. Extract xpubs and master fingerprints
2. Encrypt xpubs and master fingerprints using a random seed
3. Use shamir secret sharing to split the seed into **n** shares, where **k** shares is enough to recover it
4. Encrypt each share with a corresponding xpub
5. Append the encrypted data to the stripped descriptor

To facilitate recovery while preserving privacy, each pair of master fingerprints is hashed, and the first four bytes are appended to the inscribed text. These hashes are used to find the encrypted descriptor later using the indexer.

If you're curious what an encrypted descriptor looks like, here's an inscribed example: https://mempool.space/tx/c33203c3c589affbbca8635abf90a0faf8676db8e4a5b52395b4c0f7fee4deed. The plaintext descriptor is:

```
wsh(sortedmulti(2,[3abf21c8/48h/0h/0h/2h]xpub6DYotmPf2kXFYhJMFDpfydjiXG1RzmH1V7Fnn2Z38DgN2oSYruczMyTFZZPz6yXq47Re8anhXWGj4yMzPTA3bjPDdpA96TLUbMehrH3sBna/<0;1>/*,[a1a4bd46/48h/0h/0h/2h]xpub6DvXYo8BwnRACos42ME7tNL48JQhLMQ33ENfniLM9KZmeZGbBhyh1Jkfo3hUKmmjW92o3r7BprTPPdrTr4QLQR7aRnSBfz1UFMceW5ibhTc/<0;1>/*,[ed91913d/48h/0h/0h/2h]xpub6EQUho4Z4pwh2UQGdPjoPrbtjd6qqseKZCEBLcZbJ7y6c9XBWHRkhERiADJfwRcUs14nQsxF3hvx7aFkbk3tfp4dnKfkcns217kBTVVN5gY/<0;1>/*))#hpcyqx44
```

**Summary**

* *Manageable one-time cost* - cost to inscribe is equivalent to making ~4 multisig transactions (~400vb or 800 sats at 2 sats/vb)
* *Strong data availability guarantees* - no risk of loss due to fire, hardware failure, vendor error, or vendor closure
* *Privacy preserving* - no information leaked publicly nor to any vendor, not even a thief who has stolen a seed phrase would know it’s part of a multisig
* *Simple recovery* - anyone can recover simply by connecting two devices = simplified inheritance
* *Enables simpler setup* - a wallet can present a payment link or inscribe on the user’s behalf during setup = superior UX

**Areas for future development**

* Taproot multisig support
* Miniscript support
* Support for non-standard descriptor formats (Caravan, BSMS, etc.)
* Further inscription size optimizations
* Software wallet integrations

Happy to answer questions and appreciate any feedback!

Source code: [https://github.com/joshdoman/multisig-backup](https://github.com/joshdoman/multisig-backup)

-------------------------

jsarenik | 2025-02-07 08:26:07 UTC | #2

Thank you!

I love this and I use 2-of-2 multisig (only) with a real human counterpart since 2019. Reason is simple: after noticing the logarithmic chart on Marc Bevand's site I saw the most important thing is to secure the sats from myself. And it allows me to backup while alive and the backup can not spend anything alone. My counterpart also has a backup which I won't cooperate with on spending anything while the counterpart is alive and well. And I pray for that every day when I rehearse my mnemonic (and think of the counterpart).

With some students I have already dissolved our 2-of-2 in peace and voluntary cooperation. I still rehearse those (unique) mnemonics though 

Yes. Bitcoin does not ask for any monthly fees. The network and computers would run where I live anyway.

And such multisig gives plausible deniability so neither me nor my counterparts have any sats. But I also know for sure that noone else has them. Less money for wars and centrally managed social programs.

-------------------------

sipa | 2025-02-07 14:32:39 UTC | #3

I don't feel like using a globally replicated database for information that just a single person cares about is a good use of the technology. It may appear convenient at times when demand for block space is low, but I would caution against building an expectation that this is a realistic option in the long run.

You need backups for other data anyway, and it ought to be trivial to hide this amount of data in your backups in a plausibly deniable way too.

-------------------------

