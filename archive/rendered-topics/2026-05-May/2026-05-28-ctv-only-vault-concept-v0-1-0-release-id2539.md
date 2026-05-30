# CTV-only Vault Concept v0.1.0 release

ademan | 2026-05-28 20:47:57 UTC | #1

# Announcing MCCV v0.1.0

I am happy to announce the first release of MCCV, my [More Complicated CTV Vault](https://github.com/LNHANCE-Expedition/mccv) concept, built using BIP-119 `OP_CHECKTEMPLATEVERIFY`.
MCCV demonstrates that a surprisingly capable vault can be constructed using only `OP_CHECKTEMPLATEVERIFY`, without requiring more expressive covenant opcodes.
While the resulting design has significant computational and usability tradeoffs, it manages to provide deposits, delayed withdrawals, recovery, velocity control, and repeated vault operations.

At a high level, funds are deposited from a normal wallet into the vault in predefined increments.
Withdrawals are made in predefined increments, and are subject to a relative timelock scaled according to the amount withdrawn.
The relative timelock provides a challenge period during which the pending withdrawal and remaining vault balance can be swept to a recovery key that has been kept more secure.

Because the timelock scales linearly with the amount withdrawn, the vault provides consensus-enforced limits on withdrawal rate, reducing potential losses.

# Overview

NOTE: This is experimental proof-of-concept software and should be used only on signet (or regtest).

[The current implementation](https://github.com/LNHANCE-Expedition/mccv) is a working CLI proof of concept written in Rust using [BDK](https://github.com/bitcoindevkit) and [`rust-bitcoin`](https://github.com/rust-bitcoin/rust-bitcoin).
It runs on regtest and signet using Bitcoin Inquisition v29, which provides support for [`OP_CHECKTEMPLATEVERIFY`](https://github.com/bitcoin/bips/blob/7f9434c9c81bb49825200a5be5ddb1ae53fd6dcc/bip-0119.mediawiki), TRUC, and ephemeral anchors.

The implementation supports:

* Generating vaults
* Receiving and spending with a hot wallet
* Depositing fixed-size increments into the vault
* Initiating delayed withdrawals in fixed-size increments
* Sweeping mature withdrawals back to the hot wallet
* Recovering the entire vault to a cold recovery key

* Github: https://github.com/LNHANCE-Expedition/mccv
* User Guide: https://github.com/LNHANCE-Expedition/mccv/blob/17ed3e3220b673f1e57746d0b4609448e8064609/docs/user-guide.md
* Protocol Specification (WIP): https://github.com/LNHANCE-Expedition/mccv/blob/17ed3e3220b673f1e57746d0b4609448e8064609/docs/protocol.md

# Features

## Protection against unauthorized withdrawal

As with all vault constructions, withdrawals from this vault enter a challenge period during which the user can sweep the withdrawal and remaining vault balance to a recovery key.

## Velocity Control

This vault not only provides a way to claw back funds in case of unauthorized access, but it adds consensus-enforced restrictions on the rate at which withdrawals can be performed.

## Multi-vault-operations

This protocol supports a substantial but bounded number of deposits and withdrawals on the same vault.
This gives the vault significant additional flexibility over one-shot vault designs.

## Backups

This type of vault can be backed up in a very small, constant amount of data.

# Significant drawbacks

## Complicated

This proposal generates potentially millions of transactions; it needs a strong code quality pass and auditing to achieve baseline trustworthiness.
Moreover, the complexity is itself a risk, which makes this protocol less attractive for exactly the sort of large balances it is meant to protect.
As long as the cold/recovery key escape hatch exists, this does somewhat mitigate the most concerning risks.

## Computationally Expensive

The computational expense grows with how fine grained the deposit increments are, how many increments can be withdrawn or deposited at one time, and how many total vault operations the vault supports.
All of the values that improve the user experience increase the computation cost as well as the on-chain cost.

## Trusting Vault Generation

The computational complexity of this scheme causes a problem with establishing trust.
The device that generates the vault needs to be sufficiently powerful to enumerate the vault the user desires, while also being trusted.
Generating vaults with practical parameters currently requires hardware on par with modern commodity PCs or mobile devices, but such systems are difficult to trust.
An offline PC is the best fit for these requirements, but an offline PC also satisfies user security requirements fairly well without any complicated vault.
STARKs could potentially enable practical verification of vaults on hardware wallets, which I believe is a compelling direction for future work.

## Two step withdrawal

Because every vault state is precomputed, funds cannot be withdrawn directly to the target payee.
Instead, mature withdrawals are spendable by hot keys, which can send the funds to the payee.
This means that if a hot wallet is compromised, funds can be stolen after the withdrawal matures.
This also means that authenticating a withdrawal is complicated, because the ultimate destination of the funds is not part of the withdrawal transaction.
Both `OP_VAULT` and `OP_CCV` solve this issue (and in fact obsolete this entire design).

## Privacy

Tight velocity control requires funds to be consolidated into a single UTXO.
This presents a major tradeoff with on-chain privacy.

## Fees

This is not inherent in the protocol, but the way this proof of concept is designed, the hot keys and hot wallet keys are related.
Because of this, an attacker could first drain your hot wallet, preventing you from paying fees to confirm a clawback/recovery transaction.

This issue can be mitigated by ensuring watchtowers have separate keys with separate funds to pay for recovery transaction fees.
This applies whether the watchtower is run by the user or a third party.

# Protocol Overview

I am currently working on a [detailed protocol specification](https://github.com/LNHANCE-Expedition/mccv/blob/17ed3e3220b673f1e57746d0b4609448e8064609/docs/protocol.md).

A vault is a massive DAG of precomputed transactions and transitions between them enforced by `OP_CHECKTEMPLATEVERIFY`.
Vaults are configured with the following parameters:

* `scale` - the base deposit and withdrawal increment. Every operation on the vault is in multiples of this number of satoshis.
* `delay` - blocks of delay per increment withdrawn
* `max-withdrawal` - maximum amount that may be withdrawn at once
* `max-deposit` - maximum amount that may be deposited at once
* `max-depth` - number of operations the vault can support (the depth of the DAG)

These parameters control the tradeoffs between security, usability, and precomputation cost.

# History

This vault project began as an experiment to determine how feasible it is to build a user-friendly vault using only CTV.
It was generally accepted that CTV alone does not enable constructing the ideal reactive vault, but CTV does enable building precomputed state machines.
These constructions enable building a wide array of on-chain contracts, but the limiting factor is the precomputation.
In a purely theoretical sense, CTV with taproot can express extremely complex but finitely enumerable transaction transition graphs, but only by exhaustively enumerating every reachable state and transition.
In practice, this is computationally infeasible beyond a tightly bounded parameter set.

What I found is that this scheme is practical for some modest parameters, but the precomputation speed harms the user experience quickly as the parameter sizes increase.
Caching can and should be implemented to mitigate this latency on subsequent runs, but even with that, the initial vault calculation can take a very long time.
As a result, the hope of "effectively unbounded" vaults, with parameters comfortably above any practical predicted usage, does not seem feasible.
Nevertheless, accepting some practical limits clearly communicated to the user, MCCV offers useful functionality.

# Comparison

This is far from the first Vault design, and it's not the first Vault design to leverage `OP_CHECKTEMPLATEVERIFY`.
As far as I'm aware, however, this is the first published Vault design to combine `OP_CHECKTEMPLATEVERIFY` with taproot in this way.

## simple-ctv-vault

True to its name, [simple-ctv-vault](https://github.com/jamesob/simple-ctv-vault) is about as simple as possible.
Simple CTV Vault is what inspired me to build MCCV, I wanted to see how practical more complicated constructions were.
simple-ctv-vault allows a single UTXO to either be unvaulted to a hot address after a delay, or swept to a cold address.
This is still the basic principle that MCCV uses, but it only supports a one-shot vault and unvault.

## Bryan Bishop's "Bitcoin Vaults"

I wish I had seen Bryan Bishop's [Bitcoin Vaults](https://github.com/kanzure/python-vaults) earlier in development.
This vault is conceptually and directionally similar to MCCV.
Like MCCV, it also divides its vault up into chunks of value, which it calls shards.
It supports restricted vault operations enforced by either deleted key presigned transactions, or `OP_CHECKTEMPLATEVERIFY`, but was created before taproot activation.
Without the benefit of taproot, increasing numbers of spending paths required increasingly large `witnessScript`s.
Bryan judiciously chose spending paths for pushing to cold storage, splitting the vault into many individual shards, and peeling off a single shard, with the rest remaining in the vault.
The last item in particular is similar to what MCCV provides.
My current intuition is that taproot's ability to express alternative spending paths efficiently substantially weakens the motivation for breaking the vault into many individual shards.
With MCCV, the user may withdraw enough increments to satisfy their requirements, and retain vault protections on the remaining balance.
Taproot efficiency also permits MCCV to provide deposits into existing vaults as well, which are not available in Bryan's protocol, though this is at the expense of precomputing many more states.
Although MCCV's protocol design was finalized before I discovered Bryan's work, some of the similar design elements like chunking vaults into shards and peeling quantities off in withdrawals felt like validation of this design direction.

## `OP_VAULT` and `OP_CHECKCONTRACTVERIFY` Designs

Hopefully without causing any controversy regarding covenant opcodes, I will just note that these opcodes largely supersede my design.
With these opcodes it is possible to avoid all of the precomputation that makes this vault awkward.
They additionally allow for single-step withdrawals, where the recipient can be identified in the withdrawal, facilitating thorough authorization by watchtowers, and enabling the recipient to directly claim their funds (though the recipient would need some additional information to sweep the output).
All of this being said, I view `OP_VAULT` and `OP_CHECKCONTRACTVERIFY` activation to be considerably less likely than `OP_CHECKTEMPLATEVERIFY`, and therefore it's worthwhile to explore the limits of a CTV-only design.

# Request for Feedback

I would greatly appreciate feedback on:

* The overall protocol design
* The tradeoffs of this design vs simpler ones
* General thoughts about vaults
* User experience - This *is* a proof of concept with known UX warts, but the set of user operations are present to test. Please consult the user guide.
* Performance and precomputation costs
 
# Conclusion

This scheme has plenty of drawbacks, but still offers an interesting improvement in security for managing funds.
It offers a compelling user experience without requiring `OP_VAULT` or `OP_CCV`, especially when improved with some of the future work below.

# Future Work

I have a lot of other work demanding my attention, so this isn't a commitment to work on these on any immediate timeline.
However, a few of these I consider very interesting and worth exploring in the near term.

Potential future work includes:

- watchtower software
All of the features a watchtower requires are present in the CLI, but not integrated into a coherent automated feature.
- delegated recovery
Allow semi-trusted watchtowers by delegating only the ability to initiate recovery.
- cached vault computation
Substantially improve the responsiveness of the software after the initial vault generation
- CPU optimizations
Various optimizations, many of which are low-hanging fruit.
- generalization
Adapt the software stack to support other CTV state machine protocols.
- GPU acceleration
The vault's parallel-friendly design also lends it to GPU acceleration.
- potentially STARK-based verification for hardware wallets

The STARK verification idea is particularly interesting.
It could allow powerful consumer hardware to generate vaults and proofs of their correctness, while more secure hardware wallets verify correctness with compact proofs.

-------------------------

reardencode | 2026-05-28 21:44:54 UTC | #2

Just for posterity: It seems like this would work just as well with OP_TEMPLATEHASH. Do you see any problems with that claim?

-------------------------

ademan | 2026-05-28 21:58:17 UTC | #3

No problems at all! In fact it's on The List (which is confusingly in the protocol doc for now). `OP_TEMPLATEHASH` should be a drop-in replacement, just replace `<hash> OP_CHECKTEMPLATEVERIFY OP_DROP` with `OP_TEMPLATEHASH <hash> OP_EQUALVERIFY`.

https://github.com/LNHANCE-Expedition/mccv/blob/b737434b44a5c0f43f9b9a40186b87b7a5f687a5/docs/protocol.md#op_templatehash

-------------------------

1440000bytes | 2026-05-30 00:17:54 UTC | #4

[quote="ademan, post:1, topic:2539"]
Protocol Specification (WIP): https://github.com/LNHANCE-Expedition/mccv/blob/17ed3e3220b673f1e57746d0b4609448e8064609/docs/protocol.md
[/quote]

[quote="ademan, post:1, topic:2539"]
I am currently working on a [detailed protocol specification](https://github.com/LNHANCE-Expedition/mccv/blob/17ed3e3220b673f1e57746d0b4609448e8064609/docs/protocol.md).
[/quote]

These links don't work.

<details>
<summary>Archive</summary>

https://web.archive.org/web/20260530001627/https://delvingbitcoin.org/t/ctv-only-vault-concept-v0-1-0-release/2539/4

</details>

-------------------------

