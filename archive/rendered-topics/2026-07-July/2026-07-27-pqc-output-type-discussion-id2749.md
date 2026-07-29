# PQC output type discussion

sipa | 2026-07-27 21:25:57 UTC | #1

This topic of PQC transaction output types in Bitcoin has been discussed in many different places, but often in somewhat unrelated threads. So I'm creating this thread in the hope to have those discussions in one place.

Before starting, I want to give an overview of what output types I have seen discussed so far, their variations, and their properties. I hope to keep this objective, and will go into my own opinion below. Feel free to suggest corrections or omissions, as I haven't followed this debate very actively until recently, and the list below is mostly informed by my own interests and the discussions I participated in.

### Output types

  * **P2MR** "Pay to Merkle Root" ([BIP-360](https://github.com/bitcoin/bips/blob/8c369ac8e60629ac6c032ffe21bb5ec5b35213d7/bip-0360.mediawiki)): The txout stores the Merkle root of a tree of leaf scripts. Spending reveals the Merkle path, the leaf script, and the inputs to satisfy it. I assume the addition of opcodes or leaf script versions that introduce PQC functionality (not included in BIP-360). This loses some spending efficiency compared to P2TR (see below), but does not reveal EC points on chain before spending.
  * **P2TRv2** "Pay to Taproot v2" (unclear origin): The exact same semantics as [BIP-341](https://github.com/bitcoin/bips/blob/9783d61f1b9c81231581fee026c8e8cb9499d265/bip-0341.mediawiki) P2TR, but with PQC opcodes/leaf types added, and with an expectation that a future consensus change will disable ECC keypath/opcodes *just within the output type itself*. Leaves an EC point exposed on chain for every output, but is most similar to today's usage.
  * **P2TRH** "Pay to Taproot Hash" (proposed [here](https://delvingbitcoin.org/t/public-key-recovery-for-ec-leaves-in-p2mr-bip-360/2603/17)): Like P2TRv2, but the scriptPubKey contains a hash of the tweaked point *Q*. Key path spending uses public key recovery (see below), giving it the same weight profile as P2TRv2, without EC point on chain. Needs a new [BIP-340](https://github.com/bitcoin/bips/blob/9783d61f1b9c81231581fee026c8e8cb9499d265/bip-0340.mediawiki) variant, and breaks batch validation.
  * **P2QR** "Pay to Quantum Resistant" (unclear origin): Identical to P2MR, but all ECC opcodes are disabled from the start. This makes it unconditionally quantum-resistant, but loses the efficiency of ECC even before Q-day.

#### Variations

  * **Public Key Recovery** (PKR) (proposed [here](https://delvingbitcoin.org/t/public-key-recovery-for-ec-leaves-in-p2mr-bip-360/2603)): a special leaf version is added whose "script" is just an EC public key hash. Spending needs just a signature, and verification is done through EC public key recovery to match against the hash. This reduces the spending size by ~32 bytes by not publishing the public key explicitly at spend time, at the cost of needing a [BIP-340](https://github.com/bitcoin/bips/blob/9783d61f1b9c81231581fee026c8e8cb9499d265/bip-0340.mediawiki) variant, and breaking batch validation. This could also be done through a separate opcode instead.
  * **New witness style** (proposed [here](https://delvingbitcoin.org/t/public-key-recovery-for-ec-leaves-in-p2mr-bip-360/2603/23), discussed more [here](https://delvingbitcoin.org/t/segwit-commitment-to-post-quantum-witness-data/2702)): this allows a new output type (or a new opcode) to access an additional witness extension to the transaction format, which can have arbitrary discount/costing rules for new witness data. This comes at the cost of needing to deploy a transaction serialization change and P2P protocol extension.
  * **Tripwire** (proposed on [ML](https://groups.google.com/g/bitcoindev/c/aWYtPLVPZ3U/m/htpzI5r3AgAJ)): a proof for ECDLP breaking can be published on-chain, and this automatically disables ECC opcodes/keyspaths *just within the output type itself*. This makes it unambiguously clear that EC disabling is expected.
  * **Miner lockdown** (proposed on [ML](https://groups.google.com/g/bitcoindev/c/aWYtPLVPZ3U/m/htpzI5r3AgAJ)): a softfork signalling mechanism (e.g. [BIP-9](https://github.com/bitcoin/bips/blob/8c369ac8e60629ac6c032ffe21bb5ec5b35213d7/bip-0009.mediawiki)) is used, not for a separate softfork, but to let a hashrate majority trigger disabling of ECC opcodes/keypaths *just within the output type itself*. This avoids the need for ecosystem action for EC disabling, at the cost of giving miners more power.
  * **Hybrid PQC opcodes** (one scheme discussed [here](https://delvingbitcoin.org/t/bird-of-prey-2-non-malleable-schnorr-pq-signatures/2514)): instead of offering pure PQC opcodes/paths, have them accept hybrid ECC+PQC signatures. This provides security if a PQC scheme were broken (possibility [classically](https://eprint.iacr.org/2022/975)) or used incorrectly (especially with efficient hash-based schemes being stateful), as long as ECC is not broken. The downside is (even) larger signatures and higher verification costs.
  * **Separate PQC leaf version** (proposed on [ML](https://groups.google.com/g/bitcoindev/c/aWYtPLVPZ3U/m/C-YAZUZaAgAJ)): restrict PQC opcodes to a separate leaf version which has no ECC opcodes. Depending on tripwire/lockdown/disabling semantics, this may avoid surprises when mixing both, at the cost of preventing the usage of (user-defined) hybrid schemes in script.

### Properties

(based on [this post](https://delvingbitcoin.org/t/public-key-recovery-for-ec-leaves-in-p2mr-bip-360/2603/22)</small> and [this post](https://delvingbitcoin.org/t/public-key-recovery-for-ec-leaves-in-p2mr-bip-360/2603/23))

|Property | P2TRv2 | P2TRH | P2MR | P2MR+PKR | P2QR|
|--- | --- | --- | --- | --- | ---|
|**Security**<sup>1</sup> |  |  |  |  | |
|&nbsp;&nbsp;&nbsp;After deposit | 🟥 | 🟨 | 🟨 | 🟨 | 🟩|
|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;+ PQC spend | 🟥 | 🟧 | 🟨 | 🟨 | 🟩|
|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;+ ECC spend | 🟥 | 🟧 | 🟧 | 🟧 | ⬜️|
|**Efficiency**<sup>2</sup> (1 ECC + 1 PQC) |  |  |  |  | |
|&nbsp;&nbsp;&nbsp;ECC sig spend bytes | 64 | 64 | 128 | 96 | ∞|
|&nbsp;&nbsp;&nbsp;PQC overhead bytes | 32 | 32 | 32 | 32 | 0|
|**Miscellaneous** |  |  |  |  | |
|&nbsp;&nbsp;&nbsp;Unmodified BIP-340 | :white_check_mark: | :cross_mark: | :white_check_mark: | :cross_mark: | ⬜️|

Where:
* 🟥: only secure if ECC was disabled, or no CRQC exists.
* 🟧: also secure under the combination of (a) there is no address reuse, (b) there is no pubkey sharing<sup>3</sup>, (c) no short-range CRQC exists, and (d) there is no collusion between long-range CRQCs and a hashrate majority
* 🟨: also secure under the combinaton of (a) no reuse and (b) no pubkey sharing, without the need for (c) or (d).
* 🟩: CRQC is no threat
* ⬜️: not applicable

---
* <sup>1</sup>: P2MR can be used in a PQC-only mode, which has equivalent security to P2QR.
* <sup>2</sup>: The bytes under "Efficiency" correspond to WU when used with the existing segwit witness. With new witness styles, they can be given arbitrary WU.
* <sup>3</sup>: P2MR can be used in a way that remains secure under a limited form of descriptor sharing before ECC spend, if all ECC opcodes are inside PKH constructions, and no actual EC points are shared. It is not compatible with xpub sharing, MuSig, adapter signatures, or FROST.

-------------------------

sipa | 2026-07-27 21:23:00 UTC | #2

Next I'll give my own (current) view. It is an opinion, and I would like to convince others of its merits, but is by no means a NACK of alternative options.

I believe the best solution is a combination of P2TRv2 and P2MR, covering distinct use cases.

* **To make "emergency" PQC available to as many users as possible**: a P2TRv2 output type, with Tripwire and Miner Lockdown, and with a hash-based PQC signature opcode.
  * The primary design goal is maximizing ease/incentives for adoption: every user/coin that adopts it, is one that is no longer subject to the steal-or-freeze dilemma. I consider that dilemma an severe threat to Bitcoin (a more urgent one than CRQCs themselves even), because both options involve giving up a fundamental value proposition Bitcoin has. Still, all coins that move to PQC-enabled outputs beforehand are taken out of the equation, and that long tail is something near-future development can have an impact on. Thus, I believe maximizing those ought to be our #1 priority. To that end:
    * P2TRv2 preserves the same fee profile as P2TR, avoiding "not migrating because it costs more", and does not lose the P2TR incentives that brings. This also minimizes negative impacts on Bitcoin prior to Q-day.
    * For developers/infrastructure, it is maximally close to today's world, avoids the complexity of adding a new witness style or a new ECC signature scheme.
    * It is compatible with most/all workflows I'm familiar with, including xpub/descriptor sharing (it would mean a static/short list of PQC keys in the descriptor, which isn't great for privacy, but much better than users not being able to adopt PQC at all).
  * I'm ambivalent about making the PQC opcode hybrid. Stateful signature schemes are scary, and only permitting them to be used in conjunctions with ECC would give me more confidence about not losing security due to incorrect usage. It however also adds complexity for adoption, and as mostly an emergency mechanism, users may opt for the safer stateless signature schemes anyway.
  * No P2MR or P2TRH. These add additional complexity, cost, or both, to users that may stifle adoption, and I believe the additional security to be had from the ability to not expose EC points on chain before ECC spending requires careful and uncommon workflows from users anyway (no key reuse, no xpub/descriptor sharing, very different hardware wallet designs, ...). Those who are willing to adopt those clearly do not need "ease of adoption", can thus use P2MR (see below) instead.
  * The primary goal is having something available that can be adopted quickly. The actual automatic ECC-disabling mechanisms (especially Miner Lockdown) that may be more bikeshed-prone could be introduced later when/if CRQC threats are closer.

* **To prepare for longer-term full migration**, a P2MR-based output type, with Tripwire and Miner Lockdown as well, plus a new [witness style](https://delvingbitcoin.org/t/segwit-commitment-to-post-quantum-witness-data/2702) to give its ECC usage the same fee structure as P2TR. It comes with hash-based PQC opcodes as well, and post EC disabling, newly discounted versions of those can be added.
  * The primary design goal is putting the pieces in place for a longer-term migration plan, even if the actual cryptographic schemes to make that realistic don't exist yet, are too slow/large, or have too low long-term confidence in their security. To that end:
    * No more P2TR variants, to avoid the overhead/complexity of a privileged ECC-only path which is useless in a post-ECC world.
    * Starts by adding infrastructure for new witness styles, even though that adds complexity, because a full migration post-CRQC will need those anyway. This likely complicates rollout and adoption, but that is fine; this is mostly intended for usage after Q-day.
  * Urgency is not as important here, and this can be deployed on a longer timescale than the P2TRv2 output type above. The use of a new witness style may need a lengthy upgrade anyway, because it needs a transaction serialization / P2P change to rely new witness data.
  * This can also functions as a fallback option for users to send theirs coins to post-CRQC if somehow ECC disabling does not happen in time, or users who want to safeguard their coins for that eventuality. I think the benefits of that option are limited, but recognize there is demand for it.
  * Depending on which PQC scheme is added here, and CRQC sentiment at the time, hybridization may be more important here, as there is less downside to its complexity and size, and this is intended for longer-term migration.
  * Use of PKR is possible here, but given that PKR verification is (slightly) slower than normal ECC verification, I don't think it's unreasonable to just use normal BIP-340 for ECC instead, and use a costing/discount model that compensates for it.

---

#### Regarding "relying on EC disabling"

[quote="conduition, post:32, topic:2603"]
I have more to say on P2TRv2, but for now I will hold my tongue and issue a request to proponents of P2TRv2: solve the EC-disabling timing problem, then let’s circle back and reassess.
[/quote]

Let's zoom out for a bit. *All* continued use of ECC in the face of rising CRQC probability relies on some entities making the call as to when ECC is to be considered vulnerable, and none of them have perfect visibility into whether a CRQC might exist. The differences between the various schemes are just in which entities those are:
1. Any scheme implicitly allows the **future Bitcoin ecosystem** to make that call, through an ECC-disabling consensus change.
2. With Tripwire, any **cooperative CRQC** can make that call.
3. With Miner Lockdown, the **hashrate majority** can make that call.
4. With P2MR and P2TRH, the **owners of the coins** can make that call too, under certain conditions.

The EC-disabling timing problem, as you call it, is the P2TRv2-specific lack of (4), I believe? That is undeniably a difference, but I think it is easy to overstate its importance, as in practice many if not most users of other other output types will equally rely on (1)-(3). Many casual users are likely not paying close enough attention, and will defer to others regardless. Any user that shared public keys/xpubs/descriptors to untrusted parties or reused addresses, gave up their ability to make that call, possibly in ways beyond their control (how can they stop being paid to the same address twice?), or even unknowingly.

So seen in perspective, P2TRv2 is effectively offering users the ability to opt out of making that call themselves. Given that many likely weren't going to exercise that anyway, and it comes with reduced cost, simplicity, and/or impact on the ecosystem, possibly increasing adoption before Q-day, I think that is a win. And once P2MR is available too, (4) remains an option for those actually willing and able to take advantage of it.

I'm not disagreeing that this results in a tenuous position of needing to rely on timely ECC-disabling, but I don't think it is unique to P2TRv2. It is the price we pay for wanting to continue to use ECC for as long as possible. 

[quote="conduition, post:32, topic:2603"]
Otherwise P2TRv2 is DOA, because it would be built on hope and not cryptography. Until then, I doubt further debate will be fruitful.
[/quote]

I don't think things are that black & white.

If you want a "cryptographic" level of confidence, the only option is P2QR (or even disabling ECC entirely). Anything else, e.g. P2MR, involves some hope that one of (1)-(4) above can act in time before a CRQC emerges. We don't do just P2QR, because of a (very reasonable) trade-off between adoption and security: P2QR would only be adopted by a tiny minority, so we accept (1)-(4) in addition to purely cryptographic assumptons, in exchange for (likely) far more coins being able to move to PQC outputs in the first place. Those who do not want to rely on (1)-(4) have the option of using P2MR in a PQC-only fashion.

Going from P2MR to P2TRv2 is a (IMO small) further step in the same direction: it accepts just (1)-(3) is enough, in exchange for getting an output type that is possibly more easily adopted, cheaper, with less impact on Bitcoin before Q-day. Those who do not want to rely on (1)-(3) alone and want (4) have the option of using P2MR still if both are available.

You can argue that that step is a step too far, and that this trade-off is not worth it. I would disagree with that, but it is a defensible position. Dismissing it as "not cryptography" is not a constructive stance, however.

#### Regarding "Timing Q-day accurately"

A common theme I see being brought up in predicting Q-day is that we shouldn't expect a nice progressing of milestones being broken along the way, [quipped](https://scottaaronson.blog/?p=9665#comment-2029013) by Scott Aaronson as:

> asking “so when are you going to factor 35 with Shor’s algorithm?” becomes sort of like asking the Manhattan Project physicists in 1943, “so when are you going to produce at least a *small* nuclear explosion?”

I accept that view, and the conclusion that one shouldn't wait to deploy PQC until there are measurable milestones. That is, we should be working now towards providing PQC output types in Bitcoin. But that is what we're doing, I think, and this discussion is part of that.

But I also think it's good to not take this too far; I expect there will still be milestones, they will likely just happen at a compressed pace, shortly before CRQCs become feasible. The fact that we shouldn't wait to *deploy PQCs* isn't the same as saying we shouldn't wait to *disable ECC*. Even a few weeks notice may be enough with Miner Lockdown. If it's something that's clearly close to CRQC-levels, I can see that happening without much drama (note, again, I'm just talking about ECC disabling within the PQC output type(s), not general freezing).

This is an optimistic scenario. I obviously cannot promise that this is how things will go. If CRQCs happen entirely unexpectedly, and come into non-cooperative/malicious hands, we inevitably have a problem. However, I think there is little we can do today (besides providing a PQC output type) that improves the outcome in that situation: there will be chaos, forks with emergency general freezes, and various techniques implemented for recovery of old coins. Since it's likely an undermining of Bitcoin's value proposition anyway ("altcoin bootstrapped with Bitcoin's UTXO set"), I see little reason why recovery techniques couldn't be hardforked in even.

On the other hand, preparing for outcomes where ECC disabling can be timed correctly is something we can do today, so that is where I think our focus should lie.

-------------------------

jeanpablojp | 2026-07-29 13:39:19 UTC | #3

A small data contribution to the table: I [implemented BIP-360 P2MR in Bitcoin Core on regtest](https://delvingbitcoin.org/t/bip-360-p2mr-implemented-in-bitcoin-core-on-regtest-vector-results-measurements-spec-feedback/2751) and measured spends end to end, so some of the Efficiency estimates can be grounded in actual transaction numbers.

Measured with identical transactions where only the input side differs (one input, one P2TR output, `SIGHASH_DEFAULT`, a `<key> OP_CHECKSIG` leaf):

|spend | control block | weight | vsize|
|--- | --- | --- | ---|
|P2MR, 2-leaf tree | 33 B | 513 | 129|
|P2MR, 4-leaf tree | 65 B | 545 | 137|
|P2TR script path, 2-leaf tree | 65 B | 545 | 137|

The P2MR witness comes out 32 bytes lighter than the equivalent P2TR script path spend at every depth (except m = 7, where the control blocks fall on opposite sides of a compact size boundary and it is 34), so P2MR effectively gets one extra tree level for the price of a P2TR script path. These are script paths on both sides: a P2TR key path spend at 64 witness bytes stays far cheaper, consistent with the 64 vs 128 comparison in the table.

One omission that may be worth a line in the comparison, since adoption complexity comes up repeatedly: implementation cost. The consensus delta for P2MR as specified is ~96 lines on top of Core's existing taproot machinery, because witness v2 reuses tapscript execution, the BIP-342 sighash and the Merkle walk unchanged; only the commitment check differs. Of the types listed, P2MR (and P2QR, which shares its structure) is the only one that is nearly free to implement given taproot; P2TRH and PKR need a BIP-340 variant, and the new witness styles need serialization and P2P changes.

Happy to measure other configurations on regtest if concrete numbers would be useful here.

-------------------------

