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

moonsettler | 2024-01-07 20:46:56 UTC | #6

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

What `Alice`would do is give `Bob` a new `<sigAlice> <template>` pairs and the new outputs that hash to the template to move funds to `Alice`. outputs can include HTLCs and then their updated settlement.

-------------------------

