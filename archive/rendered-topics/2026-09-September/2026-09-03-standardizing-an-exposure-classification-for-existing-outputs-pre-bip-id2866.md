# Standardizing an exposure classification for existing outputs (pre-BIP)

duncan0k | 2026-09-04 12:43:17 UTC | #1

BIP 360 gives us an output type whose key stays off-chain until a script-path
spend, and BIP 361 proposes a phased sunset of legacy signature verification.
Both presuppose something no BIP currently specifies: a shared answer to
*"which existing outputs are exposed?"*



I've drafted a specification for exactly that, and I'd like this forum's
scrutiny before taking it further.



**Why I think it's worth specifying.** Published estimates of the exposed
supply range from \~25% to over 34%. Having chased the sources, I'm fairly
convinced the spread is definitional rather than measurement error — tools
disagree on whether P2TR counts as exposed at rest, whether a spent-from
address holding a balance differs from P2PK, and what to report for a P2SH
whose script was never revealed. If a BIP 361-style migration activates,
that question gets asked at scale, with money attached, by software that
ought to agree.



**What the draft does.** Four levels — `EXPOSED_AT_REST`,
`EXPOSED_ON_SPEND`, `NOT_EXPOSED`, `UNDETERMINED` — with a per-output-type
assignment table and a fail-closed rule: where the data can't distinguish two
levels, the more-exposed one must be assigned. Two consequences are likely to
be the contentious ones:



- A reused, spent-from P2PKH that still holds a balance is `EXPOSED_AT_REST`,
  the same level as P2PK. To an adversary they're the same situation, and
  labelling them differently quietly implies reuse is safer than P2PK.

