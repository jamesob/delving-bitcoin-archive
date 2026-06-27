# Is CSFS+CAT already a general covenant primitive? Three on-chain demonstrations, and an observation for discussion

AaronZhang | 2026-06-26 23:37:59 UTC | #1

I'd like to float an observation and ask people to poke holes in it.

**The observation (tentative).** CSFS+CAT --- both already activated on Inquisition signet --- seems, in practice, to behave like a *general* covenant primitive: on the introspection axis (committing to, or reading, transaction fields), the dedicated-opcode family (CTV, TXHASH, OP_VAULT, OP_TEMPLATEHASH, OP_PAIRCOMMIT) looks largely reachable on top of it. I don't think this is new as a *theoretical* claim --- Poelstra (2021) and Harding (2023) argued CAT-composition is broadly powerful years ago. What I tried to add is concreteness: I built three of these and **confirmed them on chain**, rather than asserting "CAT can do it."

**What I actually did --- three confirmed Inquisition-signet spends:**

1. **Output binding = CTV's output template.** Forces `sha_outputs` to a hardcoded value. tx [`2f345180…f37e6`](https://mempool.space/signet/tx/2f345180bc6654353a247a50559c0be42153707b2ca29b621e57671ed41f37e6) (block 300379). Writeup: [Delving /t/2483/3](https://delvingbitcoin.org/t/taproot-native-prevout-binding-via-sighash-preimage-decomposition/2483/3).
2. **Input binding --- something neither CTV nor APO does.** Forces the spend to consume a specific prevout set (`sha_prevouts`): i.e. "spend A only if B is co-spent" (the BitVM-bridge connector). tx [`7311da6e…3558`](https://mempool.space/signet/tx/7311da6e30886fab6594b633bcb97e4436d49f02ecc244d4bc7c59cb1ea83558). Writeup: [Delving /t/2483](https://delvingbitcoin.org/t/taproot-native-prevout-binding-via-sighash-preimage-decomposition/2483).
3. **Eltoo state replacement --- APO's *use-case*, not APO's primitive.** A CSFS+CTV delegation ladder reaches the eltoo topology. Note this is O(n) delegation, *not* APO's O(1) rebinding --- I'm reproducing the use-case, not the primitive. txs [`92efc475…`](https://mempool.space/signet/tx/92efc47554d25d74d9f567594d37375d8c8a1a2ea6bd1864600d5a49531fda34) (fund) + [`b96324da…`](https://mempool.space/signet/tx/b96324da612950339f564fbc88fe1d5c5751070e57bf796d32ee7171238382de) (settle). Writeup: [Delving /t/2430](https://delvingbitcoin.org/t/eltoo-state-chain-on-signet-again-three-rounds-two-transactions-csfs-ctv-with-rekey-and-ladder-no-cat/2430).

The mechanism behind all three is the same: **same-signature binding** (one signature must pass both `OP_CHECKSIG` against the node's sighash *and* `OP_CHECKSIGFROMSTACK` against a witness-supplied preimage --- so the witness preimage has to be the real one) + **preimage decomposition** (slice out and check any field). OP_VAULT, OP_TEMPLATEHASH, and OP_PAIRCOMMIT (which is a restricted CAT) follow by the same composition --- I have **not** demonstrated those on chain, and I mark them as reasoned, not experiments.

**Three things I'm deliberately NOT claiming:**

* This is **capability, not efficiency.** My scripts and witnesses are bigger; the dedicated opcodes are far cheaper, and that's exactly their point.
* It covers **signer-participating spends only** --- same-signature binding needs a spending signature, so genuinely keyless covenants are out of scope.
* I'm **not** arguing CAT is harmless. The constructions actually show the feared capabilities are real. My only point there is a *consistency* one: those same capabilities are reachable via the alternatives too, so singling out CAT isn't obviously consistent. And the narrow opcodes' narrowness is often a deliberate, valuable safety/review feature --- I'm not dismissing it.

**I'm not advocating for or against activating anything.** I'd genuinely like to know where this breaks: where does "largely reachable" fail? Which covenant patterns are *not* reachable with CSFS+CAT for signer-participating spends?

Full paper (byte-level constructions, every txid inline and clickable): https://github.com/aaron-recompile/btcaaron/blob/c6e6ce84adb81c9a05d15c4559625372f085ef22/examples/csfs_cat_covenant_parity/Capability_Parity_CSFS_CAT_Covenants.pdf . Critique very welcome.

-------------------------

