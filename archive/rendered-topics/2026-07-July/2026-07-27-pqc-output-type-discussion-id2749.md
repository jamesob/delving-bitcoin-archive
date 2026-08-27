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

AdamISZ | 2026-07-30 13:11:35 UTC | #4

[quote="sipa, post:2, topic:2749"]
A common theme I see being brought up in predicting Q-day is that we shouldn’t expect a nice progressing of milestones being broken along the way, [quipped](https://scottaaronson.blog/?p=9665#comment-2029013) by Scott Aaronson as:

> asking “so when are you going to factor 35 with Shor’s algorithm?” becomes sort of like asking the Manhattan Project physicists in 1943, “so when are you going to produce at least a *small* nuclear explosion?”

I accept that view, and the conclusion that one shouldn’t wait to deploy PQC until there are measurable milestones. That is, we should be working now towards providing PQC output types in Bitcoin. But that is what we’re doing, I think, and this discussion is part of that.
[/quote]

Largely off topic, but, I don't accept that view.

It's completely the wrong analogy for me. Bombs are the activation of something, not the stabilisation of that thing. QCs require control, not release; they are not bombs. So if you're going to compare with nuclear you have to argue it's more like fission reactors (that were created within a span of years) than fusion reactors (which were promised every decade since the 80s but still don't exist (economically), nearly 100 years after we knew they were possible in theory...).

-------------------------

conduition | 2026-08-09 19:34:40 UTC | #5

Hey @sipa and thank you for aggregating all this important information. I think your OP summary and analysis is correct, modulo small discrepancies in byte sizes (e.g. for pushdata length prefixing). 

Seeing @fjahr's impressive CISA proposal has forced me to reevaluate my opinions on PQ output type candidates. CISA provides a natural vessel for P2TRv2 which is asymptotically (as the number of <s>outputs</s> inputs grows) so efficient as to only require 1 byte witnesses for full-agg EC spends. This is exactly the kind of incentive we want in a PQ output type: Better efficiency than even P2TR, so much better that even users who don't care about PQ-security will migrate to it. The problem of course is that, like P2TRv2, CISA's output type requires we post bare EC pubkeys on-chain in the SPK and so it has the same PQ-security profile as P2TRv2.

P2MR can also be deployed with CISA, but we cannot use public-key recovery (except for with one input, according to @starius), and we must instead publish the EC public key for each aggregated signature. This would lower P2MR's asymptotic per-input EC spend cost from 96 bytes (with PKR), but only to 64 bytes (with CISA), almost at par with P2TR today. 

This trick works for P2TRH as well, with asymptotic EC witness weight reduced to 32 bytes per input (just one pubkey per full-agg output).

Of course, in all cases, these EC efficiency gains evaporate on Q-Day.

[quote="sipa, post:2, topic:2749"]
I believe the best solution is a combination of P2TRv2 and P2MR, covering distinct use cases.
[/quote]

If CISA didn't exist, i think deploying both P2TRv2 and P2MR in tandem as you suggest would be reasonable. 

However, if CISA is deployed concurrently, then P2TRv2+CISA will be so overwhelmingly efficient compared to P2MR (with or without CISA), that I expect most users will gravitate towards P2TRv2, and the economic security of P2MR coins will still ultimately hinge on P2TRv2's key-spend path being safely disabled in time.

That's not necessarily a bad thing - more users on addresses with PQ keys is a win. But this makes the stakes higher: **With CISA, disabling ECC and switching to PQC is now even more expensive than it would've been without CISA.** More users will be affected by such a change. We can expect even more fierce debate over when and how to activate the EC disable fork, and harsher consequences for timing it wrong. 

Possibly the risk is worth it, in return for the migration incentive. I'm not sure, haven't made up my mind yet. But regardless, we'll still want P2MR (possibly deployed without any ECC?) for maximum efficiency after Q-day. So it's just a question of whether we want P2TRv2+CISA as an (extremely enticing) intermediate stage in that migration path.

----

The far more interesting research avenue IMO is to try to bring P2MR's PQ efficiency up to speed with that of CISA, to the point that P2MR outcompetes legacy EC output types.

