# OP_CC: A simple introspection opcode to enable cheaper consolidations

billymcbip | 2025-12-30 12:53:14 UTC | #1

Hello everyone,

I want to share an idea for a new Tapscript opcode that makes consolidation transactions significantly more space-efficient: **OP_CHECKCONSOLIDATION (OP_CC)**.

It's a simple introspection opcode: **OP_CC pushes false for the first transaction input spending a particular scriptPubKey, and true for every subsequent input spending the same scriptPubKey.** This enables a construction where all inputs spending the same SPK rely on the signature for the first input with that SPK.

A simple example:
* We create a Taproot SPK ("address") with a known key-path and a single Tapscript leaf of just OP_CC.
* Now assume we have three UTXOs with that SPK and we want to spend all of them in the same transaction.
* The first UTXO is unlocked with a simple key-path spend. It's crucial that the signature commits to all inputs and outputs (SIGHASH_ALL).
* The second and third UTXOs are unlocked with the OP_CC script-path spends. No signatures are required.

We save 58 witness bytes compared to spending all three UTXOs via key-path spends:
* Key-path witness size (per input): signature length (1), signature (64) → 65 bytes
* Script-path witness size (per input): script length (1), script (1), control block length (1), control block (33) → 36 bytes

And that's just a simple example. The savings are much greater for `n<m` multisigs, and greater still in a post-quantum world with much larger signatures.

