# LNHANCE bips and implementation

reardencode | 2024-01-07 18:41:08 UTC | #1

I've just opened pull requests for bips and implementation of a combination of BIP119, OP_CHECKSIGFROMSTACK(VERIFY) and OP_INTERNALKEY (together I'm calling these "LNHANCE").

https://github.com/bitcoin/bitcoin/pull/29198
https://github.com/bitcoin/bips/pull/1534
https://github.com/bitcoin/bips/pull/1535

This combination of features seems to be a potentially safe and powerful starting point for bringing output restricting covenants to bitcoin and simultaneously supporting the next iteration of lightning development in the form of ln-symmetry and PTLCs.

-------------------------

michaelfolkson | 2024-01-07 19:17:58 UTC | #2

(Copying across from my comment on the Core repo)

This should be opened to bitcoin-inquisition rather than this repo at this stage? I thought that was the whole point of bitcoin-inquisition. I can summarize why bitcoin-inquisition was introduced in the first place if that helps. It might seem overly finicky but having all proposed consensus changes (especially if they are new) discussed in the Core repo just adds too much noise to all the work that is being done on non consensus changes in Core (imo).

Also I'm interested in why you think LN-Symmetry would be better implemented not using APO?

-------------------------

reardencode | 2024-01-07 19:30:43 UTC | #3

[quote="michaelfolkson, post:2, topic:376"]
This should be opened to bitcoin-inquisition rather than this repo at this stage? I thought that was the whole point of bitcoin-inquisition. I can summarize why bitcoin-inquisition was introduced in the first place if that helps. It might seem overly finicky but having all proposed consensus changes (especially if they are new) discussed in the Core repo just adds too much noise to all the work that is being done on non consensus changes in Core (imo).
[/quote]

Inquisition is fine, and I'm happy to open a separate PR there for CSFS and INTERNALKEY if that is helpful to reviewers or others in the community. It is not, however, part of the flow for changes to bitcoin core. It is an _optional_ place to test significant changes before they are proposed for activation in bitcoin core if that is helpful to the process for any particular change.

[quote="michaelfolkson, post:2, topic:376"]
Also I’m interested in why you think LN-Symmetry would be better implemented not using APO?
[/quote]

APO is one way to achieve LN-Symmetry. In the range of possible ways to achieve LN-Symmetry, APO is somewhat of a closed development path. It introduces new key types for the existing sigops with a concrete set of sighash flags that cannot later be upgraded without adding another new set of key types. I have a PR open against APO that suggests an alternative design for those sighash flags, and there is a known issue that they should be changed to commit to the control block or merkle path to the script being executed in some modes. In short, there is a large design space for those sighash flags and a lack of direct upgradeability.

In contrast, CTV has a fairly narrow design space. The basic output commitment template is pretty clearly exactly as BIP119 describes it, and this can be combined with a well understood operation (CHECKSIGFROMSTACK) to provide the same committment as that used in @instagibbs LN-Symmetry implementation. Once we have CTV and CSFS in production for some time, we will be much better positioned to further analyze what other sighash modes might be useful for our next Tapscript key versions, or to analyze what hashing modes might be worth adding as a flags byte (or bytes) to CTV.

In short, this approach gives us small building blocks that are lowest common denominator for a variety of 2nd layer enhancements and position us to continue the discussion on future improvements.

-------------------------

michaelfolkson | 2024-01-07 19:55:48 UTC | #4

[quote="reardencode, post:3, topic:376"]
It is an *optional* place to test significant changes before they are proposed for activation in bitcoin core if that is helpful to the process for any particular change.
[/quote]

I mean.... sure it is optional. But if everyone takes your approach we get tens of different PRs in the Core repo from different authors all with different combinations of their favorite opcodes and sighash flags and supposedly that helps review and makes progress towards activation. In a repo that already suffers from being unnecessarily noisy. But ok, whatever.

[quote="reardencode, post:3, topic:376"]
APO is one way to achieve LN-Symmetry. In the range of possible ways to achieve LN-Symmetry, APO is somewhat of a closed development path. It introduces new key types for the existing sigops with a concrete set of sighash flags that cannot later be upgraded without adding another new set of key types. I have a PR open against APO that suggests an alternative design for those sighash flags, and there is a known issue that they should be changed to commit to the control block or merkle path to the script being executed in some modes. In short, there is a large design space for those sighash flags and a lack of direct upgradeability.
[/quote]

