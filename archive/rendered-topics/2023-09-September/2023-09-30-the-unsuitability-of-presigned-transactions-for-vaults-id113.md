# The unsuitability of presigned transactions for vaults

jamesob | 2023-09-30 11:54:55 UTC | #1

During recent discussions about https://delvingbitcoin.org/t/covenant-tools-softfork/98, a few people have raised the ideas that
1. Vaults can be implemented today using transactions which are presigned with ephemeral keys, and
2. The apparent absence of these vaults indicates a lack of demand for vaults as a custody approach.

I will argue here that (i) large operations are using vaults (or painfully emulating them), and (ii) the limitations of vaults with existing script capability make them basically unusable for most people.

### Quotes

> With vaults in particular, if those were a strong justification we'd expect someone to be simulating them on mainnet with a semi-trusted third party oracle service.
> - [Peter Todd](https://twitter.com/peterktodd/status/1707761756704051532)

> Vaults can be done today with pre-signed transactions, even if the trade-offs are different it’s a practical construction.
> - @ariard (https://github.com/bitcoin/bitcoin/pull/28550#issuecomment-1741554330)

> 
> **Vaults** can be done today with presigned transactions. The presigned versions are a lot harder to implement correctly than with `OP_CTV` + `OP_VAULT`, they can’t receive payments without interaction with the payer, and the vault user needs to store more data. However, it seems to me that if there were high demand for the general ability to have hot spends announced onchain with a cancellation window, a significant number of people would be using presigned vaults—but I’m not aware of that happening.
> - @harding in https://delvingbitcoin.org/t/covenant-tools-softfork/98/11

## No one using vaults? 

This point is quick and easy to address: there are large custodial operations that are both emulating vaults with automated multisig signing _and_ implementing vaults with presigned transactions, however this isn't widely publicized. These are institutions that manage a lot of bitcoin. I have firsthand knowledge of one (presigned), and I'll describe the shortcomings that motivated me to pursue CTV and then ultimately `OP_VAULT`.

It's worth noting that there are open efforts to implement (or emulate) vaults using the existing scripting capabilities we have today, namely [Revault](https://wizardsardine.com/revault/), [Liana](https://wizardsardine.com/liana/), and Bryan Bishop's [prototype code](https://github.com/kanzure/python-vaults).

## Vaults with presigned transactions

As Bryan Bishop first noted a few years ago, and as I described at length in [my vaults paper](http://jameso.be/vaults.pdf), one form of vaults _can_ be implemented using ephemeral keys, however there are major usability issues with this approach.

- **Use of ephemeral keys**: to emulate vaults today (without multisig/autosigning), users must generate temporary private keys that must be destroyed after use to lock funds into a pretermined spend path using "bearer asset" transactions. None but the most sophisticated and well-funded operations can generate ephemeral keys in a way that is convincingly secure, and even then there are questions. Proving non-existence of attack is impossible.
- **Address reuse => burnt coins**: because transaction graphs are pregenerated, if coins are accidentally sent to an "already sealed" presigned txn vault, the coins are burnt. This means end users cannot be given vault addresses publicly, and even sophisticated users must be very careful.
- **Difficult UTXO management**: vault creation is very onerous since it requires generating ephemeral keys securely, and this setup process must be performed every time the user deposits. This makes coin consolidation difficult.
- **Static/difficult fee management**: because transactions are pregenerated, the fee management strategy must be decided up front and locked in. This means that inefficient CPFP must be used with a statically defined fee wallet. If that fee wallet is lost, the vault's operation is hampered.
- **"Toxic" vault data must be persisted indefinitely**: the nature of using presigned transaction means that vaulted funds become "bearer assets" within these transactions, and they must be persisted indefinitely by the user.
- **Inefficient chain use**: since no batched withdrawals or recoveries are possible, each deposit must have its own on-chain lifecycle, resulting in much more chain use (and fee spend) than is strictly necessary.

## Presigned transaction vaults are functionally hopeless

Needless to say, no one aside from the most sophisticated industrial operation is going to make use of this strategy. To claim that no one implementing a scheme like this is evidence that no one wants or would benefit from vaults is a fraught argument.

## What about just using CTV?

Some time ago I [implemented a prototype](https://github.com/jamesob/simple-ctv-vault) of vaults assuming the availability of CTV. I was hoping to make usable vaults with just one new primitive. I discovered that this approach removes the need for ephemeral key generation and backup of critical data, but all of the problems associated with the nature of pregenerated transactions graphs persist.

This exercise led me to write [the vaults paper](https://jameso.be/vaults.pdf), which details the ways that these usability issues are solved by `OP_VAULT`.

### Summary

In summary, presigned transactions are not really a suitable substrate for vaults unless you have a large, well-funded engineering team. I fear the clear infeasibility of this particular design has predisposed some contributors to assuming vault operation is more onerous than it needs to be. It should come as no surprise that this is not a widely deployed design, and shouldn't be used as an indictment against other vault strategies.

-------------------------

harding | 2023-09-30 17:39:14 UTC | #2

As noted in the quote from me from the other thread, I think @jamesob's vault design is superior from a user's perspective than presigned vaults.  However, it seems to me that users ought to be able to get most of the key benefits using presigned transactions.  As such, I'll play devil's advocate here.

[quote="jamesob, post:1, topic:113"]
there are large custodial operations that are both emulating vaults with automated multisig signing *and* implementing vaults with presigned transactions, however this isn’t widely publicized. These are institutions that manage a lot of bitcoin. I have firsthand knowledge of one (presigned)
[/quote]

This is very useful for me to hear and is the type of information that would help change my mind on the desirability of adding vault-specific (or vault-motivated) opcodes.

[quote="jamesob, post:1, topic:113"]
**Use of ephemeral keys**: to emulate vaults today (without multisig/autosigning), users must generate temporary private keys that must be destroyed after use to lock funds into a pretermined spend path using “bearer asset” transactions. None but the most sophisticated and well-funded operations can generate ephemeral keys in a way that is convincingly secure, and even then there are questions. Proving non-existence of attack is impossible.
[/quote]

What is the fundamental difference between an ephemeral key and an ephemeral nonce (the private form of a signature nonce)?  If Alice accidentally leaks the private form of a signature nonce for one of her signed transactions, any funds she received to the corresponding key or a related key using public BIP32 derivation can be stolen (as long as those funds haven't already been spent in a confirmed transaction).

It feels to me like ephemeral keys and ephemeral nonces are really closely related, so any argument that states ephemeral keys aren't secure enough is an argument that neither ECDSA nor schnorr is secure enough, especially in the presence of address reuse.

[quote="jamesob, post:1, topic:113"]
**Address reuse => burnt coins**: because transaction graphs are pregenerated, if coins are accidentally sent to an “already sealed” presigned txn vault, the coins are burnt. This means end users cannot be given vault addresses publicly, and even sophisticated users must be very careful.
[/quote]

I mention this is a clear advantage of consensus-enabled vaults, but now I'm not so sure.  IIRC, in the OP_VAULT design, the most-secure-key can spend at any time without any time constraint, so why can't all of the paths in a presigned transaction vault have a tapleaf that allows spending by the most-secure-key?  That way, if something goes wrong, the most-secure-key can be used to recover and coins never become permanently burnt.

That said, I agree it's still the case that address reuse breaks the vault workflow, so external users can't be given vault address publicly, requiring an extra hop for any deposits.  However, address reuse is also something that's not generally guaranteed to work---people lose access to old keys from time to time---so the fact that it always doesn't work for presigned transaction vaults just means we need to work harder on deploying system-wide solutions, such as wallet-enforced address expiry times.

[quote="jamesob, post:1, topic:113"]
**Difficult UTXO management**: vault creation is very onerous since it requires generating ephemeral keys securely, and this setup process must be performed every time the user deposits. This makes coin consolidation difficult.
[/quote]

I maintain my previous argument about secure ephemeral key generation being something we have extensive near-experience with.  I don't see a fundamental difference between the difficulty of securely handling ephemeral nonces each time a user spends to securely handling ephemeral keys each time a user needs to receive.  I think this is basically a repeat of your first point.

[quote="jamesob, post:1, topic:113"]
**Static/difficult fee management**: because transactions are pregenerated, the fee management strategy must be decided up front and locked in. This means that inefficient CPFP must be used with a statically defined fee wallet. If that fee wallet is lost, the vault’s operation is hampered.
[/quote]

I'm not sure CPFP needs to be used (can sighash flags be used?), but even if it does, presigned vault transactions may be able to have an efficiency advantage over OP_VAULT transactions by the presigned versions being able to use all keypath spends (with BIP68 sequence bits set), whereas OP_VAULT must use scriptpath spends (unless the most-secure-key is used).  I think the vbytes difference probably favors the OP_VAULT version there (unless a deep taproot path is used), but I think it makes the comparison less clear cut.

Additionally, I think ephemeral anchors, if deployed, will eliminate the need to use a statically-defined wallet.  Instead, any onchain funds will be able to fee bump the vault transaction without introducing pinning risks.

[quote="jamesob, post:1, topic:113"]
**“Toxic” vault data must be persisted indefinitely**: the nature of using presigned transaction means that vaulted funds become “bearer assets” within these transactions, and they must be persisted indefinitely by the user.
[/quote]

I think this is largely the same problem faced by tens of thousands of LN nodes at present---they must store revoked states robustly but also securely, as any loss and release of that data can cost them money.  This is certainly less than ideal, but it hasn't stopped widespread use of LN.

[quote="jamesob, post:1, topic:113"]
**Inefficient chain use**: since no batched withdrawals or recoveries are possible, each deposit must have its own on-chain lifecycle, resulting in much more chain use (and fee spend) than is strictly necessary.
[/quote]

I'm agreed here, but I think this actually works towards my overall point: if vaults were a highly desired feature, people would be willing to pay more for them and then we could figure out how to optimize them for everyone's benefit.  But if very few people are willing to pay even a marginal increase in fees for vaults, I think that's an argument that optimizing them is premature.

-------------------------

harding | 2023-09-30 17:53:01 UTC | #3

I wanted to add this point from @ajtowns in the other thread, which is an argument I do find compelling about the advantage of OP_VAULT over presigned transaction vaults.

[quote="ajtowns, post:15, topic:98"]
I don’t think pre-signed vaults really get the right security properties? As described in [2019](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2019-August/017231.html):

> One of the biggest problems with the vault scheme […] is an attacker that
> silently steals the hot wallet private key and waits for the vault’s
> owner to make a delayed-spend transaction to initiate a withdrawal
> from the vault. If the user was unaware of the theft of the key, then
> the attacker could steal the funds after the delay period.
[/quote]

-------------------------

