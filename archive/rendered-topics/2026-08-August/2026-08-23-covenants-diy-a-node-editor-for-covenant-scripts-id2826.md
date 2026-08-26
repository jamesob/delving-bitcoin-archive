# Covenants.diy: a node editor for covenant scripts

askii21m | 2026-08-24 03:51:48 UTC | #1

I'm working on an open-source node editor for experimenting with covenant scripts.

[covenants.diy](https://covenants.diy/)

You can wire nodes together and it builds the taproot output, assembles the tapscript, computes the sighash, and runs the script in an interpreter that reports what each opcode did to the stack. Every hash and every signature is computed in the browser.

Every graph has a permalink, so the nine worked examples below open already built and running. Click one and step through it.

| Example | |
|---|---|
| [Vault](https://covenants.diy/g/Dv-X8wuxmB) | BIP 119. Delay to notice a theft, and a cold path to stop it |
| [Congestion control](https://covenants.diy/g/mDRppTu20W) | BIP 119. One transaction commits to a tree of payouts |
| [Delegation](https://covenants.diy/g/WEMIQfZ8rl) | BIP 348. Hand someone one spend without handing them your key |
| [Oracle payout](https://covenants.diy/g/zBdUaXPMTJ) | BIP 348 + 119 |
| [Rebindable state](https://covenants.diy/g/i2hiLVS5xF) | BIP 448. The update leaf is three bytes |
| [Rebindable state, the older way](https://covenants.diy/g/-OtKkAyIWP) | BIP 118. The same channel with an ANYPREVOUT key type |
| [Merkle proof](https://covenants.diy/g/S2e0EJ8jJG) | BIP 347. A script folds a leaf back into a root it commits to |
| [CAT-only covenant](https://covenants.diy/g/D-uv04aDnu) | BIP 347. No CSFS: the script builds its own signature |
| [Recursive covenant](https://covenants.diy/g/z0CxkxMX9e) | BIP 347 + 348. A coin that can only be spent back into itself |

If there is a construction you would like wired up as an example, say so and I will add it :slight_smile:.

A selector in the header sets which proposals you assume are active, and every script is marked enforced, degraded, or open against that choice.

[Source](https://github.com/askii21m/covenants-diy) is MIT, except the interpreter fork, which keeps the CC0 dedication it came with.

**Copying a permalink uploads the graph, which is what makes the link short; nothing else leaves the browser.*

-------------------------

nerd2ninja | 2026-08-25 23:21:08 UTC | #2

Oh wow, hey thanks for making this!

-------------------------

askii21m | 2026-08-26 12:02:49 UTC | #3

Thanks, glad it's useful!

The ruleset is a checklist now, so you can turn on any combination of opcodes instead of picking a bundle.

Currently implemented:

* `OP_CHECKTEMPLATEVERIFY`
* `OP_CHECKSIGFROMSTACK`
* `OP_CAT`
* `ANYPREVOUT`
* `OP_TEMPLATEHASH`
* `OP_INTERNALKEY`
* `OP_PAIRCOMMIT`
* `OP_TXHASH`

Working on adding `OP_CHECKCONTRACTVERIFY` and `OP_VAULT` at the moment.

Anything you make can be easily shared, hopefully this helps with the research around sharing new covenant protocols/constructions.

![Screenshot 2026-08-26 at 11.13.51 am|690x286, 75%](upload://wIE88bPukrMRsaRnOWx5tICK8AA.png)

-------------------------