Thanks, I wasn't aware of this [BIP PR](https://github.com/bitcoin/bips/pull/1472) you opened. I suspect that isn't a universally agreed "known issue" unless you can point me to a different discussion I missed. But it does introduce a new public key type, if that's your criticism I guess that holds.

-------------------------

instagibbs | 2024-01-07 20:31:47 UTC | #5

Hi! Do you have a short list of the well-understood constructs that can be built using these, along with their relative idea maturity(your estimate is fine :) )?

I think it'd be useful to make it clear what you're getting. E.g.,, PTLCs are definitely possible today, but APO-like construction is certainly leaps nicer!

-------------------------

moonsettler | 2024-01-07 20:48:53 UTC | #6

Simple unidirectional NIC (non-interactive channel) that that is immune to TXID malleability and can be natural part of pool vTXOs (virtual UTXOs) from user `Alice` where you have a coordinator `Bob` that acts as an LSP:
```
<sigBob>
<sigAlice>
<template>
---
CHECKTEMPLATEVERIFY
<pubAlice>
OP_CHECKSIGFROMSTACKVERIFY
<pubBob>
OP_CHECKSIGVERIFY
```
You would have an alternative spending branch with a 24h timelock and `Alice` single sig. also have a 2-of-2 musig keyspend naturally.

`Alice` can keep giving `Bob` newer more favorable distributions. `Bob` gates them with his single sig.

What `Alice` would do is give `Bob` a new `<sigAlice> <template>` pairs and the new outputs that hash to the template to move funds to `Alice`. outputs can include HTLCs and then their updated settlement.

-------------------------

urza | 2024-01-08 06:31:45 UTC | #7

Would this also allow construction of other things that APO would? Namely channel factories and Spacechains?

Ping @RubenSomsen

-------------------------

moonsettler | 2024-01-21 23:24:01 UTC | #8

indeed. LN-symmetry is an other big thing it enables. (will try to dig up the contract, pretty sure someone already laid it out) it won't do everything APO can, and that's on purpose. just the most important things that are highly desired.

edit:

```text
# S = 500000000
# IK -> A+B
<sig> <state-n-hash> | CTV IK CSFSV <S+1> CLTV
```
before funding sign first state template:
```text
# state-n-hash { nLockTime(S+n), out(contract, amount(A)+amount(B)) }
# settlement-hash { nSequence(2w), out(A, amount(A)), out(B, amount(B)) }

# contract for state n < m
IF
  <sig> <state-m-hash> | CTV IK CSFSV <S+n+1> CLTV
ELSE
  <settlement-hash> CTV
ENDIF
```
`CLTV` ensures only a larger `nLockTime` transaction can spend the current on-chain state, the relative timelock for the last co-signed state's `CTV` distribution is committed to in the `settlement-hash`

-------------------------

reardencode | 2024-01-09 20:34:43 UTC | #9

Here's a new one, [Oracle slashing](https://twitter.com/jxpcsnmz/status/1744780287136137276) (just proposed).

