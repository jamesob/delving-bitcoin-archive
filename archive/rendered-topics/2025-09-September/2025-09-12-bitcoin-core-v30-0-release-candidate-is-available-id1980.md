# Bitcoin Core v30.0 release candidate is available

fanquake | 2025-09-12 10:10:56 UTC | #1

Binaries for Bitcoin Core version v30.0rc1 are available from:

https://bitcoincore.org/bin/bitcoin-core-30.0/test.rc1/

Source code can be found under the signed tag

https://github.com/bitcoin/bitcoin/tree/v30.0rc1

This is a release candidate for a new major version release.

Preliminary release notes for the release can be found here: https://github.com/bitcoin-core/bitcoin-devwiki/wiki/v30.0-Release-Notes-Draft

There is also a testing guide here: https://github.com/bitcoin-core/bitcoin-devwiki/wiki/30.0-Release-Candidate-Testing-Guide

Please put your issues or comments here: https://github.com/bitcoin/bitcoin/issues/33368

Release candidates are test versions for releases. When no critical problems are found, this release candidate will be tagged as v30.0.

-------------------------

jsarenik | 2025-09-18 09:02:44 UTC | #2

Rebuilding Coinstatsindex to the new power-failure-safe one on pruned nodes with very low-power CPU takes days (like any IBD would there).

Would it be possible to provide a migration script in case muhash is still in sync with some recent known one?

For example, does this (still in the old coinstatsindex) look good? Comparing to my another pruned node’s (running 29.1) output it is in sync.

```
$ bitcoin-cli gettxoutsetinfo muhash 915117
{
  "height": 915117,
  "bestblock": "000000000000000000005ebf9d74ec7ab39402801cfbfdb763ac43cf18e45fcd",
  "txouts": 168188922,
  "bogosize": 13176628835,
  "muhash": "c539860039f893733c0adf51bdb559905cbfd88896561b41d62705c488dd17f0",
  "total_amount": 19922013.67356559,
  "total_unspendable_amount": 230.07643441,
  "block_info": {
    "prevout_spent": 7521.85097974,
    "coinbase": 3.20043866,
    "new_outputs_ex_coinbase": 7521.77554107,
    "unspendable": 0.00000001,
    "unspendables": {
      "genesis_block": 0.00000000,
      "bip30": 0.00000000,
      "scripts": 0.00000001,
      "unclaimed_rewards": 0.00000000
    }
  }
}
```

-------------------------

