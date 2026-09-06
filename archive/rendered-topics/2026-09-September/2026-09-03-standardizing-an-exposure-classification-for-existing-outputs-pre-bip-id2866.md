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