To that end, I have recently started exploring the idea of [PQ-SNARK signature aggregation](https://delvingbitcoin.org/t/post-quantum-signatures-and-scaling-bitcoin-with-starks/1584) of hash-based signatures, which is [the design that the Ethereum Foundation is pursuing](https://pq.ethereum.org/#roadmap). Recent advancements such as [Flock](https://blog.succinct.xyz/introducing-flock/) ([paper](https://eprint.iacr.org/2026/1329)) have brought this within practical reach, without introducing slow arithmetic hash functions like Poseidon as a dependency. 

For the uninitiated, this would allow us to aggregate thousands of hash-based signatures across a whole block into one succinct (few hundred KB) proof which would suffice to convince any validator that they verified a signature for every message/pubkey pair in a given block. The actual signatures never need to be saved, even by archival nodes.

My biggest concern with this approach is the complexity of circuit design. We don't want a repeat of [the Zcash inflation bug](https://x.com/zodl_co/status/2062022829990658305). But if this risk can be mitigated through formal verification or other modern cryptographic tooling, then maybe SNARK aggregation could be the silver bullet to incentivize migration to PQ. 

With a mostly constant-size block-wide proof of sig-validity, individual signatures would need to be weighed not by size, but by proving cost, which can be reduced to a few hundred hash compressions (or less with stateful signatures). Flock can prove validation of hundreds of such signatures per second, especially when parallelized. 

It would require some very careful re-engineering of Bitcoin's traditional size-based fee accounting systems, but if done correctly we could massively increase the network's throughput while also incentivizing migration to natively PQ-secure wallets, with no EC cruft carried over. 

I'm not sure how viable this strategy will be yet, but i want to mention it for consideration. The game board is very large, and we have many more and better options than I used to believe.

-------------------------

sipa | 2026-08-17 22:28:41 UTC | #6

[quote="conduition, post:5, topic:2749"]
Seeing @fjahr’s impressive CISA proposal has forced me to reevaluate my opinions on PQ output type candidates. CISA provides a natural vessel for P2TRv2 which is asymptotically (as the number of ~~outputs~~ inputs grows) so efficient as to only require 1 byte witnesses for full-agg EC spends. This is exactly the kind of incentive we want in a PQ output type: Better efficiency than even P2TR, so much better that even users who don’t care about PQ-security will migrate to it. The problem of course is that, like P2TRv2, CISA’s output type requires we post bare EC pubkeys on-chain in the SPK and so it has the same PQ-security profile as P2TRv2.
[/quote]

Interesting change of stance!

I love signature aggregation, primarily because of its privacy-incentive mechanism (it was the original motivation for [MuSig(1)](https://eprint.iacr.org/2018/068), and the broken [scheme](https://bitcointalk.org/index.php?topic=1377298.0) that preceded it), but I'm not convinced that it matters that much in this context. 

There are two hurdles to overcome when getting users to adopt new output types:
1. Software/infrastructure needs to be available. This is the hard one: you need to convince developers of open source wallets, hardware wallets, proprietary wallet software, and centralized custodians to implement it. Especially the last one seems to take a long time even for small changes like [*sending*](https://whentaproot.org/) to new address formats, and even longer for changing receiving/managing. As I see it, the latter is mostly through:
   * Companies out of business / projects being abandoned, and their users migrating to others, especially newer ones built in more feature-rich software stacks.
   * Companies seeing a business advantage directly, e.g. because there is a high-fee environment and they pay transaction fees as a business cost. It's possible that for some, PQC itself is sufficient here (maybe if thou-shall-use-PQC regulations would appear in some legislatures), but not the long tail.
   * Users demanding it at scale.
2. Users must want to adopt it. Software/infrastructure *forcing* users to migrate is somewhat possible but rare and frowned upon, and more commonly it's presented as an option ("legacy wallet" / "segwit wallet") retaining compatibility.

In my view, (1) is unlikely to be influenced significantly by feerate gains (IIRC CISA has a max WU reduction of ~28%, and only for extreme many-inputs few-outputs transactions), unless we re-enter a high-feerate environment that persists for a long time (months at least, business/project decisions tend to move slowly), unlike today where feerates are pretty much their historic lowest ever (in BTC terms, at least). On the other hand, (2) may be influenced more by feerate, but I don't think that's really the bottleneck. Rather, I think the converse effect is more relevant: users may *not* want to adopt things that *increase* cost. This is why I argued against (pure) P2MR before.

On the other hand, (1) is likely influenced significantly by implementation complexity, and CISA does add to that. Of course, I expect the actual aggregation to be optional, allowing adoption of P2TRv2 without it, but then the benefits and incentives of feerate reduction through aggregation also disappear. I even have a mild concern about entities taking a stance of "we'll do a big rewrite one day and switch from P2TR to P2TRv2+PQC+CISA, but stick with P2TR for now", causing them (and the ecosystem with it) to miss out on PQC adoption until that time.

I'm happy to hear more thoughts here, but (and I hate to say that) my initial reaction is actually that adding CISA to the mix may be a net negative for the goal of getting the long tail to adopt PQC, due to more complexity in getting the output type spec'ed and softforked in, and (probably) little change in incentives for adoption.

[quote="conduition, post:5, topic:2749"]
However, if CISA is deployed concurrently, then P2TRv2+CISA will be so overwhelmingly efficient compared to P2MR (with or without CISA), that I expect most users will gravitate towards P2TRv2
[/quote]

Unless we enter a high-feerate environment again (which I think we need in the long term, but I'm not very hopeful), I don't think CISA will change much here. If P2TRv2 and P2MR are both available, I expect some of the more sophisticated (users+software stacks) to choose P2MR, but P2TRv2 can act as default catering to the long tail ("just stick a PQC script path in there, and bump the witness version").

[quote="conduition, post:5, topic:2749"]
the economic security of P2MR coins will still ultimately hinge on P2TRv2’s key-spend path being safely disabled in time.
[/quote]

I agree.

[quote="conduition, post:5, topic:2749"]
That’s not necessarily a bad thing - more users on addresses with PQ keys is a win. But this makes the stakes higher: **With CISA, disabling ECC and switching to PQC is now even more expensive than it would’ve been without CISA.** More users will be affected by such a change. We can expect even more fierce debate over when and how to activate the EC disable fork, and harsher consequences for timing it wrong.
[/quote]

It depends on what the PQC, or even ECC, usage in P2MR is like. Have you seen the [discussion](https://delvingbitcoin.org/t/segwit-commitment-to-post-quantum-witness-data/2702/3) on new witness styles? In a post-CRQC world, we likely need to move away further from witness-serialized-size as resource limiting metric, and give more relative weight to computation, because in particular hash-based schemes have a much larger size per CPU than ECC, so size stops being a good proxy.

We may want to use a scheme like the one discussed in that thread in P2MR, even for the ECC part. This would complicate matters for deployment and adoption (needs P2P changes to relay additional witnesses), but if/when it is clear CRQC are coming, we'll want that complexity adopted as much as possible *before* Q-day anyway. I wouldn't do this for the intermediary-step P2TRv2 output type, but if the differentiation becomes that P2MR is more for the longer term, it does make sense to have it there from the beginning even if that delays P2MR somewhat.

With that, it becomes possible to assign arbitrary cost functions (as long as they're 32 WU per transaction or per input, depending on design) to new output types / spending mechanisms. For example, P2MR with a key-like script at depth 1 could be given the same cost as P2TR today. Or, it may even be reasonable to assign it a cost that's similar to CISA, without actually needing aggregation of the signatures. This can be justified too, due to batch validation at the transaction level actually having similar performance for validation as half-aggregation (full aggregation is still better).

---

> The actual signatures never need to be saved, even by archival nodes.

They still need to be relayed, however, to reach miners who can then aggregate them into blocks. I am [very skeptical](https://groups.google.com/g/bitcoindev/c/wKizvPUfO7w/m/CvmuBnDRCwAJ) about the idea of having aggregation be done incrementally by third-party relay nodes, as the bundling of transactions removes the ability to reason about them individually. I worry this will quickly incentivize direct submission to miners instead, entrenching existing mining pools.

It's a really cool development, but I do worry about it effectively hiding the real cost of bandwidth within the consensus layer that still exists. One possible future outcome, if we lose sight of that cost, is that the real consensus network is just mining pools + a few fast relays between them, and the rest of the network with weaker network connectivity lagging behind after just receiving the aggregate proofs. That's sufficient for auditing after the fact, but removes them real participation in the form of feerate estimation / mempool / replacement reasoning, ...

Now, all of this at the level of individual transactions is fine. If it were the case that an aggregate proof can be constructed that has a size comparable to say a dozen individual signatures, it would allow for effectively PQ CISA, where network users are (possibly very strongly) incentivized to self-aggregate prior to submission to the network already. I would be much more comfortable with such an evolution, but I suspect the numbers won't really work out for that.

[quote="conduition, post:5, topic:2749"]
But if this risk can be mitigated through formal verification or other modern cryptographic tooling, then maybe SNARK aggregation could be the silver bullet to incentivize migration to PQ.
[/quote]

I would caution about thinking of anything as a silver bullet; most if not all things come with significant trade-offs. I may also just have seen a few too many such claims :)

[quote="conduition, post:5, topic:2749"]
With a mostly constant-size block-wide proof of sig-validity, individual signatures would need to be weighed not by size, but by proving cost, which can be reduced to a few hundred hash compressions (or less with stateful signatures). Flock can prove validation of hundreds of such signatures per second, especially when parallelized.

It would require some very careful re-engineering of Bitcoin’s traditional size-based fee accounting systems, but if done correctly we could massively increase the network’s throughput while also incentivizing migration to natively PQ-secure wallets, with no EC cruft carried over.
[/quote]

I think this is the case regardless of block-wide aggregation or not: a PQC world will need resource metric limits that take weigh CPU more and bandwidth less (but still some). The constants/metrics involved will of course depend greatly on the technology used (what type of PQC, aggregation or not, ...), but we have options even without aggregation.

[quote="conduition, post:5, topic:2749"]
I’m not sure how viable this strategy will be yet, but i want to mention it for consideration. The game board is very large, and we have many more and better options than I used to believe.
[/quote]

Indeed!

-------------------------

conduition | 2026-08-19 17:25:20 UTC | #7

[quote="sipa, post:6, topic:2749"]
On the other hand, (2) may be influenced more by feerate, but I don’t think that’s really the bottleneck. Rather, I think the converse effect is more relevant: users may *not* want to adopt things that *increase* cost. This is why I argued against (pure) P2MR before.
[/quote]

Interesting, so your take is essentially that the fee efficiency of an output type only matters beyond the point where it becomes _less_ efficient than what currently exists? 

[quote="sipa, post:6, topic:2749"]
I’m happy to hear more thoughts here, but (and I hate to say that) my initial reaction is actually that adding CISA to the mix may be a net negative for the goal of getting the long tail to adopt PQC, due to more complexity in getting the output type spec’ed and softforked in, and (probably) little change in incentives for adoption.
[/quote]


That's fair, but CISA isn't so complex to implement locally, e.g. for single-signer wallets. I think @AdamISZ has even pointed out there are ways to simplify the execution of the aggregation protocol for the common special case where all cosigners are the same entity. And even if devs don't want to invest the time up-front in doing even the most rudimentary (half) aggregation, they could still very easily deploy the CISA output type using non-aggregated BIP340 signatures for spending. Adding half or full aggregation later is as simple as a software update without any user-action needed (e.g. v3.2.1 changelog: "We made your transactions 26% cheaper!"). So even if devs don't use CISA's advantages right away, the CISA output type is still a desirable target for devs and users, more so with PQC in the mix.

[quote="sipa, post:6, topic:2749"]
It depends on what the PQC, or even ECC, usage in P2MR is like. Have you seen the [discussion](https://delvingbitcoin.org/t/segwit-commitment-to-post-quantum-witness-data/2702/3) on new witness styles? In a post-CRQC world, we likely need to move away further from witness-serialized-size as resource limiting metric, and give more relative weight to computation, because in particular hash-based schemes have a much larger size per CPU than ECC, so size stops being a good proxy.
[/quote]

I've seen the thread but I'm still forming an opinion. Currently i gravitate away from the idea, even for P2MR. On one hand, verification cost is critical for performance and hash-based sigs do offer the advantage of blazing fast hardware-accelerated verification and lost cost-per-byte compared to Schnorr... but witness sizes do still matter.

Verification costs are a one-time cost per-node, but storage costs for archival nodes are eternal. A block size increase of the magnitude needed to make even the smallest (XMSS) hash-based sigs fee-competitive with ECC would be an 8 fold increase (4mb -> 32mb per block) _at least,_ larger (more like 64x) if we consider stateless signatures. *Someone* has to store all that witness data.

This amplifies the absolute cost for archival node storage massively. We'd move from a world where bitcoin's chain grows by a few hundred gigabytes per year, to a world where chain growth is measured in *terabytes* per year, quickly pricing out amateur archival node runners like myself.

Think of it this way: If verification of signatures suddenly became 8x slower, most node runners and users - except maybe those who run raspi nodes - wouldn't notice that much. IBD would get slower, but that's a one-time cost. We could even mitigate by exploiting [multithreaded verification](https://conduition.io/code/fast-slh-dsa-verification/#Results) which I believe core currently doesn't do.

OTOH if signatures and blocks suddenly became 8x larger, that'd be a big problem for the network even if the fee market also became 8x cheaper. IBD and block propagation would also slow down, esp for users with low network bandwidth, and many node runners would need to start pruning witnesses once they run out of HDD space.

Therefore, my stance is currently that to increase the block size, we must also allow retroactive aggregation of signatures (or blocks) via SNARKs to curtail chain growth, and while I think that road is worth researching, it is not in the cards in the near future IMO.

With SHRINCS we have not coupled our proposal to a blocksize increase - at least, not initially. If that's what the community wants, SHRINCS can be reparameterized accordingly to reduce cost-per-byte, but for our initial draft (coming soon!) we're not assuming such a controversial change will be necessary. Adding a new sig algo, especially a semi-stateful one, is controversial enough as it is! :sweat_smile: 

[quote="sipa, post:6, topic:2749"]
It’s a really cool development, but I do worry about it effectively hiding the real cost of bandwidth within the consensus layer that still exists. One possible future outcome, if we lose sight of that cost, is that the real consensus network is just mining pools + a few fast relays between them, and the rest of the network with weaker network connectivity lagging behind after just receiving the aggregate proofs. That’s sufficient for auditing after the fact, but removes them real participation in the form of feerate estimation / mempool / replacement reasoning, …

Now, all of this at the level of individual transactions is fine. If it were the case that an aggregate proof can be constructed that has a size comparable to say a dozen individual signatures, it would allow for effectively PQ CISA, where network users are (possibly very strongly) incentivized to self-aggregate prior to submission to the network already. I would be much more comfortable with such an evolution, but I suspect the numbers won’t really work out for that.
[/quote]

I agree the bandwidth cost must be accounted for somehow even in the best case scenario where signatures can be aggregated block-wide with zero cost by miners. I'm not sure how that'd look, still trying to determine if the current SNARK technology would even work for our use-case. Maybe there would be a fixed "min fee per byte" policy requirement for nodes to relay transactions, even though the size of the signature doesn't really count towards blockspace in that scenario, so that minimum might be very small depending on the size of signatures.

Re: TX-level aggregation, i agree. I am hopeful a more "fancy" PQ signature algo will support CISA-style aggregation someday. Currently the best I know of is [Falcon + LaBRADOR](https://eprint.iacr.org/2024/311), based on lattices, with aggregated sizes of about 80-100 kilobytes. But in the near-term it seems like SNARKs will remain far too large to be practical within bitcoin transactions.

-------------------------

sipa | 2026-08-20 18:18:02 UTC | #8

[quote="conduition, post:7, topic:2749"]
Interesting, so your take is essentially that the fee efficiency of an output type only matters beyond the point where it becomes *less* efficient than what currently exists?
[/quote]

In the long term, I believe/hope that fee effects matter. There will likely be times of mempool congestion with feerate spikes, and future businesses and projects will make decisions that are conscious of fee impact.

But for P2TRv2, which IMO ought to prioritize adoption by the long tail, and ideally soon to give a long window for that adoption. For that long tail, I expect the bottleneck to be software/wallet/custodian support by existing providers, and with high probability in the relatively short term, I don't think fee arguments will affect them much.

---

[quote="conduition, post:7, topic:2749"]
[quote="sipa, post:6, topic:2749"]
I’m happy to hear more thoughts here, but (and I hate to say that) my initial reaction is actually that adding CISA to the mix may be a net negative for the goal of getting the long tail to adopt PQC, due to more complexity in getting the output type spec’ed and softforked in, and (probably) little change in incentives for adoption.
[/quote]
That’s fair, but CISA isn’t so complex to implement locally, e.g. for single-signer wallets. I think @AdamISZ has even pointed out there are ways to simplify the execution of the aggregation protocol for the common special case where all cosigners are the same entity. And even if devs don’t want to invest the time up-front in doing even the most rudimentary (half) aggregation, they could still very easily deploy the CISA output type using non-aggregated BIP340 signatures for spending.
[/quote]

Your point here is about easy of implementation by developers. I agree in principle, except for a minor concern about "we'll look into this later" if the change seems complex, which I commented on earlier:

[quote="sipa, post:6, topic:2749"]
On the other hand, (1) is likely influenced significantly by implementation complexity, and CISA does add to that. Of course, I expect the actual aggregation to be optional, allowing adoption of P2TRv2 without it, but then the benefits and incentives of feerate reduction through aggregation also disappear. I even have a mild concern about entities taking a stance of “we’ll do a big rewrite one day and switch from P2TR to P2TRv2+PQC+CISA, but stick with P2TR for now”, causing them (and the ecosystem with it) to miss out on PQC adoption until that time.
[/quote]

The part you're responding to above is about *timing consensus changes*, not wallet implementation however. In short, I think adding CISA to the P2TRv2 bundle will increase the time it takes for P2TRv2 to become available, and that will likely reduce adoption before Q-day by the long tail more than fee incentives will increase it. I'm sympathetic to the fact that CISA inherently needs a new output type, which makes it a natural fit for combining, but I still think it distracts from P2TRv2's goal. If it's not bundled with P2TRv2, it will need yet another output type, be bundled with P2MR, or not done at all.

---

[quote="conduition, post:7, topic:2749"]
I’ve seen the thread but I’m still forming an opinion. Currently i gravitate away from the idea, even for P2MR. On one hand, verification cost is critical for performance and hash-based sigs do offer the advantage of blazing fast hardware-accelerated verification and lost cost-per-byte compared to Schnorr… but witness sizes do still matter.
[/quote]

Post-Q-day, if hash-based PQC is adopted, I think the need for a different witness costing rule (and thus larger (pre-aggregated) serialized block sizes) is almost an inevitability, regardless of whether block-wide aggregation is involved. The actual formula will of course be greatly influenced by the details, ideally accounting for all facets of resource impact (bandwidth, storage, validation, relay). Historically, serialized size and CPU validation costs were roughly proportional, justifying a largely size-based formula. But with cryptography that has a different CPU/size ratio, it seems inappropriate to only account for size. That doesn't need to mean that PQC sigs need to be competitive with ECC ones, but some capacity (in bytes) increase seems warranted if it doesn't come with a proportional CPU increase:

[quote="ajtowns, post:6, topic:2702"]
It might be that the decision matrix is for the next decade should be something like:

* ideal block size if Q-day doesn’t happen is 3.23MB – that maximises decentralisation
* ideal block size if Q-day does happen is 12.76MB – that trades off some losses in decentralisation, versus avoiding a significant decrease in transaction capacity

If that’s the case, then it might make sense to provide \~10MB of capacity in a “pqdata” area, but require as consensus that the pqdata area is only used for post-quantum signatures. That way if Q-day doesn’t happen, people don’t use the pqdata area and blocks stay closers to 3.23MB target (because a post-quantum signature uses a higher percentage of 10MB than an ECC signature does of 4MB), but if Q-day does happen, the additional capacity is already available.
[/quote]

IMO, we *will* need a new witness extension if Q-day happens, and it seems reasonable to aim to have the technology ready for that before Q-day. A way of doing that in a structured way that improves some of segwit's pain points is described in the [witness styles](https://delvingbitcoin.org/t/segwit-commitment-to-post-quantum-witness-data/2702/3) thread (but that's just one option), and it seems reasonable (but not necessary) to combine that with the later-stage P2MR construction, as it can give a fee structure even for ECC that matches the incentive structure of P2TR, or even exceeding it by matching (half-agg) CISA.

[quote="conduition, post:7, topic:2749"]
Verification costs are a one-time cost per-node, but storage costs for archival nodes are eternal. A block size increase of the magnitude needed to make even the smallest (XMSS) hash-based sigs fee-competitive with ECC would be an 8 fold increase (4mb → 32mb per block) *at least,* larger (more like 64x) if we consider stateless signatures. *Someone* has to store all that witness data.

This amplifies the absolute cost for archival node storage massively. We’d move from a world where bitcoin’s chain grows by a few hundred gigabytes per year, to a world where chain growth is measured in *terabytes* per year, quickly pricing out amateur archival node runners like myself.
[/quote]

Sure, *someone* has to store the full block data, but not *everyone*. Nodes and their operators themselves don't need full data (pruning doesn't reduce security or functionality), and if pressure on non-pruned nodes by IBD'ing new nodes grows too big, block storage could be sharded. Using [FEC](https://en.wikipedia.org/wiki/Error_correction_code) techniques it is possible to for example let every node store 20% of every block, in such a way that any combination of 5 nodes together can let you reconstruct everything.

Of all the potential ways additional block data impacts network properties, I consider archive node storage size the least concerning. It's simply never even been close to a problem, and disks have been growing faster than the chain has for a long time already, and furthermore technological solutions exist that can push it further away even. 

[quote="conduition, post:7, topic:2749"]
We could even mitigate by exploiting [multithreaded verification](https://conduition.io/code/fast-slh-dsa-verification/#Results) which I believe core currently doesn’t do.
[/quote]

Yes, [it does](https://github.com/bitcoin/bitcoin/pull/2060), since 2013 (inside blocks, not for individually relayed transactions).

[quote="conduition, post:7, topic:2749"]
IBD and block propagation would also slow down, esp for users with low network bandwidth, and many node runners would need to start pruning witnesses once they run out of HDD space.
[/quote]

IBD is for most users not bandwidth constrained (but we're getting closer in the upcoming 32.0 release thanks to a number of performance improvements), and I don't think pruning is an issue.

Bandwidth and CPU cost for validation at the tip however is much more concerning to me, as it's what allows the network to stay in sync, both between miners, and between nodes. But I also don't think we need to stick to limits set 9 years ago with the adoption of segwit.

[quote="conduition, post:7, topic:2749"]
Therefore, my stance is currently that to increase the block size, we must also allow retroactive aggregation of signatures (or blocks) via SNARKs to curtail chain growth, and while I think that road is worth researching, it is not in the cards in the near future IMO.
[/quote]

Aggregation, even at the block level, won't remove the need for bandwidth for transaction relay prior to block building. It removes one facet of the impact of increased sizes (the ability for weak nodes to keep up with auditing the chain). The ability for nodes to participate in providing the market for block space (i.e., the mempool), which is also an important facet of decentralization (allows for permissionless entry into the mining market, without centralized block-building services), isn't really affected by aggregation. Improving one may justify an increase in the resource limit on pre-aggregated block data, but I don't think it's reasonable to disregard that limit entirely and only look at aggregated size.

My stance is that resource limits in blocks should, for the time being, be set in terms of the CPU usage and pre-aggregated bandwidth imposed on validation nodes. Some increases in bandwidth are warranted, both because network and processing capacities have increased the past decade, and because with hash-based signatures, the CPU cost of validation doesn't have the same per-byte cost anymore. Aggregation is something that can be explored independently, and when it's ready, maybe that's a justification for future resource costing.

[quote="conduition, post:7, topic:2749"]
With SHRINCS we have not coupled our proposal to a blocksize increase - at least, not initially.
[/quote]

I agree, it shouldn't. I think in general the "near-term P2TRv2" idea shouldn't include any such thing in general.

[quote="conduition, post:7, topic:2749"]
If that’s what the community wants, SHRINCS can be reparameterized accordingly to reduce cost-per-byte
[/quote]

I don't really see what that has to do with the signature scheme. To give a different weight to block data (below 1 WU/byte) you need a new witness extension, which is a combination of P2P-level and consensus-level changes, reaching beyond script or signature logic.

[quote="conduition, post:7, topic:2749"]
Adding a new sig algo, especially a semi-stateful one, is controversial enough as it is! :sweat_smile:
[/quote]

Yes...

[quote="conduition, post:7, topic:2749"]
Maybe there would be a fixed “min fee per byte” policy requirement for nodes to relay transactions, even though the size of the signature doesn’t really count towards blockspace in that scenario, so that minimum might be very small depending on the size of signatures.
[/quote]

Policy won't suffice here. The reason for having resource limits is to prevent miners from creating blocks that have a too large impact, on the network and on each other.

-------------------------

conduition | 2026-08-22 05:53:21 UTC | #9

[quote="sipa, post:8, topic:2749"]
In short, I think adding CISA to the P2TRv2 bundle will increase the time it takes for P2TRv2 to become available, and that will likely reduce adoption before Q-day by the long tail more than fee incentives will increase it.
[/quote]

I see what you mean... If we bundle CISA+P2TRv2 then P2TRv2 can't deploy until CISA consensus validation rules are ready. Perhaps you're right. I don't know enough about CISA's development progress to gauge how big a lift it would be to bundle. @fjahr would be a better source.

However I do think you underestimate the mass appeal of an output type with PQC support which is *also the cheapest output type available.* Heck, people seem to like P2MR quite a bit even though until very recently it was twice as expensive as P2TR.

[quote="sipa, post:8, topic:2749"]
Post-Q-day, if hash-based PQC is adopted, I think the need for a different witness costing rule (and thus larger (pre-aggregated) serialized block sizes) is almost an inevitability, regardless of whether block-wide aggregation is involved.
[/quote]

[quote="sipa, post:8, topic:2749"]
IMO, we *will* need a new witness extension if Q-day happens, and it seems reasonable to aim to have the technology ready for that before Q-day.
[/quote]

If hash-based sigs become the default, then I agree we'll need to do *something* drastic, and maybe that'll be SNARK aggregation, or maybe it'll be an extension block. Recently i've also started thinking maybe we could do both: SNARK aggregation happening _after_ blocks are mined, to assuage the long-term storage impact and make IBD faster.

My hope is that it won't come to that, and someone will discover a highly efficient PQ signature scheme based on isogenies or lattices or something in 5-10 years' time, and this can be backstopped by hash-based signatures for safety. But i do appreciate that we should prepare first for the worst-case, and work with what we have without assuming something better will magically appear.

[quote="sipa, post:8, topic:2749"]
if pressure on non-pruned nodes by IBD’ing new nodes grows too big, block storage could be sharded. Using [FEC](https://en.wikipedia.org/wiki/Error_correction_code) techniques it is possible to for example let every node store 20% of every block, in such a way that any combination of 5 nodes together can let you reconstruct everything.
[/quote]

Neat idea. how would one do this in a robust way without some central directory?

[quote="sipa, post:8, topic:2749"]
Yes, it does, since 2013 (inside blocks, not for individually relayed transactions).
[/quote]

aha, i stand corrected!

[quote="sipa, post:8, topic:2749"]
Bandwidth and CPU cost for validation at the tip however is much more concerning to me, as it’s what allows the network to stay in sync, both between miners, and between nodes.
[/quote]

I'm somewhat undereducated here. Could you tell me, what are the key metrics you care most about in this vein? Block propagation speed? Transaction relay speed?

Do you know of any research on the effects of larger signatures on these metrics? 

[quote="sipa, post:8, topic:2749"]
The ability for nodes to participate in providing the market for block space (i.e., the mempool), which is also an important facet of decentralization (allows for permissionless entry into the mining market, without centralized block-building services), isn’t really affected by aggregation. Improving one may justify an increase in the resource limit on pre-aggregated block data, but I don’t think it’s reasonable to disregard that limit entirely and only look at aggregated size.
[/quote]

Agreed. Block-wide aggregation wouldn't affect the efficiency of mempool transaction relay, so any costing adjustments that consider aggregation also have to keep TX relay in mind too. We can't just fork in super huge signatures even if they have near zero verification cost. But we can try to strike a balance between performance and size. Thankfully hash-based sigs are *extremely* configurable. 

[quote="sipa, post:8, topic:2749"]
[quote="conduition, post:7, topic:2749"]
If that’s what the community wants, SHRINCS can be reparameterized accordingly to reduce cost-per-byte

[/quote]

I don’t really see what that has to do with the signature scheme. To give a different weight to block data (below 1 WU/byte) you need a new witness extension, which is a combination of P2P-level and consensus-level changes, reaching beyond script or signature logic.
[/quote]

I'm saying that if compute cost is a more important metric, and a witness extension with a new steeper discount is acceptable, then SHRINCS can be parameterized to target a low computational cost per byte to match the discount offered in the the witness extension.

For example, currently SHRINCS parameters are designed such that verification cost per byte is about $\frac{1}{16}$ of BIP340 schnorr ($\frac{1}{4} \times$ schnorr in the rare case where SHA256 hardware acceleration is not available). Therefore we could justify putting 16x more signature bytes into a block and still incur roughly same CPU cost during signature validation.

However, we could go steeper, to as low as $\frac{1}{64}$ th of the cost per byte of Schnorr, without gaining too much in size. For example, SLH-DSA with parameter set `h=40 d=4 a=13 k=12 w=4` would give us 7696 byte sigs (about the same size as standard SLH-DSA-SHA2-128s) but which only cost about 1000 SHA256 compressions to verify. If you use hardware acceleration, that's about 64x cheaper per byte to verify than Schnorr, so a 64x block size increase would be feasible (modulo bandwidth and storage considerations). Stateful signatures can be made even cheaper.

We've been basing our decisions mostly around Schnorr as a yardstick for verifier performance, but if one is comfortable throwing the Schnorr yardstick out the window, we could do even wackier things, like budget for a specific number of SHA256 compressions per block and parameterize for that instead.

-------------------------

ajtowns | 2026-08-23 04:11:30 UTC | #10

[quote="conduition, post:9, topic:2749"]
[quote="sipa, post:8, topic:2749"]
if pressure on non-pruned nodes by IBD’ing new nodes grows too big, block storage could be sharded. Using [FEC](https://en.wikipedia.org/wiki/Error_correction_code) techniques it is possible to for example let every node store 20% of every block, in such a way that any combination of 5 nodes together can let you reconstruct everything.
[/quote]

Neat idea. how would one do this in a robust way without some central directory?
[/quote]

See this thread:

https://delvingbitcoin.org/t/fountain-codes-a-way-to-reduce-blockchain-storage-costs/2624

The 246/10 approach [described there](https://delvingbitcoin.org/t/fountain-codes-a-way-to-reduce-blockchain-storage-costs/2624/10) has you divide nodes into 246 groups, each storing 10% of total blockchain data, and you need to connect to (at least) 10 peers from distinct groups, not absolutely any combination of 10 peers.

-------------------------

fjahr | 2026-08-23 22:55:02 UTC | #11

[quote="conduition, post:9, topic:2749"]
I don't know enough about CISA's development progress to gauge how big a lift it would be to bundle. @fjahr would be a better source.

[/quote]

Happy to chime in, but maybe I am also the worst source since I am biased pro CISA :wink:

The two aggregation scheme BIPs (458, 459) as well as the CISA BIP 460 have had a few eyes on them. More would be better, of course, but as far as I am concerned they are complete with reference implementations and pretty extensive test vectors. There are pull requests to secp256k1 for BIP458 and BIP459. Review of the secp256k1 implementations is probably the biggest lift looking at how long Silent Payments took. I consider this the primary blocker in terms of implementation. The Bitcoin Core code is IMO comparatively simple and there should be enough review power if consensus is formed around the change.

[quote="conduition, post:7, topic:2749"]
And even if devs don't want to invest the time up-front in doing even the most rudimentary (half) aggregation, they could still very easily deploy the CISA output type using non-aggregated BIP340 signatures for spending.

[/quote]

I have been thinking about PQ P2TRv2 + CISA since @conduition started raising this as an idea on the ML. This discussion now even prompted me to make a revision of the BIP. This new version of BIP 460 that makes opted-out inputs plain BIP 341 key path spends, details in this BIP pull comments: https://github.com/bitcoin/bips/pull/2212#issuecomment-5387112895 (will update the PR after some initial feedback). What this should do is take wallet adoption out of the equation since there is no need for them to adopt any new signing logic. From their perspective P2TRv2 should initially be equivalent with or without CISA since a CISA-enabled witness v2 is a strict superset of P2TRv2's ECC key path. Wallet support for aggregation can be added at any time without pressure.

[quote="sipa, post:8, topic:2749"]
In short, I think adding CISA to the P2TRv2 bundle will increase the time it
takes for P2TRv2 to become available, and that will likely reduce adoption
before Q-day by the long tail more than fee incentives will increase it.

[/quote]

I don't think it has to. I think this is not an unreasonable take given how long Silent Payments has taken to make it into secp256k1 but I am much more optimistic.

I think there are primarily three dimensions that could introduce delays in user adoption: forming community consensus, technical implementation and review, and wallet adoption.

As mentioned above with the latest change using plain BIP341 key path spends as the opt-out in CISA, I think the wallet part can be taken out of the equation. The existence of CISA should not prevent any wallet from adopting P2TRv2 unless I am overlooking something here. I agree that the low fee market may mean that the savings are not compelling enough for everyone but at the very least Payjoin and Coinjoin users and by extension anyone that cares about their on-chain privacy are a developer and user group in the long tail that would be very excited and quick to adopt anyway based on the conversations I have had.

In terms of technical implementation and review I think CISA is currently far ahead of whatever the PQ part of P2TRv2 would look like. Sure, using hash-based signatures mostly takes care of security assumptions but the proposed schemes are all developed very recently and it feels like the whole field is still evolving. The first serious attempt at a library for such a scheme was published less than 2 weeks ago: https://delvingbitcoin.org/t/libshrincs-a-c-implementation-with-a-machine-checked-security-proof/2795 . This also goes for the design details of the necessary companion pieces:  tripwire, miner lockdown etc. And while certainly some of the most capable people in this space are focusing on this, compared to CISA I think there are still a lot more unknowns and discussions to be had (take this with a huge grain of salt obviously since I know a lot about one but not the other). The most interesting question here IMO is if the review work needed to get CISA across the line draws resources away from the PQ part which then delays that side. I think this is the case to a certain degree since we want the most experienced researchers and developers to review both ideally. But in the range of realistic timelines I don't think this should make a meaningful difference. On secp256k1 for example, where resources have been very scarce for a while, a group of new people are contributing and reviewing and afaik none of them are working on hash-based PQ stuff in parallel (I did not try to verify this). Part of the work on these hash-based and PQ in general seems to be new contributors that have joined the space recently and have experience in this particular area. The low-ish overlap between these two parts appears to be a strength here. But this is still new territory since we never had a soft fork that implemented two consensus changes that are so different. And I certainly didn't think I would argue there is enough review power for any big project in Bitcoin but in this case I am optimistic that if we have community consensus we can get both done without delay.

Last, community consensus. It's impossible to predict what will happen but I can see this going both ways: Either adding CISA gets more people on board quicker, particularly CRQC skeptics, or CRQC believers will argue against it as a waste of time since it wastes resources on ECC stuff. I think that offering benefits for the case that CRQCs are delayed or never happen should help to form community consensus much faster but I have been speaking with people interested in CISA much more frequently than people working on PQ stuff. And thinking a bit more in scenarios: It is highly likely that (in public perception at least) CRQC development with either slow down or speed up in the coming 1-2 years compared to current expectations. If there is a speed-up happening it is relatively easy to drop the CISA part from the soft fork that is in development. But if CISA is not part of the proposal but CRQC development slows down, there may be no appetite for it at all in the community and development/deployment/adoption could stall (wherever we are at that point). CISA can be an insurance to keep driving adoption in this scenario when CRQCs may be closer than they appear. The fact that CISA "distracts from P2TRv2’s goal" may not be a negative for reaching the goal regardless.

@sipa Curious where you see the bottlenecks that would lead to a delay of P2TRv2 if CISA is bundled with it.

-------------------------

sipa | 2026-08-24 17:56:36 UTC | #12

@conduition 

[quote="conduition, post:9, topic:2749"]
However I do think you underestimate the mass appeal of an output type with PQC support which is *also the cheapest output type available.*
[/quote]

I wish you were right, that was the whole idea behind Taproot! But we're five years in, have seen multiple periods of mempool congestion in that time (some taking months), and not even all providers support *sending* to it, let alone use it as default for new addresses. Adoption is mostly within systems built around newer features or technology stacks, and stupid hype stuff. Not the large-scale migration we'd want to see for PQC security, neither in terms of BTC nor in terms of users.

Realistically, I don't think we should expect CISA to have a bigger impact than that, I think. That's not to say it won't influence behavior at all, but as I said, I think it's more a long term thing that will affect future software projects/companies when people inevitably migrate anyway, not existing wallets.

To back this up with actual numbers, here are graphs with logscale vsize savings for P2WPH inputs -> P2TR key path spends, and then from P2TR key path spends to CISA. Full-agg CISA can improve over P2TR more than P2TR did over P2WPH, but only *for sufficiently complex transactions*. That's the point of course: incentivizing those, but adopting workflows that allow such transactions is an even bigger task than adding PQC as it will typically involve adding interactivity (for CoinJoin/PayJoin like constructions):

![consolidation|631x500, 50%](upload://nG42fFoq5HxffLg6HyiF6z0mnc8.png)
![coinjoin|631x500, 50%](upload://wEg5FYsEZJjqNXIAi9VQj4vTake.png)

[quote="conduition, post:9, topic:2749"]
Heck, people seem to like P2MR quite a bit even though until very recently it was twice as expensive as P2TR.
[/quote]

That sounds more like an argument that people like P2MR for other reasons than its economics. I don't see how you'd conclude from this that feerates are relevant to them?


[quote="conduition, post:9, topic:2749"]
If hash-based sigs become the default, then I agree we’ll need to do *something* drastic, and maybe that’ll be SNARK aggregation, or maybe it’ll be an extension block.
[/quote]

Just to make sure we're talking about the same thing. An extension block, as discussed years ago as a scalability proposal, is something very different and much more invasive than what we're talking about here. It's a completely new block area, with its own transactions and separate UTXO set, and mechanisms to move coins in both directions between the two areas. It's completely incompatible with existing wallet designs; you need transfers between the two blocks to pay to an old address. It can be done as a soft-fork, but it's probably the most invasive thing you can imagine that still qualifies.

I am just talking about a new witness in transaction serialization, like Segwit did, and the design discussed [here](https://delvingbitcoin.org/t/segwit-commitment-to-post-quantum-witness-data/2702) is somewhat less invasive than that even (no need for new wtxids or P2P changes beyond the transaction serialization). I don't want to minimize the impact either, it's still a big change, much bigger than just adopting a new output type or introducing a new signature opcode, but it is something the ecosystem has done before.

And I think we'll want this even if the long-term migration plan ends up using something else than hash-based. Because it's very unlikely it'll have the same size/verification characteristics as ECC, even if it's not as extreme as hash-based. And if we're going to need it anyway, I'd rather have the infrastructure in place beforehand.

[quote="conduition, post:9, topic:2749"]
Neat idea. how would one do this in a robust way without some central directory?
[/quote]

See the thread AJ linked to for a concrete idea, but I'd like to give an intuitive description.

Split your block up into 10-byte chunks. Then extend each of those chunks (implicitly, we won't actually compute/store these) to 256 bytes, by adding 246 error-correction bytes, in such a way that you can recover the whole chunk if you have any 10 bytes of it (and know which positions those bytes are from).

Every node now picks one (or a few) random numbers in range [0,255], and just stores those position bytes of every chunk. So for every number they pick, they store 1/10th of each block. For reconstruction, it suffices to pick peers which together have 10 distinct numbers. So it is indeed not quite the case you can have *any* 10 peers (or 5 or whatever, depending on the constants chosen), they need to be peers that chose distinct numbers. It's possible to switch to more complex codes which offer a larger range of numbers (making the probability of collisions in them lower), in exchange for more reconstruction complexity.

[quote="conduition, post:9, topic:2749"]
Could you tell me, what are the key metrics you care most about in this vein? Block propagation speed? Transaction relay speed?
[/quote]

The big one is block propagation speed: practically, minimizing the time between a miner finding a block, getting it across the network, up to the point where *other miners* can start hashing on top of a successor block. So this includes:
* time to relay transactions (if those weren't already relayed before)
* time to relay the block (which can use [compact blocks](https://github.com/bitcoin/bips/blob/7fe0b034ec967b52a5a28276419117326df93263/bip-0152.mediawiki) or [FIBRE](https://bitcoinfibre.org/))
* time to validate it along each hop (compact blocks typically allow relay before full validation, but it still needs reconstruction/PoW checking)
* time to validate it by the miner (which cannot be skipped; only for transactions not yet validated beforehand)
* time to build a new block template on top from mempool (in miner's nodes)
* time for that block template to make it to hashers (internal to miner setup, protocol changes have little impact here).

It's really only the first one that is directly impacted by block/transaction size, and it is hard to measure its real-life worst-case impact, because in practice most blocks are primarily filled with transactions that were relayed and validated ahead of time. That said, KIT has a [page](https://www.dsn.kastel.kit.edu/bitcoin/) with block and transaction relay statistics going back many years, which shows the impact of adoption of certain technologies and transaction composition changes.

[quote="conduition, post:9, topic:2749"]
I’m saying that if compute cost is a more important metric, and a witness extension with a new steeper discount is acceptable, then SHRINCS can be parameterized to target a low computational cost per byte to match the discount offered in the the witness extension.
[/quote]

Ah, of course!

I think aiming for roughly the same signature verificiation as BIP-340 seems like a reasonable rule of thumb. It does sound like a potential for bikeshedding, though.

---

@fjahr 
[quote="fjahr, post:11, topic:2749"]
This discussion now even prompted me to make a revision of the BIP. This new version of BIP 460 that makes opted-out inputs plain BIP 341 key path spends, details in this BIP pull comments
[/quote]

Cool. I think that's helpful.

[quote="fjahr, post:11, topic:2749"]
The existence of CISA should not prevent any wallet from adopting P2TRv2 unless I am overlooking something here.
[/quote]

I have one concern here, but it is admittedly a weak one that's probably addressable with education/communication. If a custodial company CEO hears "a simple change that just adds quantum protection", they may agree to put resources on implementing it quickly. If they hear "a new output type with several features like key aggregation and PQC", they may decide "We'll implement that whenever we need to rewrite that part of our stack anyway".

[quote="fjahr, post:11, topic:2749"]
but at the very least Payjoin and Coinjoin users and by extension anyone that cares about their on-chain privacy are a developer and user group in the long tail that would be very excited and quick to adopt anyway based on the conversations I have had.
[/quote]

Yeah, they would be the obvious parties who would want CISA. But I also expect them to be relatively quick adopters of a PQC output type without CISA? Just by virtue of being users/developers willing to be at the forefront of development. Under "long tail", I mostly think of many custodial and multi-currency software solutions; they tend to put more effort into supporting more altcoins/tokens than keeping up with (from their perspective, relatively) stable Bitcoin that won't gain them more customers.

[quote="fjahr, post:11, topic:2749"]
In terms of technical implementation and review I think CISA is currently far ahead of whatever the PQ part of P2TRv2 would look like. Sure, using hash-based signatures mostly takes care of security assumptions but the proposed schemes are all developed very recently and it feels like the whole field is still evolving. The first serious attempt at a library for such a scheme was published less than 2 weeks ago: https://delvingbitcoin.org/t/libshrincs-a-c-implementation-with-a-machine-checked-security-proof/2795 . This also goes for the design details of the necessary companion pieces: tripwire, miner lockdown etc.
[/quote]

I think that's fair. CISA adds a fair bit of complexity, but it may well be further ahead than some other aspects we'd also need.

[quote="fjahr, post:11, topic:2749"]
Last, community consensus.
[/quote]

This is where my concern lies mostly.

I feel like P2TRv2 and CISA kind of pull in opposite directions in terms of messaging. The output's goal is preparing for CRQCs, but then it also adds an optimization that stops working when that actually happens.

Maybe I'm wrong, and there is a good synergy between those parts of the community who would favor it, and I'm happy to support the idea if there is momentum. Still, my belief is that the large class of users we'd want to adopt PQC with a P2TRv2 output type will at scale not really adopt CISA anyway, so that doesn't help them. And in the other direction, if CRQC-skeptical CISA-fans exist, they may be annoyed at being required to add PQC support to get CISA?

-------------------------

ajtowns | 2026-08-25 15:06:32 UTC | #13

[quote="sipa, post:12, topic:2749"]
Taproot! But we’re five years in, have seen multiple periods of mempool congestion in that time (some taking months), and not even all providers support *sending* to it, let alone use it as default for new addresses. Adoption is mostly within systems built around newer features or technology stacks, and stupid hype stuff. Not the large-scale migration we’d want to see for PQC security, neither in terms of BTC nor in terms of users.
[/quote]

I don't think five years is the right figure there: rather I think there's largely four types of common payment paths:
 * single sig: taproot is very slightly more expensive than p2wpkh due to the larger output (ignoring the prisoner's dilemma aspect that mildly encourages taproot adoption)
 * n-of-n multisig: wasn't more efficient until musig became available which was either late 2024 for libsecp, earlier this year with 31.0 for core, and added to the spec for lightning in May 2026 but still only usable for private channels
 * k-of-n multisig: we've got FROSTsnap signing devices, but as I understand it we  don't have a spec/standard for FROST yet
 * complicated alternative scripts (hash-path or timelock path for HTLCs, eg): afaik in practice we don't have any complicated enough scripts in common use (ie, not "new features/tech stacks/stupid hype stuff") that they would be significantly cheaper with taproot

So I'd say the timeframe for fee-driven adoption of taproot has only been at most 6 months (for n-of-n; still waiting for k-of-n), and that's been in a time of record low fee levels. So I don't think you can draw any meaningful conclusions: prior to this year, getting cheaper fees was an open research problem; during this year, getting discounted fees probably doesn't justify the engineering time.

[quote="sipa, post:12, topic:2749"]
Realistically, I don’t think we should expect CISA to have a bigger impact than that, I think.
[/quote]

I think for exchanges it could be significant -- maybe reducing tx fees by 30% for larger consolidation txs; perhaps 60% if you got to switch from CHECKMULTISIG to FROST key path sigs at the same time. Still only meaningful if fee rates rise, though.

[quote="sipa, post:12, topic:2749"]
It’s possible to switch to more complex codes which offer a larger range of numbers (making the probability of collisions in them lower), in exchange for more reconstruction complexity.
[/quote]

I think you want a relatively high chance of collisions because the set of nodes you collide with is also your anonymity set as a node (because changing your number after you've picked it is only possible if you store 100% of the blockchain, which defeats the purposes).

[quote="sipa, post:12, topic:2749"]
[quote="conduition, post:9, topic:2749"]
I’m saying that if compute cost is a more important metric, and a witness extension with a new steeper discount is acceptable, then SHRINCS can be parameterized to target a low computational cost per byte to match the discount offered in the the witness extension.

[/quote]

Ah, of course!

I think aiming for roughly the same signature verificiation as BIP-340 seems like a reasonable rule of thumb. It does sound like a potential for bikeshedding, though.
[/quote]

8MB blocks with w=64 seems almost appealing (pq sigs ending up 2.77x as expensive as a schnorr sig, while being 5.5x the size of a schnorr sig, if I'm not totally off base with the maths); would probably require new p2p messages for chunking blocks or something though. Perhaps also a limit on an individual tx to not having more than... 100kB of pq sig data?

-------------------------

sipa | 2026-08-26 02:22:33 UTC | #14

[quote="ajtowns, post:13, topic:2749"]
So I’d say the timeframe for fee-driven adoption of taproot has only been at most 6 months (for n-of-n; still waiting for k-of-n), and that’s been in a time of record low fee levels.
[/quote]

I was mostly talking about single-sig; the situation is indeed more complicated for (efficient) multisig.

I think it's unlikely that the prisoner's dilemma played much of a role here. Senders were/are (almost completely) willing to send to P2WSH already, with a similar per-output cost as P2TR. With that, the choice to give our a P2TR address just reduces spending costs. A more plausible explanation is that receivers didn't want a different output type for change and for receiving, and that P2TR for both combined didn't seem sufficiently beneficial (or just that the potential fee gains were not interesting enough to them in general).

[quote="ajtowns, post:13, topic:2749"]
I think you want a relatively high chance of collisions because the set of nodes you collide with is also your anonymity set as a node (because changing your number after you’ve picked it is only possible if you store 100% of the blockchain, which defeats the purposes).
[/quote]

Ah, fair point.

[quote="ajtowns, post:13, topic:2749"]
would probably require new p2p messages for chunking blocks or something though. Perhaps also a limit on an individual tx to not having more than… 100kB of pq sig data?
[/quote]

I'd argue for a general tx weight limit (which would implicitly limit serialized size up to style=1). Some limit is necessary for verifiable chunking of blocks (send some range of transactions within a block, plus Merkle path for the transaction before and Merkle path for the transaction after) to be guaranteed to be possible. But a tx weight limit might not be a bad thing in general (from a block template optimization perspective).

-------------------------

AdamISZ | 2026-08-25 20:18:38 UTC | #15

I tend to agree with Pieter that it's somehow slightly 'off' to merge CISA into this. CISA, I agree, does not have nearly as big of a selling point in practice as some people want to believe.

It would be better to have it available as a "here's an output type that will be cheaper for your ordinary wallet when it matters, which is to say, when you have to consolidate a bunch of utxos" and, without overselling it, talk about 25% cheaper as a typical figure.

When taproot was about to release I was telling people "it'll take years for this to get adoption and it'll only really get *any* adoption when we get MuSig/PTLC in Lightning". My reasoning was: people are slow to switch and reluctant to switch unless they see a reason, which is either, cheaper fees or "there's a big chunk of the ecosystem using it". That was the case with segwit - it was very non-trivially cheaper so the ecosystem moved and then you got a snowball effect. If we'd actually got PTLC and so on in LN in a reasonable timeframe, it could have pulled a lot of the ecosystem with it. But as of now, it's only the tinkerer or functionally heavier wallets like Sparrow or the heavier ones like Liana that have it. The LN/PTLC angle didn't turn out to be straightforward.

The best CISA can hope for is that a 20-25% discount for consolidation matters, I think. The whole coinjoin/batching etc. etc. angle is going to remain niche unless something big changes. Big operators with a lot of backend stuff going on will perhaps care, though. And it would be nice if they didn't have to overlay another technical decision on top of it (quantum stuff).

-------------------------

fjahr | 2026-08-27 10:10:53 UTC | #16

[quote="sipa, post:12, topic:2749"]
I wish you were right, that was the whole idea behind Taproot! But we're five years in, have seen multiple periods of mempool congestion in that time (some taking months), and not even all providers support sending to it, let alone use it as default for new addresses.

[/quote]

One reason to be much more optimistic this time around is that in the end these are simply software changes and I think one major reason for slow adoption of Taproot in provider support is that there are many understaffed projects that had the will but not the manpower to make these changes safely. AI has made making such changes a lot faster, cheaper and (arguably) safer, especially when there is a spec that you can point the bots to. Generally I am not that concerned with the technical hurdles hindering adoption of wallets etc. because of this. Apathy of users to move coins and general disbelief in CRCQs happening are where I see the bigger issues.

[quote="sipa, post:12, topic:2749"]
Full-agg CISA can improve over P2TR more than P2TR did over P2WPH, but only for sufficiently complex transactions. That's the point of course: incentivizing those, but adopting workflows that allow such transactions is an even bigger task than adding PQC as it will typically involve adding interactivity (for CoinJoin/PayJoin like constructions)

[/quote]

We currently don't think that implementing fullagg in Payjoin requires additional communication rounds but we will confirm it with a PoC soon (work/research in progress between me, Payjoin Foundation, and others).

[quote="sipa, post:12, topic:2749"]
I have one concern here, but it is admittedly a weak one that's probably addressable with education/communication. If a custodial company CEO hears "a simple change that just adds quantum protection", they may agree to put resources on implementing it quickly. If they hear "a new output type with several features like key aggregation and PQC", they may decide "We'll implement that whenever we need to rewrite that part of our stack anyway".

[/quote]

As the concepts stand today (if I understand the PQC side correctly), P2TRv2 + PQC + CISA could be implemented and used without CISA (only use plain BIP341 opted-out) or without PQC (not implementing the required paths). So if there are just multiple options, I don't see how the decision would block anyone. If this was a concern to do everything at once I would expect that they first add it without CISA and then add CISA when they have additional resources for it.

[quote="sipa, post:12, topic:2749"]
Yeah, they would be the obvious parties who would want CISA. But I also expect them to be relatively quick adopters of a PQC output type without CISA? Just by virtue of being users/developers willing to be at the forefront of development. Under "long tail", I mostly think of many custodial and multi-currency software solutions; they tend to put more effort into supporting more altcoins/tokens than keeping up with (from their perspective, relatively) stable Bitcoin that won't gain them more customers.

[/quote]

I think that's mostly fair, though some have not adopted P2TR yet and many only did so recently so not all are on the bleeding edge. My AI comment above also means that I am not that concerned about custodial solutions. They can just move the funds for the users and they are a single point of contact that should be relatively easy to reach. I think reaching all self-custodial users and motivating them to move their funds is the harder part, even when their favorite wallet already has released an upgrade that supports the new output type.

[quote="sipa, post:12, topic:2749"]
I feel like P2TRv2 and CISA kind of pull in opposite directions in terms of messaging. The output's goal is preparing for CRQCs, but then it also adds an optimization that stops working when that actually happens.

[/quote]

[quote="AdamISZ, post:15, topic:2749"]
I tend to agree with Pieter that it's somehow slightly 'off' to merge CISA into this.

[/quote]

Generally I would never advocate for bundling unrelated, even contradicting changes that could never even be used on-chain at the same time. But in this unprecedented situation that the potential danger from CRQCs put us in, it is the only situation I can imagine where it makes sense IMO. Based on the conversations I have had so far I think only few people believe that CRQCs in the short/mid term are a 100% certainty and that basically everyone wishes (for Bitcoin's sake at least) that they never materialize because no matter what happens the transition will be extremely painful and we will lose many nice properties in the process. It is a pill we have to swallow, a preventative medication. I also think that there will be doubters of the existence of CRQCs until the last minute and, as written before, I see a high chance that we will see some leveling off of the Quantum progress before it happens. This is why I think bundling would help PQC adoption in many possible scenarios. It makes the pill not as bitter and gives the doubters a reason to still be interested in the change.

The messaging for the people that believe that CRQCs are a serious threat IMO doesn't matter that much because they will take what they need to be secure. They don't need to be convinced. But for those that don't see it as a danger (yet) the messaging matters much more and it may be better for them to hear "efficiency upgrade with PQC included just in case CRQCs do happen".

Another, weaker, angle how this is a natural match: As CRQCs appear to get closer and migration takes action demand for block space will rise necessarily. People will be interested in consolidating coins and going through coinjoins with coins they may be forced to move before transactions generally become a lot more expensive on-chain when PQC is the only option.

[quote="AdamISZ, post:15, topic:2749"]
It would be better to have it available as a "here's an output type that will be cheaper for your ordinary wallet when it matters, which is to say, when you have to consolidate a bunch of utxos"

[/quote]

Having a P2TRv2 with PQC and another P2TRv3 with CISA seems to be worse for adoption in my mind. Wallet implementers, custodial operators and non-custodial users would constantly need to make a decision between having quantum safety and cheaper fees/higher privacy (assuming payjoin/coinjoin has adopted CISA at this point). I think it would likely hurt adoption of both concepts more than a combined output type would. UI challenges and marketing/messaging still remain tough even with AI.

[quote="AdamISZ, post:15, topic:2749"]
without overselling it, talk about 25% cheaper as a typical figure.

[/quote]

I hope I am not overselling it? I am always trying to be careful to give correct numbers while simply being optimistic about the outlook of CISA adoption if it were to be deployed. Without being optimistic on that front it wouldn't really make sense for me to argue for it and work on it.

[quote="AdamISZ, post:15, topic:2749"]
When taproot was about to release I was telling people "it'll take years for this to get adoption and it'll only really get any adoption when we get MuSig/PTLC in Lightning".

[/quote]

I don't think that this situation is still comparable because those are mostly off-chain concepts and network-effects are at play whereas CISA is validated on-chain (we can't do a CISA softfork without the finalized and implemented DahLIAS spec) and current Payjoin/Coinjoin implementations only need to implement it for themselves and don't need to coordinate on it (only some more advanced concepts will do that). We will have demo implementations soon, for Payjoin certainly this year. In the case of MuSig in particular there was the indirection of the switch to MuSig2 which caused things to be delayed even further so BIP327 was merged into the BIPs repo \~1.5 years after Taproot activated and the MuSig2 module was merged into libsecp256k1 almost 3 years after Taproot activated. So it seems to me we are currently years ahead of that schedule for CISA in a hypothetical P2TRv2.

[quote="AdamISZ, post:15, topic:2749"]
The best CISA can hope for is that a 20-25% discount for consolidation matters, I think. The whole coinjoin/batching etc. etc. angle is going to remain niche unless something big changes. Big operators with a lot of backend stuff going on will perhaps care, though. And it would be nice if they didn't have to overlay another technical decision on top of it (quantum stuff).

[/quote]

Some people, myself included, do believe that CISA can help move Coinjoin/Payjoin out of the niche but I don't think this is the right place for this discussion. CISA inclusion would need to achieve community consensus and I think as long as the underlying information is correct (like savings numbers) it's fine that people have hope for the changes to have as big of an impact as possible. Without optimism I don't think we can get any softfork done.

But I don't understand what you mean by "overlay another technical decision", (taken mostly from my response to one of sipa's points above) as the concepts stand today (if I understand the PQC side correctly), P2TRv2 + PQC + CISA could be implemented and used without CISA (only use plain BIP341 opted-out) or without PQC (not implementing the required paths). If this was a concern I would expect that they first add it without CISA and then add CISA when they have additional resources for it. The other way around would also be possible, just adding CISA and ignoring PQC, with the separate output types for each concept you are actually forcing them to make a decision for either one, the change becomes more complicated and stretches into UI/UX, marketing etc. where decisions are less straight forward than simply making the code work.

-------------------------