I made a [draft implementation](https://github.com/billymcbip/bitcoin/pull/2) to explain the idea using code and unit tests.

I’ll post to the ML as well after collecting some initial feedback here on Delving.

Looking forward to your feedback.

-------------------------

fjahr | 2025-12-30 13:23:39 UTC | #2

Hi, this is probably a non-starter because it would encourage address reuse which has always been something we are trying to avoid. Also, you would get the same savings with [Cross-Input Signature Aggregation](https://cisaresearch.org/) of Schnorr signatures plus a lot more possible upside. I am a bit biased because I am working on it. And I am not very deep into the post-quantum stuff yet but for that scenario there is already a proposal called `OP_CIV` by Tadge Dryja that shares some similarities with your as well but is more flexible: https://groups.google.com/g/bitcoindev/c/oFbEQb_DB3I

-------------------------

billymcbip | 2025-12-30 14:39:49 UTC | #3

@fjahr:
> it would encourage address reuse which has always been something we are trying to avoid

Yes, undeniably. OP_CC is a simple way to increase the economic bandwidth of the base layer without increasing block size. We're seeing address reuse today, indicating that the privacy implications are bearable for many actors. And it's an opt-in feature.

>Also, you would get the same savings with [Cross-Input Signature Aggregation](https://cisaresearch.org/) of Schnorr signatures plus a lot more possible upside.

I'm not opposed to CISA! Clarifying question: For `n<m` multisigs using OP_CHECKSIGADD, wouldn't it still be necessary to reveal the script including the pubkeys in the witness? With OP_CC, the leaf script for repeated SPKs is a single byte.

Also, I'm not sure CISA is realistic for quantum signatures.

> there is already a proposal called `OP_CIV` by Tadge Dryja that shares some similarities

Yep, I'm aware. However, @adiabat concedes his proposal does not save space vs. EC sigs. See [https://youtu.be/cqjo3rmd6hY?t=1154](https://youtu.be/cqjo3rmd6hY?t=1154). Moreover, I think OP_CC is much easier for wallets to implement.

-------------------------

reardencode | 2025-12-30 18:44:51 UTC | #4

Something similar can be achieved with `OP_CHECKCONTRACTVERIFY` (CCV). I’m a bit rusty on the exact parameters, but in the simplest form you would have a taptree with a single leaf that requires that all value flow to output0 of the same spk with no signatures. This would enable signatureless consolidation with external fee payment. A UTXO using this construction would also be spendable with the key spend as normal to any destination.

cc @salvatoshi

-------------------------

billymcbip | 2025-12-31 12:16:33 UTC | #5

@reardencode: I think you're correct in the sense that `OP_CCV` supports consolidations with the following input parameters:
- `mode`: `CCV_MODE_CHECK_OUTPUT`
- `taptree`: `-1` (current input taptree merkle root)
- `pk`: `-1` (current input internal key)
- `index`: `0` (first output)
- `data`: empty

The problem is that the consolidation transaction can only roll value forward into the same contract. To pay a third party (new SPK), you need a second transaction spending the consolidated output.

-------------------------

salvatoshi | 2025-12-31 22:29:20 UTC | #6

[quote="billymcbip, post:5, topic:2177"]
The problem is that the consolidation transaction can only roll value forward into the same contract.
[/quote]

I think that's what people usually call a 'consolidation' transaction, so perhaps `OP_CHECKCONSOLIDATION` might be confusing for readers if the goal is more like a 'delegation' (that is: one UTXO delegate the authorization for spending to other UTXOs)

[quote="billymcbip, post:5, topic:2177"]
To pay a third party (new SPK), you need a second transaction spending the consolidated output.
[/quote]

You can have a scriptPubKey-based delegations with a script like:

```python
  CCV_MODE_CHECK_INPUT
  -1  # no taptweak
  <delegatee_pk>
  0  # first input
  <>  # no data tweak
  CHECKCONTRACTVERIFY
```
So this says "Everything goes as long as the first input is tr(<delegatee_pk>)". However, this Script is still 38 bytes, so this kind of delegation with CCV is probably useful to save bytes only if the `delegatee_pk` is spent with a rather large Script.

A delegation "to itself" (for example, checking that the first input has the same Script as the current input being spent), while expressible with CCV, would indeed not be safe as it would be tautologically true for the first input. So I think you are correct that OP_CCV does not implement your proposal.

-------------------------

sipa | 2025-12-31 16:59:33 UTC | #7

[quote="billymcbip, post:3, topic:2177"]
We’re seeing address reuse today, indicating that the privacy implications are bearable for many actors.
[/quote]

This seems backwards.

Privacy is a common good. If individuals, for whatever reason, take actions that hurt their own privacy, they are transitively also worsening privacy for everyone they interact with. The public nature of a blockchain means that any information revealed by anyone worsens the situation for everyone.

We can't prevent people from reusing addresses, or otherwise leaking information. But we (we being the collective of Bitcoin users that jointly decide the system's rules) can incentivize that behavior (through changes like the ones you propose here) or disincentivize it (through CISA, e.g., as it means CoinJoins slightly cheaper than individual transfers).

In any case, you shouldn't use the fact that some participants choose to give up their privacy as evidence that it's desirable to incentivize it for everyone.

-------------------------

billymcbip | 2025-12-31 17:35:57 UTC | #8

If consolidation transactions took up less block space, more block space would remain for everyone, including users who do not want to reuse addresses.

-------------------------

billymcbip | 2025-12-31 17:54:57 UTC | #9

> In any case, you shouldn’t use the fact that some participants choose to give up their privacy as evidence that it’s desirable to incentivize it for everyone.

@sipa, I didn't. I see the tradeoff and I made an empirical observation about on-chain data without value judgement. The scalability benefits outweigh the privacy implications in my opinion.

-------------------------

sipa | 2025-12-31 21:58:32 UTC | #10

It only benefits whose who engage in undesirable behavior - I don't see why you'd want to reward them for it.

Bitcoin was designed to have some level of transaction linking privacy. If its users didn't care about it, protocol designers should be aiming to move towards an account-based system, which naturally gives better scalability at the cost of not caring about revealing identity.

-------------------------

reardencode | 2025-12-31 23:01:47 UTC | #11

[quote="billymcbip, post:5, topic:2177"]
The problem is that the consolidation transaction can only roll value forward into the same contract. To pay a third party (new SPK), you need a second transaction spending the consolidated output.

[/quote]

Sure, but all we really care about is bytes not transactions. Using OP_CCV uses 5 extra bytes per consolidated input, plus a 1-input, 1-output transaction compared to OP_CC. It is a bit more expensive, but still much cheaper than current consolidations. The OP_CCV-style could be broadcast as a TRUC transaction using package relay, so it wouldn’t even need a separate fee input if the consolidated output is spent immediately.

-------------------------

1440000bytes | 2026-01-01 15:00:59 UTC | #12

[quote="sipa, post:7, topic:2177"]
We can’t prevent people from reusing addresses, or otherwise leaking information. But we (we being the collective of Bitcoin users that jointly decide the system’s rules) can incentivize that behavior (through changes like the ones you propose here) or disincentivize it (through CISA, e.g., as it means CoinJoins slightly cheaper than individual transfers).
[/quote]

CISA incentivizes all transactions with multiple inputs. This includes transactions that do not require coordination and all inputs belong to the same user.

<details>
<summary>Archive</summary>

https://web.archive.org/web/20260101145930/https://delvingbitcoin.org/t/op-cc-a-simple-introspection-opcode-to-enable-cheaper-consolidations/2177/12

</details>

-------------------------

billymcbip | 2026-01-03 12:46:34 UTC | #13

Is CISA realistic with quantum signatures?

-------------------------

murch | 2026-01-05 20:32:49 UTC | #14

In case you are looking for more opinions, financially incentivizing address reuse is a non-starter, and I sincerely doubt that this idea has any chance of adoption.

-------------------------

