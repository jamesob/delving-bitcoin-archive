# Avoiding xpub+derivation reuse across wallets, in a UX-friendly manner

kloaec | 2025-04-29 17:54:43 UTC | #1

# Introduction:
Signing devices can be reused for multiple wallets, but xpubs+derivation should not.
Some simple examples are when rotating to a different wallet to change one key in a multisig setup, or when rotating the wallet to push an absolute timelock date. For privacy, it is important to not reuse xpubs+derivation even if the same devices are still present in the setup.

Moreover, it is unreasonable to assume users (will) stick to a single software, they might use a multisig on one, a timelocked wallet on another... 

It is also unreasonable to assume a user remembers or knows every (any?) paths they have used, and to remember to increase their derivation path each time they create a new wallet.

I'm writing this post as we are implementing "multi wallet" in [Liana](https://github.com/wizardsardine/liana) and I am not happy with any commonly used solution to avoid xpub reuse.


# The issue:
One should not reuse xpubs+derivation across different wallets, it is bad for privacy.
A signing device should be easy to reuse for different wallets without requiring the user to switch keys (passphrase or other), as a different derivation path is enough.

An extra layer of complexity is when dealing with air-gapped signers, the user is now expected to understand the difference between BIP numbers and wallet types.

### The current ways this is addressed:

* ask the user to select their "account" number
* or keep a state of the last account used in that specific software wallet (typically on a wallet company server, or sometimes locally)

The first is yet another confusing thing users need to learn and keep track of, the second is specific to one software. Key reuse can still happen if reused on a different software.

I am not aware of other solutions currently used in prod, let me know if there are some.

# Better UX to avoid key reuse - ideas

Since [output descriptors](https://bitcoinops.org/en/topics/output-script-descriptors/) technically supersede "BIPxx" standard derivation paths by explicitly recording the derivation paths used, and because multisig and more advanced/complex wallets shouldn't rely on brute force reproduction of the script anyway, I believe it is time to think of a better UX than "increase your account number" of [BIP48](https://en.bitcoin.it/wiki/BIP_0048#Account).

That being said, output descriptors also apply to single sig wallets... where users have been taught that keeping their Mnemonic is enough. Moving away from this for single sig wallets will not be trivial. This post is mainly intended at multisig or more advanced wallets, but if we can fix single sig key reuse too, that'd be a win.

I would like to define yet another "standard" that would reasonably guarantee that a user will not reuse keys, even on different software wallets, when they reuse a signing device.

### Desired properties:

- **Works across vendors**. Different software following the "standard" will always generate wallets with different xpubs, avoiding key reuse
- **Does not depend on others implementing it**. A software **NOT** following the "standard" will not use the same xpub as a wallet created with a software following the spec (unless malicious)
- **Should work with current wallet flows**. Some multisig software wallets already offer "one click" wallet rotation, without requiring access to the signing device. Others might generate multiple xpubs per seconds. Both of these use-cases should be covered, and more if we find them
- **Should be fool proof**, the point is to improve UX without breaking privacy


### Some proposed ideas

Hardened path, requires the signing device to generate the wallet

1. **Random path**. As "standard" derivation paths are irrelevant to output descriptors, we can simply randomize the paths. [FINGERPRINT/xx'/Random'].
This is the simplest of all solutions and would work pretty great for all cases. For very large signers (belonging to many thousands of wallets, typically cosigners), another derivation layer could be added. [FINGERPRINT/xx'/Random'/Random2']

2. **Deterministic path**. For example using UNIX time at the time the xpub is requested to the device. [FINGERPRINT/xx'/1745925'/432']. In this example I used the UNIX time. I arbitrarily split it at the 1 000sec digit. It would fit in a single level but will hit the 32-bits limit in 80 years.
Another approach would be to use a human readable date, for example [FINGERPRINT/xx'/20250429'/124630'], where the first level is the date, the second is the time in Hours Minutes Seconds. This is quite easy from a UX perspective
3. **Storing state at each xpub request to the signing device**. As the same seed can be imported to multiple hardware devices, this state has to be synchronized, therefore be shared with a server (public or not). The "account" can then be increased by 1 for each request. [FINGERPRINT/48'/0'/x'/2'] or whatever format, where x is increased. This method would need to be multi-vendor compatible

Out of these options, my preferred is 2, followed by 1.

1- Random path:

- Some malware used random derivations in the past, creating stigma
- It works all the time, even if you generate 1000s of xpubs per seconds (co-signers?)
- Doesn't rely on any state
- Hardware don't like "non-standard paths"

2- Deterministic:

- Can be made user-friendly, understanding why the path looks like it does (date/time)
- Works for humans, but might not work for a machine as-is. If more than 1 xpub is needed per second, another level needs to be added (or more precision, fractions of seconds)
- Hardware don't like "non-standard paths"

3- State is synchronized:

- Wallet software needs to be connected to such server, bad for privacy and UX
- Needs to be vendor-agnostic, and persistent
- Might conflict with non-compatible wallet paths, depending on what path is incremented

Other considerations:

For the deterministic wallet, we cannot assume the software will be at the correct time/date. I believe with precision level at the second it should be OK, but it is simple to increase this to a millisec if believed useful.
One UX issue I am aware of is that some hardware wallets will display a warning for "non-standard" derivation paths. This is definitely a drawback of these methods, but a much lesser evil than reusing xpubs.

### Non-hardened paths, to rotate without access to the signing device

One use-case that multiple wallets are offering now (Casa, Nunchuk, Keeper, Unchained...) is to rotate/migrate wallets for key replacement or timelock management. To make the UX "one click", some of these wallets are using a non-hardened path, and incrementing that counter.

While this is acceptable, it is currently not a fit for our cross-vendor compatibility. One should be able to import a wallet created on a software, to another, and then rotate.
The main issue here is that if a user wants to import a backup from one vendor to another, each vendor will potentially know about all other such wallets of this user (unhardened paths).

I suppose for this specific use case, the Random option is the best one but is becoming quite heavy, adding a few extra levels of non-hardened paths, preventing brute forcing.

Or just avoid rotations without the signing device being present: we stick to hardened paths.

### What about single sig!?

That one is a tough one. The problem persists for single sig, as you may reuse a signing device across software/wallets.

Getting rid of the "standard" derivation paths is a battle I know won't be worth it. If people don't want descriptors, then they will have to sacrifice something: privacy if they need a sync server for example.
But in that case (sync server), why not just use said server to [keep an encrypted backup of your descriptor](https://delvingbitcoin.org/t/a-simple-backup-scheme-for-wallet-accounts/1607/1) instead?

### Other discarded ideas (from myself and others) were:

- State stored in the signing device, which wouldn't persist in case the secret needs to be imported again, and which wouldn't be synchronized across devices of the same seed
- Checking on-chain for prior use of a key, which assumes a key has been used/revealed. For a wallet like [Liana](https://wizardsardine.com/liana/) or similar using multiple Taproot branches, using a wallet might not reveal a key was used. Or even just depositing funds to a wallet does not imply spending from it for a while, so an xpub could be already part of a setup without having yet revealed any key on-chain
- Tying the path to a "name" of the wallet, as names may not be unique either

---
---
Thanks for reading, I hope we can find a good way to move forward without the risk of users reusing keys across wallets.
The idea here is to come up with a best practice for wallet devs. I don't pretend to have the perfect answer.

-------------------------

kloaec | 2025-04-29 17:58:30 UTC | #2

Regarding air gapped wallets, we might want to define a "xpub request" QR from the software to the device, instead of the user typing the derivation.

-------------------------

