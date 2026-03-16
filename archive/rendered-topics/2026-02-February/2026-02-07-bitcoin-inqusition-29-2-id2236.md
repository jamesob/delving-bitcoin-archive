# Bitcoin Inqusition 29.2

ajtowns | 2026-02-07 13:49:27 UTC | #1

Bitcoin Inquisition 29.2 is now available from

https://github.com/bitcoin-inquisition/bitcoin/releases/tag/v29.2-inq

This is based on [Bitcoin Core 29.3rc2](https://github.com/bitcoin/bitcoin/releases/tag/v29.3rc2/).

This includes support for the following proposed consensus changes:
 * [BIP 118](https://github.com/bitcoin/bips/blob/584f4a732ba94199fc097fbb9b4660db868dd712/bip-0118.mediawiki) SIGHASH_ANYPREVOUT  ([PR#84](https://github.com/bitcoin-inquisition/bitcoin/pull/84))
 * [BIP 119](https://github.com/bitcoin/bips/blob/584f4a732ba94199fc097fbb9b4660db868dd712/bip-0119.mediawiki) OP_CHECKTEMPLATEVERIFY ([PR#85](https://github.com/bitcoin-inquisition/bitcoin/pull/85))
 * [BIN-24-1](https://github.com/ajtowns/binana/blob/8264328e6c7fd9e9f30efb8273fb94700f001454/2024/BIN-2024-0001.md) aka [BIP 347](https://github.com/bitcoin/bips/blob/740e826c19391a7a290933f514c15518e00780f0/bip-0347.mediawiki) OP_CAT ([PR#83](https://github.com/bitcoin-inquisition/bitcoin/pull/83))
 * [BIN-24-3](https://github.com/bitcoin-inquisition/binana/blob/3477736b354a8523615695971c6250cc4f98d452/2024/BIN-2024-0003.md) aka BIP 348 OP_CHECKSIGFROMSTACK ([PR#87](https://github.com/bitcoin-inquisition/bitcoin/pull/87))
 * [BIN-24-4](https://github.com/bitcoin-inquisition/binana/blob/3477736b354a8523615695971c6250cc4f98d452/2024/BIN-2024-0004.md) aka BIP 349 OP_INTERNALKEY ([PR#86](https://github.com/bitcoin-inquisition/bitcoin/pull/86))
 * [BIP 54](https://github.com/bitcoin/bips/blob/ed7af6ae7e80c90bcfc69b3936073505e2fc2503/bip-0054.md) Consensus Cleanup ([PR#90](https://github.com/bitcoin-inquisition/bitcoin/pull/90), [PR#99](https://github.com/bitcoin-inquisition/bitcoin/pull/99))

The first two of these have been active on the default signet since block 106704 (2022-09-06), the third since block 193536 (2024-04-30), and the fourth and fifth have been active since block 272592 (2025-10-06).

Note that transactions making use of these new features will not be relayed by nodes that don't support these features, so if you'd like your transaction to get to a miner, you'll need to peer with other inquisition nodes. An easy way to do this is to specify `addnode=inquisition.bitcoin-signet.net`. By default, Bitcoin Inquisition nodes will advertise themselves via the subversion string, which can be viewed with `bitcoin-cli -netinfo 3`. See the [23.0 announcement](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-December/021275.html) for more details.

BIP 54 activation has ~~not yet been signalled for on the default signet network, pending miner upgrades, but is expected to be soon~~ now been triggered for activation (as of block 290513), so should be active as of block 291168, in about five days.

-------------------------

AaronZhang | 2026-03-09 08:50:59 UTC | #2

I've been running experiments on the Bitcoin Inquisition signet, starting with OP_CAT. For this first round I built a witness-locked script: data lives in the witness, the script only verifies the hash — only the holder of part_a/part_b can spend.

Commit tx `084d5a9c6a8c176c24edc0a8b7ce54ed65808a326367d8a9299b4460ecaada09` → Spend tx `00072d4aa354b5987eb8f2ffec440db7467b0581c5e845a6a0ef6999b2d05656`

Code and full transaction details: https://github.com/aaron-recompile/inquisition-experiments

One observation: commit transactions show up on both standard signet explorers (mempool.space) and Inquisition nodes, but reveal/spend transactions only appear on Inquisition nodes — standard nodes reject the block entirely. The two networks fork at the first block containing a new-opcode spend. Expected, but worth noting for anyone trying to verify via public explorers.

CSFS, CTV, and INTERNALKEY experiments next.

-------------------------

AntoineP | 2026-03-09 17:10:16 UTC | #3

[quote="AaronZhang, post:2, topic:2236"]
The two networks fork at the first block containing a new-opcode spend. Expected
[/quote]

No, not expected! BIP 347 is a soft fork. Fortunately the spend tx [does show up](https://mempool.space/signet/tx/00072d4aa354b5987eb8f2ffec440db7467b0581c5e845a6a0ef6999b2d05656?showDetails=true) on Signet explorers. Presumably you actually checked *before* the spend was included in a block? It would not show up then because regular Signet nodes consider it non-standard and won't accept it in their mempool.

-------------------------

AaronZhang | 2026-03-09 18:34:21 UTC | #4

Thanks for the correction, Antoine — and for prompting a deeper look.

Verified on mempool.space/signet: the spend tx is there with 3600+ confirmations. You were right.

Your correction pushed me to think more carefully about why. Non-upgrade nodes accept the block not because they understand the new rule, but because OP_CAT is an unknown opcode in Tapscript — OP_SUCCESS under old semantics. The "network fork" I thought I observed was just a timing artifact: the tx never entered regular nodes' mempools (non-standard), so it was invisible before block inclusion.

CSFS, CTV, and INTERNALKEY experiments coming next — this time I'll reason from first principles before drawing conclusions.

-------------------------

AaronZhang | 2026-03-16 05:01:11 UTC | #5

Following up on my earlier OP_CAT note, I ran CSFS and CTV experiments on Bitcoin Inquisition and tracked timing end-to-end.

CSFS pair:

* Commit: 96df453d9e9ce50fdfca063528b03e3310033c3a61818bbe30e7fab5c61133e3
* Reveal (final, RBF replacement): 32fa307f3a570cfe93ebf7c101dba9ee8f289a5ca926dfed8baca92bb196e36b

CTV pair:

* Commit: 2378642548c7f86472d3998a0fcb2d364084783e487dd87c1e1020684aed51de
* Reveal: 9ccbce8ad87f0f94632119245a42537c9fbd2c8f706621f76f513339f220d55c

What I learned:

1. Visibility is layered (policy vs consensus).
   Before confirmation, reveal spends may be missing from standard signet mempools due to policy.
   After mining, they are visible on public explorers. So “not seen yet” != network fork.

2. Confirmation latency is the real variable.
   In my runs, commit->reveal was short, but reveal->confirm dominated and reached \~3.9 days.

3. Acceleration depends on script constraints.

* CSFS: a higher-fee replacement reveal confirmed quickly (\~113s after replacement broadcast).
* CTV: parent template constraints make direct replacement less flexible; CPFP is the practical accelerator.

4. Methodologically, recording reveal->confirm is essential.
   If we only record tx creation times, we miss the most interesting behavior.

Happy to share raw RPC snapshots / watch logs if others want to compare miner policy behavior.

-------------------------

