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