That relates closely to [simplified DLCs with CTV](https://mailmanlists.org/pipermail/dlc-dev/2022-January/000102.html).

[Timeout Trees](https://github.com/JohnLaw2/ln-scaling-covenants/blob/386736fa15cdda3c0f24f17a31b6ad608d338b08/scalingcovenants_v1.3.pdf), pretty well specified.

LN-Symmetry, as written by @instagibbs, ports directly to LNHANCE (probably all of uses for `SIGHASH_ANYPREVOUTANYSCRIPT|SIGHASH_ALL`).

[Ark](https://arkdev.info), as originally proposed by Burak, and depending on ephemeral anchors.

[Simple coin pools](https://rubin.io/bitcoin/2021/12/10/advent-13/) (with n-n signing required to advance the state of the pool) simple to implement, only useful for small (e.g. family) groups.

[Simple CTV Vaults](https://github.com/jamesob/simple-ctv-vault) implemented, but somewhat hard to use.

edit to add: Didn't mention the basic ability of CSFS to be used in delegation. This is often not the best method, but can be useful. Script like `<pubkey> SWAP IF 2DUP CSFSV DROP ENDIF CHECKSIG`.

And many many many more that I don't have immediate links to.

-------------------------

RubenSomsen | 2024-01-10 15:30:30 UTC | #10

[quote="urza, post:7, topic:376"]
Would this also allow construction of other things that APO would? Namely channel factories and Spacechains?
[/quote]

Spacechains are already possible without covenants, but yes, just about any covenant proposal that lets you pre-commit the next tx would be useful for spacechains.

-------------------------

alex | 2024-01-17 14:51:04 UTC | #11

Jeremy Rubin's [uxos.org](https://utxos.org/uses/) is still a great resource for CTV-related applications

-------------------------

reardencode | 2024-01-19 05:42:31 UTC | #12

Tonight, I created a PR against @ajtowns 's binana to create numbers for CSFS and INTERNALKEY.

https://github.com/bitcoin-inquisition/binana/pull/1

I also created draft PRs for ctv, internalkey, csfs alone against core.

https://github.com/bitcoin/bitcoin/pull/29280
https://github.com/bitcoin/bitcoin/pull/29269
https://github.com/bitcoin/bitcoin/pull/29270

These 3 PRs include everything from #29198 except activation and functional tests that depend on CTV being active in regtest. They conflict with each other, due to overlap in SCRIPT_VERIFY_FLAGS numbers, which I'm not sure the best way to deal with.

-------------------------

hampus | 2024-01-19 21:44:31 UTC | #13

[quote="urza, post:7, topic:376"]
Would this also allow construction of other things that APO would? Namely channel factories and Spacechains?
[/quote]

According to rearden, this would allow for an APO spacechain-like construction.
https://twitter.com/reardencode/status/1748452392046334366

Presumably helpful for statechains too.

-------------------------

moonsettler | 2024-01-21 23:17:19 UTC | #14

a similar contract to LN-symmetry can be used for 'immortal' statecoins.

you can potentially do something even cooler, by stacking such constructs and thus transfer one end of an LN-symmetry channel, you can in theory reuse a single channel funding for any number of users.

in this case the final 'distribution' of the first channel opens the second (latest) one on-chain, then that has it's own timeout for latest state.

-------------------------

ZmnSCPxj | 2024-02-04 12:27:13 UTC | #15

I have been thinking as well about mechanisms with internal fees, i.e. the funds that will be used to pay onchain fees in case of unilateral exit are inside the mechanism, instead of requiring an additional UTXO to be kept onchain to pay for unilateral exits.  This allows for reduced blockspace use by not requiring anchor outputs and a CPFP-ing transaction, at the cost of requiring funds to be reserved per mechanism for the onchain fee in case of unilateral exit. Here is an attempt to design an extension for `OP_CHECKTEMPLATEVERIFY` for this: https://delvingbitcoin.org/t/sighash-outputdeltabounds/504/2?u=zmnscpxj

-------------------------

cryptoquick | 2024-03-15 19:19:59 UTC | #16

Hi, I've made a modified version of rust-bitcoin-script that can do an implementation of LN symmetry script, so anything that needs the bytes for script somewhere can easily get the right format.

https://github.com/cryptoquick/rust-bitcoin-script/blob/1a7e1742b3a699fa5a021a73211e36d92a2c3bd5/tests/test.rs#L26-L44

Forgive me if my byte counts are off in the example, I wasn't sure of the exact inputs.

I don't yet have support for IK, but I think that's a Taproot thing anyway, so not the same as script.

Thanks to @cguida for nudging me towards doing this. He'll be using this or something like it in trying to make an LNHANCE CLN plugin.

-------------------------

reardencode | 2024-03-19 20:38:03 UTC | #17

Nice, thank you!

IKEY is Tapscript only indeed, as it reads the BIP341 internal key from the control block and writes it to the stack.

-------------------------

moonsettler | 2024-10-30 12:55:15 UTC | #18

The LNhance family of opcodes has a new addition:

OP_PAIRCOMMIT (PC)

When used in sequence of `CTV PC IK CSFS` they form the backbone of the LNhance-Symmetry channel.

https://github.com/bitcoin-inquisition/binana/pull/8

-------------------------

moonsettler | 2024-11-24 15:13:09 UTC | #19

I heard some claims that `CTV` needs 3 other opcodes to do LN-Symmtery. That's wrong.

LNhance is `CTV` + `CSFS` at it's core and that is enough to do LN-Symmetry.
IKEY and PC are tiny composable pieces that optimize the use of `CTV` and `CSFS` for LN-Symmetry.

There are several ways to solve the data availability problem, some don't even require a consensus change. Some don't even require a relay policy change.

LNhance is just the best way of doing it that we know of.

-------------------------

