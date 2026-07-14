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

Laz1m0v | 2026-05-07 20:21:59 UTC | #2

This is brilliant engineering for the Script era, Aaron. But manually stitching 212-byte preimages together with `OP_CAT` just to bind two inputs feels like using flint to start a fire inside a nuclear reactor.

While you guys are doing stack gymnastics on Inquisition and praying for a 2030 soft fork, we are enforcing this exact topology directly in Simplicity (`.simf`) with a single mathematical assertion.

In our TUSM architecture, we don't hack preimages to simulate binding. The state is locked into the Taproot address itself via pure SHA-256 streaming. Absolute L1 bivalence. Pass or Trap.

Let `OP_CAT` rest in peace and come forge in the steel with us. The covenant *is* the consensus.

-------------------------

AaronZhang | 2026-05-13 18:26:24 UTC | #3

# Taproot-native output binding via sighash preimage decomposition — does this replace CTV?

`sha_outputs` (bytes 138–170 of the BIP-341 SigMsg preimage) is `SHA256(serialized outputs)` — the same digest `OP_CHECKTEMPLATEVERIFY` commits to. Reading this field with the same chunked-witness pattern as the OP gives output binding semantically equivalent to CTV.

### What was done

The tapscript hardcodes the 32-byte expected `sha_outputs`. The witness supplies the preimage in chunks split along the field boundary:

```
preimage = pre_a || pre_b || pre_c || sha_outputs || post
             46      46       46         32          42       (bytes)
```

Each chunk stays under 80 bytes for standardness. The script `OP_EQUALVERIFY`s the `sha_outputs` chunk against the hardcoded value, locks one chunk’s size with `OP_SIZE`, and lets same-signature binding anchor the rest — any shifted reassembly would require a SHA256 collision against the hardcoded value.

The script then reassembles the full 212-byte preimage with `OP_CAT`, computes the TapSighash tagged hash on-stack, and runs `OP_CHECKSIGFROMSTACK` + `OP_CHECKSIG` against the same 64-byte Schnorr signature. If one signature passes both checks, the witness preimage must equal the real sighash — so the `sha_outputs` chunk that was verified is the real `sha_outputs` of this transaction.

`OP_CAT` and `OP_CHECKSIGFROMSTACK` are both active on Bitcoin Inquisition signet, where the experiment runs.

### Tested on Inquisition signet

Fund A: [`05f9372d...`](https://mempool.space/signet/tx/05f9372dff7184ad736373165bc617ba100315bd2c187b0a4c893d796b2ca0cc) — 50,000 sats, output-binding tapscript hardcoding `sha_outputs = eb0fdcd005abb92861dfa7c7680c8a7b417e75c62c823b425615a0bee9082d7d`.

Spend A with the bound output (positive case) — confirmed: [`2f345180...`](https://mempool.space/signet/tx/2f345180bc6654353a247a50559c0be42153707b2ca29b621e57671ed41f37e6) (block 300379). Witness item 5 of that spend is the hardcoded `sha_outputs` value, byte-for-byte.

Attack: spend A with a different output (substituting amount or scriptPubKey) — rejected by `testmempoolaccept` with `Script failed an OP_EQUALVERIFY operation`. The substitution changes the real `sha_outputs`, the hardcoded check fails, and the spend never makes it into a block.

### What this means — does this replace CTV?

For output-binding semantics, yes: this construction enforces what CTV enforces. For deployment economics, no: the witness carries five preimage chunks plus the signature, and the script is larger than a single `OP_CTV` invocation. The point isn’t size or elegance — it’s that the capability sits on already-activated opcodes, alongside the OP’s `sha_prevouts` binding:

* OP ([Post #1](https://delvingbitcoin.org/t/taproot-native-prevout-binding-via-sighash-preimage-decomposition/2483/1)): `sha_prevouts` — input binding (which UTXOs are spent, beyond CTV)
* this reply: `sha_outputs` — output binding (where the funds go, CTV-equivalent)

Same technique, different sighash field, dual covenant semantics.

Happy to discuss.

-------------------------

AaronZhang | 2026-07-13 22:40:57 UTC | #4

**Why CAT+CSFS must be able to replace CTV？**

The two posts above were experiments. Only afterwards did I work out why they *had* to work — and the answer is about m.

A signature, in its cryptographic definition, is a **proof of knowledge of x such that P = xG**. That's a knowledge proof; it has nothing to do with "authorization." Authorization comes from m — Fiat-Shamir sets the challenge to e = H(R ‖ P ‖ m), binding the proof to one specific message. Change one sat, m changes, the signature dies. **Authorization is a byproduct of that binding, not the essence of the signature.**

And m isn't chosen by the signer. Consensus computes it: `m = f(tx)`.

`f` is the BIP-341 SigMsg construction rule. **The spender can change tx. He cannot change f.**

Now put every covenant proposal onto that equation:

|  | what it touches |
|----|----|
| **CHECKSIG** | accepts consensus's f and m, just verifies. **No constraint on m** |
| **CTV** | f unchanged, **constrains the value of m** (must equal a constant) |
| **APO** | **modifies f** — drops the prevout field from the preimage |
| **TXHASH** | **parameterizes f** — you pick which fields go into the preimage |

They aren't answering the same question. They're **reaching into different positions of the same equation**.

**So what does CSFS touch?**

CSFS is the only signature-checking opcode that lets you **supply m yourself**. m stops being an opaque value handed down by consensus and becomes an ordinary stack item.

And `OP_CAT` lets you assemble the preimage on the stack by hand — **which is to say, it lets you re-execute f inside the script.**

> **CSFS + CAT don't give you a configuration of f. They give you f itself.**

Once f lives in the script, the rest is corollary:

* **Want CTV?** Compute f, compare against a constant.
* **Want APO?** Omit the prevout field when assembling the preimage.
* **Want TXHASH?** Only include the fields you want.

(The CTV case has on-chain transactions above as proof. The latter two hold in terms of expressiveness; the exact semantic boundaries are worth discussing.)

So:

> **CTV isn't a new capability. It's a specialization of f — the most common constraint, frozen into a single opcode. APO is another specialization. TXHASH is another.**
>
> **They are all compile-time optimizations. CSFS+CAT is the general form being optimized.**

Faster, cheaper, smaller scripts — but nothing that wasn't already there.

The covenant proposal landscape finally takes shape: each opcode picks a spot on `m = f(tx)` and reaches in. CSFS+CAT sit at the root — they don't select a configuration of f. **They rewrite f.**

The transactions above are the empirical half. This is the structural half.

This also reframes an earlier thread of mine — [What exactly is bound in CSFS, IK+CSFS, and CHECKSIG?](https://delvingbitcoin.org/t/what-exactly-is-bound-in-csfs-ik-csfs-and-checksig/2351). That table catalogued *what* each pattern binds to; the `m = f(tx)` framing is the answer to *why* the catalogue looks the way it does. Worth a reread with that lens.

This line of thinking is developed further in [Is CSFS+CAT already a general covenant primitive?](https://delvingbitcoin.org/t/is-csfs-cat-already-a-general-covenant-primitive-three-on-chain-demonstrations-and-an-observation-for-discussion/2659) — three on-chain demonstrations and an observation for discussion.

-------------------------

