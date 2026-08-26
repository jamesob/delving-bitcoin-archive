# A design bar for coercion-resistant custody, and where duressfeatures land against it

yurisvb | 2026-08-24 14:07:39 UTC | #1

I've deposited a threat model for physical-coercion ("wrench") attacks on
self-custody: https://doi.org/10.5281/zenodo.22018892 — preprint, CC BY, not
peer reviewed.

The contribution is a **criterion**, not a construction, so I'll state it first
and let the classification follow.

### The impossibility

You cannot prove you have forgotten. Knowledge is provable — sign a challenge.
Its absence is not: no holder can demonstrate that no memorized copy, backup or
resumable derivation survives within their reach.

### What follows

Under coercion the attacker is not deciding whether to break the cryptography.
He is deciding whether a released holder can still reach the asset before he
does. Where a design leaves him a path that is feasible but *slower* than the
holder's, the holder is the only obstacle — and by the impossibility above, no
assurance to the contrary can be produced. The expected gain from removing the
holder is then positive whenever the stake is large enough relative to the cost
of doing so.

**The No-Gray-Area Criterion:** a custody design should leave the attacker
either no feasible path at all, or a decisive one. Never a slower one.

### Two routes clear it — and one of them already ships

Either remove the race, or put the winning racer beyond the attacker's reach.

**The second route ships today.** Time-lock-rescue vaults and collaborative
custody with a genuinely remote co-signer clear the bar, and that route is not
mine. Its cost is stated openly rather than hidden: custody is no longer
individual. I want to be clear that this is a real trade-off and not a defect,
because the criterion is not an argument against delegated designs — it is what
tells you when one is doing its job.

The condition that matters is *reachability*, not delay length. A co-signer in
the same house during the encounter is not a second racer; they are a second
victim.

### Where it bites

Two classes do not clear it.

**A reachable racer.** Any design where the cancel, freeze or rescue authority
is on the scene — the holder's own alert device, a co-located friend — leaves
the attacker a slower path and the holder pivotal. Deterrence claims of the form
"publicly verifiable, so kidnapping is pointless" are about whether the attack
occurs, not about the game once it has.

**Erasure.** A wallet-erase PIN removes the device's copy instantly and
covertly, which is a genuine strength no delay architecture has. But the vendor
documentation that markets it for physical threat also instructs the holder to
keep a recovery backup — so what the erasure deletes is the path that required
hurting nobody, while the path that runs through the holder remains. It needs
the attacker to be ignorant of the feature *and* not to reason about backups;
succeeding at the first still leaves "the device is broken," which does not
entail "there is nothing to get."

### Empirical Section

The empirical section codes a public registry of 352 physical-attack incidents
(snapshot 2026-08-14). Scripts, regexes and caveats are in the repository. The
figures are directional, media-sampled and survivorship-biased, the fatality
share is a floor rather than an estimate, and I would rather you check the
coding than take the numbers.

### On publishing rather than withholding

These designs are shipped, documented and marketed. An attacker who reads a
vendor help page already knows what they do. Withholding the analysis protects
nobody and leaves holders unable to evaluate what they are relying on.

-------------------------

Anzus_GemWallet | 2026-08-26 02:24:22 UTC | #2

This is a useful reminder that different physical-risk situations need different plans.

It may help to include a simple “who is this for?” section: what ordinary self-custody users should do, and when someone has enough risk or value to consider a remote co-signer. Without that, some readers may come away thinking that self-custody is unsafe for everyone.

-------------------------

