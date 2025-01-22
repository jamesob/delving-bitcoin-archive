# Transitory Soft Forks for Consensus Cleanup Forks

JeremyRubin | 2024-12-23 22:53:02 UTC | #1

A while ago, @harding proposed soft forks with a "shelf life" for protocol upgrades, especially for covenants. It sparked some discussion, but was a bit convoluted and not widely adopted.

However, there might be a new angle for that same concept: applying it to consensus cleanups. I do not think that was discussed previously.

Consider someone wanting to impose a new sigops/byte limit in legacy script. That new restriction might end up confiscating funds or producing another major downside. Instead of making it permanent, we could adopt an expiration, say two years, unless explicitly renewed. That approach protects us from immediate vulnerabilities without locking us into a single solution in case we discover a better fix later. We could also allow the soft fork to be undone by signaling to disable its enforcement if we decide it is not worthwhile after all.

The biggest drawback is that scripts with timelocks (like Lightning channels) might still get partly confiscated under these rules. E.g., if the initial satisfied condition is made to be invalid, and a recovery path becomes valid when they age-in. That risk does not disappear, but perhaps the ability to revisit the cleanup is still worth having.

One key distinction between cleanup and feature forks is that feature forks introduce new capabilities that users might rely on indefinitely, so having them expire or require periodic reactivation would create uncertainty and complicate adoption. In contrast, consensus cleanups are narrower in scope. They aim to mitigate known risks or remove unnecessary cruft from the system, which often can be reversed or updated without causing long-term disruption. Since cleanups do not define new functionality that users depend on, but rather provide temporary mitigations or incremental improvements for behaviors users are not relying on, they are better suited to a time-limited or renewable enforcement policy.

I may be missing some corner cases here, so I'd be interested to hear others' thoughts on whether this approach seems feasible or if there are important reasons to avoid it.

-------------------------

harding | 2024-12-24 17:47:34 UTC | #2

[quote="JeremyRubin, post:1, topic:1333"]
I’d be interested to hear others’ thoughts on whether this approach seems feasible or if there are important reasons to avoid it.
[/quote]