- P2TR is at rest whether or not the internal key is NUMS, since consensus
  never checks how Q was constructed. As I read it, that's precisely why
  BIP 360 removes the key path — [the discussion in the BIP-360 changes
  thread](https://delvingbitcoin.org/t/changes-to-bip-360-pay-to-quantum-resistant-hash-p2qrh/1811)
  is what convinced me to state it this plainly.



Draft, a dependency-free Python reference implementation, and test vectors:

https://github.com/duncan0k/pubkey-exposure-classification



It's a draft, not a finished BIP. What I'd most like to know is whether the
four-level partition holds up against situations you've hit in practice, and
whether this belongs in a BIP at all versus staying an implementation detail.

If the answer is the latter, I'd rather hear it now.

-------------------------

Anzus_GemWallet | 2026-09-06 13:06:11 UTC | #2

From a wallet user’s point of view, each level may be more useful if it also has a clear recommended action—for example, no action needed, avoid reusing the address, or move the funds when a safer option is available. Is that guidance intended to be standardized, or left to each wallet? Otherwise, different wallets could show very different warnings for the same situation.

-------------------------

duncan0k | 2026-09-11 14:23:53 UTC | #3

Apologies for the slow reply — this got lost behind the mailing-list thread.

This is a distinction I should have drawn in the draft and didn't. I excluded
*risk scoring* on purpose — weighing balance, dormancy and CRQC timelines is
subjective and not chain-observable. But a *recommended action* is different:
each level implies one almost deterministically, and you're right that leaving
it unstated just moves the divergence from "which coins are exposed" to "what
should I do about it".

What I think each level implies, as a floor:

- **EXPOSED_AT_REST** — move to a non-exposed output type as soon as one you
  trust exists; until then, stop adding to it.
- **EXPOSED_ON_SPEND** — nothing needed at rest; when you do spend, sweep the
  whole balance and never send change or new funds back, since the first spend
  makes it AT_REST.
- **NOT_EXPOSED** — nothing.
- **UNDETERMINED** — treat as AT_REST until your software can classify it.

The awkward part is "when a safer option is available": before something like
BIP 360 activates, the only safer option is fresh-key hygiene; after, there's
an actual destination. So the action text can't hardcode a target type.

Proposal for v0.4.0: an informative appendix mapping each level to a
SHOULD-level minimum action, with wallets free to say more but not to
contradict it. That standardizes the floor without dragging the spec into
scoring. In the scanner I run, the four levels already map to exactly four
advice strings, and that's held up in production — so a deterministic mapping
seems workable.

From a wallet's side: is that granularity useful, and would you want the
action as a machine-readable key alongside the level, or is prose enough?

(v0.3.0 is up, addressing the P2TR points raised on the mailing list.)

-------------------------

Anzus_GemWallet | 2026-09-12 07:48:03 UTC | #4

From the user side, clear prose is essential. Having a shared action key behind it also seems useful, so different wallets can translate and present the same basic advice consistently. I would keep the actions simple, with each wallet explaining the reason and urgency in plain language.

-------------------------

duncan0k | 2026-09-14 20:49:44 UTC | #5

Thanks, that settles it. v0.4.0 will carry an informative appendix with a small fixed set of action keys, one per level, and a one-line SHOULD floor for each. Wallets own the wording, the reasons and the urgency on top of that. I'll post here when it's up.

-------------------------

duncan0k | 2026-09-14 21:09:11 UTC | #6

v0.4.0 is up: https://github.com/duncan0k/pubkey-exposure-classification/blob/fb0bc77a58991d43cad1095912e38984d88702fb/bip-output-pubkey-exposure-classification.md#appendix-a-holder-implications-informative

Appendix A gives each level one action key — `MIGRATE`, `SWEEP_WHEN_SPENDING`, `NONE`, `TREAT_AS_MIGRATE` — plus a one-line floor. Wallets own everything above that. The Abstract and Motivation were also rewritten to state the reasoning directly. If the floor wording needs adjusting from the wallet side, say so and I'll fold it in.

-------------------------

murch | 2026-09-16 22:05:44 UTC | #7

Hey @duncan0k, the proposed classification seems trivial to me, so I’m not sure I understand the value-add of a BIP here.

-------------------------

duncan0k | 2026-09-17 06:41:44 UTC | #8

@murch The four levels are trivial, agreed. The edge rules are where the draft spends its words, and each one is a case where shipping software or a careful reader landed on the other answer.

- A reused P2PKH that's been spent from and still holds coins. My own tool reported that as "exposed on spend" for a month. To an attacker it's a P2PK.
- P2TR. conduition read it on the list as unexposed, since a key-path spend never reveals the internal key. The draft calls it exposed at rest, because the attacker only needs the output key that's already on chain. Different questions, and it took a revision before the draft said so.
- P2SH/P2WSH where the script was never revealed. Same for keys that went out off-chain in an xpub. Tools include or exclude both without saying which.

Published figures for exposed supply run from about 25% to over 34%, and that spread comes out of those choices, not measurement. conduition wrote on the list that DropKick and Lifeboat are "both contingent on this source of truth, but nobody has yet spent the cycles to figure out how that would work", and a wallet developer up-thread wants shared action keys so two wallets don't put different warnings on the same output.

None of which is hard. It's just that tools which have to agree with each other have nothing to agree on yet. Is there a case above you'd classify differently from the draft? And if the doubt is venue rather than content, where should a definition like this live so the BIP 360 and BIP 361 discussions can point at it?

-------------------------

murch | 2026-09-17 18:36:56 UTC | #9

If other people agree that this is valuable informational BIP and should be published, the BIPs repository is a fitting venue. If you submit it and people provide feedback to that effect, it will be published.

I still don’t see how “once a hash-based output script has been spent from, all UTXOs sent to that output script are exposed at rest” or “an output script that doesn’t require a preimage for spending is always exposed at rest” are novel discoveries, especially since BIP360 already described vulnerability to long-range and short-range attacks already: https://github.com/bitcoin/bips/blob/55083d36ddebcd2a039135a2f4ee74917a5803d3/bip-0360.mediawiki#user-content-Long_Exposure_vs_Short_Exposure_Attacks

-------------------------

duncan0k | 2026-09-18 05:28:11 UTC | #10

@murch Agreed on both. Neither rule is a discovery, and BIP 360's Long Exposure vs Short Exposure section already lays out the taxonomy. The draft should have cited it and didn't. Next revision maps onto those terms: EXPOSED_AT_REST is vulnerable to long exposure, EXPOSED_ON_SPEND only to short exposure.

Rereading that section also turned up a mistake in the draft. It has P2MR as NOT_EXPOSED. BIP 360 only claims resistance to long exposure; a P2MR spend still puts the leaf key in the mempool, along with the leaf script and Merkle path, so it's the same race as P2WSH until post-quantum leaves exist. By the draft's own fail-closed rule that's EXPOSED_ON_SPEND, which leaves NOT_EXPOSED with no member today. Going into the same revision.

What's left under BIP 360's footnote ("anytime their script reveals a public key") is the operational part, and that's the whole of what the draft adds: what counts as revealed (spent from, per script, across every UTXO paying it; m of n for multisig; a complete spend recipe for script trees), what to report when history is truncated, and vectors two tools can be checked against.

I'll submit it as Informational once that's up. Thanks for the pointer.

-------------------------

duncan0k | 2026-09-18 10:38:45 UTC | #11

@murch Submitted as Informational: https://github.com/bitcoin/bips/pull/2294

The file is at v0.5.1. Since v0.4.0: the first two levels are stated as BIP 360's long exposure and short exposure vulnerabilities per output, with that section cited as the source; P2MR is corrected from NOT_EXPOSED to EXPOSED_ON_SPEND; and the pre-activation case, where a witness v2 output is spendable by anyone, is noted as outside the classification. Reference implementation and 25 vectors are in the same repo as before.

Whether it gets published depends on what people say on the PR, so if you have a view either way, that is where it counts.

-------------------------

duncan0k | 2026-09-22 06:05:22 UTC | #12

@Anzus_GemWallet The action keys in Appendix A came out of your two posts here. If they look right from the wallet side, the PR is where that counts now: https://github.com/bitcoin/bips/pull/2294. Publication depends on feedback there. If they don't look right, saying so there is just as useful.

-------------------------

