# Taproot-native prevout binding via sighash preimage decomposition

AaronZhang | 2026-05-06 19:35:12 UTC | #1

Robin Linus’s [How CTV+CSFS improves BitVM bridges](https://delvingbitcoin.org/t/how-ctv-csfs-improves-bitvm-bridges/1591) raises a specific problem: how to bind inputA so it can only be spent together with a specific inputB. AJ Towns pointed out at[ /t/1591/8](https://delvingbitcoin.org/t/how-ctv-csfs-improves-bitvm-bridges/1591/8) that the original construction binds to scriptSig bytes rather than the outpoint itself. I posted a sketch at [/t/1591/29](https://delvingbitcoin.org/t/how-ctv-csfs-improves-bitvm-bridges/1591/29) — using the `sha_prevouts` field of the sighash preimage as the binding anchor. This post gives the full witness layout, script reasoning.

### What was done

The script (\~196 bytes) hardcodes B’s outpoint (36 bytes of raw data — a transparent tweak factor). The witness supplies A’s outpoint. The script computes `SHA256(witness_A_outpoint || B_outpoint_hardcoded)` on stack and verifies it equals the `sha_prevouts` segment extracted from the chunked-witness preimage. Same-signature binding — one Schnorr signature satisfying both `OP_CHECKSIG` and `OP_CHECKSIGFROMSTACK` — ties the witness-supplied preimage to the actual transaction.

The witness is split along preimage field boundaries (each item < 80 bytes for standardness). The script uses `OP_CAT` to reassemble the 212-byte preimage, computes the TapSighash tagged hash on stack, then runs CSFS + CHECKSIG against the same 64-byte signature.

`OP_CAT` and `OP_CHECKSIGFROMSTACK` are both active on Bitcoin Inquisition signet, where the experiment runs.

### Tested on Inquisition signet

Spending A + B (positive case) — confirmed:

* [https://mempool.space/signet/tx/b7959a71b74b8bbed69d52c22a9b3b90af176391be2d0b7f33eb1a6473304606](https://mempool.space/signet/tx/b7959a71b74b8bbed69d52c22a9b3b90af176391be2d0b7f33eb1a6473304606)
* [https://mempool.space/signet/tx/7311da6e30886fab6594b633bcb97e4436d49f02ecc244d4bc7c59cb1ea83558](https://mempool.space/signet/tx/7311da6e30886fab6594b633bcb97e4436d49f02ecc244d4bc7c59cb1ea83558)

Attack: spending A + C (substituting a different UTXO for B) — rejected by `testmempoolaccept` with `Script failed an OP_EQUALVERIFY operation`.

### What this means in practice

For BitVM bridges: if B (the challenge anchor) is burned by a challenger, B is no longer in the UTXO set, and any transaction listing B as an input is invalid at the consensus level. The operator cannot withdraw A without proving B’s continued existence. More broadly, sighash preimage decomposition is a general method — `sha_prevouts` is one of several fields that can serve as a binding anchor.

Code: [https://github.com/aaron-recompile/inquisition-experiments/blob/91ca2fcee64d81e066d5cb645ae84dbd7bb27870/experiments/exp_sighash_prevout_binding_chunked.py](https://github.com/aaron-recompile/inquisition-experiments/blob/main/experiments/exp_sighash_prevout_binding_chunked.py) — happy to walk through the script step-by-step if useful.

-------------------------