Technically, I think it's feasible and I can't think of a reason we must avoid it.  I also think it might be useful to have modern experience with transitory soft forks in case we ever again have a [BIP50](https://github.com/bitcoin/bips/blob/665712c727ee898f0e6a31eee6f1a0ecab8bae4e/bip-0050.mediawiki)-like situation where a short-duration soft fork may be useful for mitigating an emergent risk.

Less technically, I got the impression before that the proposal was pretty "meh" even if I had been able to fully address all of the criticism.  Quite reasonably, several champions of soft fork proposals weren't enthusiastic about going through the effort of development, review/bikeshedding, and community consensus building for something that might, a few years later, involve additional development, review/bikeshedding, and community consensus building.

I think that concern also applies to cleanup soft forks.  What developer working on them wants to run the obstacle-course marathon of getting them to activation only to maybe have to run it again in a few years _for no additional benefit?_

-------------------------

JeremyRubin | 2024-12-24 20:33:01 UTC | #3

[quote="harding, post:2, topic:1333"]
I think that concern also applies to cleanup soft forks. What developer working on them wants to run the obstacle-course marathon of getting them to activation only to maybe have to run it again in a few years *for no additional benefit?*
[/quote]

nothin' like good ol' job security?

But also, it makes the stakes far lower on these patches which are an "anti feature" -- that is, no one particularly wants any restrictions on what transactions could be valid beyond what's needed to mitigate DoS attacks. Auto-repeal is nice because if we discover DoS A and mitigate it with patch P, and then later discover DoS B, patching B with some patch Q while P is active might be much more difficult than coming up with P' that covers both A and B.

-------------------------

ariard | 2024-12-29 19:47:59 UTC | #4

For sure, the current (lack of) process of consensus change can be already very frustrating for a dev or group of devs championing any change, being it indifferently to add a feature or do a tech debt cleanup, and leveling up the bar by requiring that new changes to be made permanent by some re-activation after some transitory period could very likely provoke more discouragement.

On the other hand, whatever the amount of work that have been poured in, if a soft fork isn't robust enough, has too many unbalanced trade-offs, do not convince enough community stakeholders of its intrinsic worthiness or there is even a deadlock on its activation, no protocol developer is entitled to have it its way. Even if it's a profound disillusionment, there is indeed nothing like good old job security.

I can see one benefit about transitory soft fork, especially for transactions restrictions consensus changes, namely when the technical rational for which the consensus changes has been motivated for is still under partial embargo for security reasons. E.g for Bitcoin Script-arising vectors of DoS, coming up with an interesting specimen of exploitation can ask a bit of dexterity and know-how. Generally, bitcoin protocol devs are a bit cautious with whom they share such exploitation samples to avoid that it falls in the hand of the first script kiddie...Sadly, the exploitation cost of DoS vulnerabilities are not always something like crazy.

Of course, ideally I think we would like all the consensus changes rationales to be perfectly transparent for the whole community and minimize the "just trust the grey beards gurus” phenomena in the process of community consensus building. However, given that consensus changes take months and years to come to fruition putting all the exploitation samples to justify a consensus change, that might not happen is a risky bet...Transitory soft fork might offer some more flexibility as a consensus change could be now deployed and activated, the whole technical rational revealed, and then the consensus changes re-activated or definitively locked-in.

I think too that auto-repeal could be fruitful to encompass many DoS vectors, discovered at different points in time, under a single newer mitigation. Though I'm also doubtful that it won't lean to forever bikeshedding on the new mitigation, where the newer mitigation, while cleaner code-wise due to being more generic does not increase full-node robustness qualitatively or quantitatively. I believe we're better off to favor more some old good UNIX hacking approach, where cleanup consensus change are versioned, modular and easy to be extend or build on to encompass newer class of DoS with time.

Overall, I also think we would be better to package each cleanup consensus change in its own activation sequence (i.e BIP9 nversion bit), rather than trying to deploy a whole set of them at once (how far are we from timewarp fix readyness ?), making it easier to put some transitory or multi-stage activation rules, if we think it's worth it.

-------------------------

harding | 2024-12-30 19:22:07 UTC | #5

[quote="JeremyRubin, post:3, topic:1333"]
it makes the stakes far lower on these patches which are an “anti feature”
[/quote]

The main way we lower the risk today is by applying the potentially confiscatory consensus rule first to relay/mining policy.  If nobody complains and there's no history of rule-violating transactions mined for a long period of time, then confiscation risk is minimal (but not zero).  I think that means the benefit of a transitory soft fork for cleanup at the transaction layer is marginal, bringing an already minimal risk a little closer to zero.

Feature forks are different.  Almost any feature that can be added can alternatively be emulated by a trusted third party who signs according to the feature's proposed rules.  This can be implemented directly on Bitcoin or through a layer of indirection (e.g. a sidechain).  However, I'm unaware of any widespread use of this emulation capability, probably because people who are comfortable trusting others are fine keeping their money on exchanges and don't need new features; those who aren't comfortable with third-party trust are only interested in new features that are guaranteed trustless.

For that reason, I think there's a strong case to use transitory soft forks for proposed new features that meet the technical bar and have strong community support but are opposed by some for reasons that can't be proved at present (e.g., that the feature may not see significant use, that an alternative may be preferred, that something better may come along later, or that they may disrupt subtle economic incentives).  A transitory soft fork allows trustless use without a permanent commitment.

However, I suggested that approach as a compromise and I haven't seen anyone---neither proposers nor detractors/equivocators---say that they're willing to accept any current proposal being added in a transitory soft fork.  I've spent the almost three years since making the transitory proposal wondering if I'm the only person whose concerns about CTV and other proposals would be addressed by a trial run.

If neither proposers nor detractors/equivocators of feature soft forks are willing to compromise, probably because neither side wants to sign up to relitigate the case a few years in the future (even though we may have better data then), I think it's asking a lot for proposers and detractors/equivocators of cleanup soft forks to sign up for potential relitigation, especially if the main benefits are a marginal reduction in confiscation risk and potentially simplifying future cleanup code.  If we are considering transitory soft forks, I think it makes the most sense to use them for all initial soft forks---both cleanups and new features---or to not use them at all.

-------------------------

JeremyRubin | 2025-01-02 01:05:17 UTC | #6

I agree that mining & relay policy, and a lack of conflicts against that is a good metric (though trivially DoSable by a bad actor who wants to mine one weird tx). 

Noting that while features like CTV or CAT can be emulated with a signer, block level introspections like transaction sponsors or must-use outputs cannot be emulated using signing federations.

On the whole, I just don't expect hat an opcode that stops working after a delay would be convincing for people to use more than a token amount of funds for... might work well for things like lightning channels where expecting to close w/in 3 years is reasonable, less good for vaults.

-------------------------

1440000bytes | 2025-01-02 14:08:06 UTC | #7

[quote="harding, post:5, topic:1333"]
I’ve spent the almost three years since making the transitory proposal wondering if I’m the only person whose concerns about CTV and other proposals would be addressed by a trial run.
[/quote]

What are these concerns? IIRC, it has something to do with the idea that nobody would need CTV anymore if other ways to implement covenants are introduced. However, it's not really a concern if we are moving in the TXHASH direction and plan to upgrade CTV later.

-------------------------

ajtowns | 2025-01-03 10:17:02 UTC | #8

[quote="harding, post:5, topic:1333"]
Feature forks are different. Almost any feature that can be added can alternatively be emulated by a trusted third party who signs according to the feature’s proposed rules. This can be implemented directly on Bitcoin or through a layer of indirection (e.g. a sidechain). However, I’m unaware of any widespread use of this emulation capability,
[/quote]

Isn't "covenant-less Ark" [[0]](https://ark-protocol.org/intro/clark/index.html) [[1]](https://arkdev.info/blog/ark-release-v0.2#covenant-less-ark) an example of this approach?

-------------------------

harding | 2025-01-03 21:22:38 UTC | #9

[quote="ajtowns, post:8, topic:1333"]
Isn’t “covenant-less Ark” [[0]](https://ark-protocol.org/intro/clark/index.html) [[1]](https://arkdev.info/blog/ark-release-v0.2#covenant-less-ark) an example of this approach?
[/quote]

Nifty!  It certainly meets most of my definition.  I like the idea of a pseudo-covenant that only requires a single successful interaction to create and is secure if you find a single honest third-party.  clArk is efficient about that because it already has a bunch of third parties in a multisignature who are ready to sign, but it doesn't add much cost to generalize it:

1. Lots of people run signing oracles.  For example, a setting which allows every relaying full node to be a signer.
2. Alice wants to pay a precommitted transaction tree paying Bob and Carol, like would be possible with CTV.  She asks Bob and Carol for a list of oracles and asks each oracle for an ephemeral pubkey (for which each oracle attests ownership).
3. Alice aggregates the pubkeys and creates (but does not sign) a transaction paying that aggregate pubkey in a P2TR output.  She gives the PSBT to the oracles.
4. The oracles create a 1-input, 1-output keypath spend from that PSBT that pays to the transaction tree and gives all the presigned transactions to Alice.  
5. Alice signs, broadcasts, and gets confirmed her PSBT and gives the presigned transactions to Bob and Carol along with the attestations.  As long as at least one oracle signed honestly and destroyed their ephemeral privkeys, the pseudo-covenant is secure.

I think the overhead compared to base CTV is 111 vbytes for the 1-in-1-out transaction plus 18 vbytes (a witness-data signature plus overhead) for each precommitted transaction.  Obviously it doesn't give you the cheap optionality of CTV in tapleaves, but it seems like an adequate solution for many CTV use cases like congestion control, channel factories, and coin pools.

-------------------------

harding | 2025-01-03 22:26:00 UTC | #10

[quote="JeremyRubin, post:6, topic:1333"]
I just don’t expect hat an opcode that stops working after a delay would be convincing for people to use more than a token amount of funds for… might work well for things like lightning channels where expecting to close w/in 3 years is reasonable, less good for vaults.
[/quote]

Yeah, this was [discussed last time](https://gnusha.org/pi/bitcoindev/c62c913ac4e36182f719ddb754a0f754@dtrt.org/).  I disagree.  Every year, dozens of new Bitcoin-related products are launched with the hope of attracting enough users to support continued development, and dozens of other Bitcoin-related products from previous years are discontinued because they failed to attract enough paying/donating users.  The risk of a new thing failing to gain traction and becoming unsupported is an ordinary risk in this industry (and probably every innovating industry).

This applies to existing long-term Bitcoin security products.  Anyone who uses Casa, Unchained, Liana, or other self-custody-with-backup-signers multisig products must prepare for the possibility that parts of the product suddenly become unavailable and that the remaining parts fail over time, necessitating a funds movement.  A transitory soft fork is similar, except that it's extremely unlikely to become unavailable prematurely.

-------------------------

1440000bytes | 2025-01-04 00:07:12 UTC | #11

[quote="harding, post:10, topic:1333"]
Every year, dozens of new Bitcoin-related products are launched with the hope of attracting enough users to support continued development, and dozens of other Bitcoin-related products from previous years are discontinued because they failed to attract enough paying/donating users.
[/quote]

Opcodes are not products though. Is OP_CLTV a product?

-------------------------

