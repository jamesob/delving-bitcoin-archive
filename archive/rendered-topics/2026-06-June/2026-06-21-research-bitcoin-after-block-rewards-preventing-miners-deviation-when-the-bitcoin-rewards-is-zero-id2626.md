# [Research] Bitcoin After Block Rewards : preventing miner's deviation when the Bitcoin rewards is zero

xodn348 | 2026-06-21 05:02:06 UTC | #1

Howdy.

I have written paper about Bitcoin;s security when the Bitcoin block rewards approach zero near 2140. 

**The original motivation of this paper is worrying about reducing block rewards and miner's profitability because of halving, while the price could not doubling in every four years.** 

**Problem :** 

How the Bitcoin's security changes when the Bitcoin rewards is 0 and how could we prevent the miner's deviation.

**Findings :** 

1\. Found G_t threshold which could determine whether miner's deviation is happening or not

2\. The massive deviation is not happening only because of block rewards

3\. In a pure fee-only regime, deviation can become rational even at 0.17% of transaction fees

4\. In a pure fee-only regime, base fee, fee floor and adaptive block size could help to alleviate massive deviation of miners.

My suggestion would not be a only solution fot the problem but I would be appreciate if this incur the other brilliant scientists' inspiration.

The link is in below.

 https://arxiv.org/abs/2606.05503 

Thank you.

Sincerely,

Junhyuk Lee

-------------------------

show1225 | 2026-06-23 15:19:14 UTC | #2

Dear Junhyuk Lee,

Provoked by your post, I summarized my thoughts on a separate [post](https://delvingbitcoin.org/t/addressing-the-diminishing-block-subsidy/2640).
While floor fee seems to be worth trying (as it is soft-forkable), an ultimate solution could be tail emission + EIP-1559 style fee burn and priority tipping in my opinion.

SHO

-------------------------

xodn348 | 2026-06-28 15:42:20 UTC | #3

That is cool. I hope there will be more discussion about this.

-------------------------

da2ce7 | 2026-07-14 22:30:31 UTC | #4

# A Reply to "Bitcoin After Block Rewards" (Lee, 2026)

**Re:** *Bitcoin After Block Rewards*, J. Lee, Texas A&M University (arXiv:2606.05503v1)

---

## 1. Introduction and Summary of the Paper's Contribution

The paper addresses one of the most consequential open questions in the study of permissionless consensus: whether the Bitcoin network can remain secure once the block subsidy has declined to zero and miner compensation derives solely from transaction fees. The author formalizes the miner's per-block choice between honest and deviating behavior, derives the deviation threshold condition G_t ≥ φ(w) · X_t, and shows that the withholding penalty factor φ(w) = e^(λw) − 1 is independent of an individual miner's hash-rate share, depending only on the orphan-risk differential induced by withholding. The empirical analysis of the 100,001-block window surrounding the 2024 halving is careful and welcome: it documents a substantial contraction in observable rewards, fees, and on-chain demand, yet finds no persistent increase in deviation-consistent behavior as proxied by block audit scores. The simulation results are sobering — in a fee-only regime, an additional private gain of merely 0.17% of X_t suffices to make deviation attractive across all hash-rate groups, and absent intervention the deviation rate exceeds 70% of blocks, well beyond the Backbone-style honest-majority requirement. The proposed remedies — an EIP-1559-inspired base fee, a fee floor, and an adaptive maximum block size — are evaluated systematically, and the finding that the base fee is the dominant stabilizing factor is a useful and insightful result.

These are genuine contributions. The threshold formulation is simple enough to serve as a practical benchmark, the empirical window is well chosen, and the policy evaluation is disciplined. This reply is offered in a constructive spirit: the concern is not with what the paper does, but with what its modeling assumptions exclude — and with the possibility that the excluded dynamics point toward a different, and complementary, class of solutions.

## 2. The Central Limitation: Hash Power Is Modeled as Static

The model fixes each miner's hash-rate share h_i and stipulates that deviation "does not alter a miner's computational capability." The only strategic variable is propagation behavior. There is consequently no entry or exit margin, no capacity that responds to price or threat conditions, and no distinction between hashrate that is *deployed* and hashrate that is *available*. Security is assessed statically: the fraction of deviating blocks must remain below one half.

This assumption is defensible for the paper's stated purpose — isolating the incentive effect of a shrinking reward on an otherwise unchanged miner population — but it forecloses precisely the dynamics that are most likely to characterize a low-subsidy environment. When the security budget flowing through the protocol becomes thin, there is no reason to expect the supply of hashpower to remain a constant, always-on quantity. Rather, one should expect mining to become *dynamic and stratified*:

**(a) A minimal-cost base load.** A steady-state stratum of hashrate sized to the fee revenue actually available — operating on marginal or stranded energy, covering ordinary block production, and setting the visible entry price for any attack.

**(b) A dark, elastic surge capacity of unknown size.** Reserve capacity — whether warehoused hardware, standby contracts with operating facilities, or committed rental capacity — held by parties with a stake in Bitcoin's stable operation (exchanges, custodians, ETF issuers, large holders), and activated only upon detection of an attack such as an attempted double spend. Critically, this capacity is *deliberately opaque*: the defense reveals only the resources sufficient to defeat a given attack, never its full extent.

The strategic logic is that of deterrence through ambiguity. An attacker facing defensive capacity of unknown magnitude cannot compute the expected cost of an attack; the visible base load no longer bounds the attacker's required expenditure. This directly severs the link — central to the paper's pessimistic fee-only results — between the observable reward X_t and the cost of subverting the chain. In the paper's framework, a collapse in X_t mechanically collapses the deviation threshold; in a stratified-security model, the threshold relevant to a serious attacker is set by capacity the attacker cannot observe. The funding logic is likewise sound in principle: the parties who internalize the largest losses from a successful reorg are the natural purchasers of the security that fee-paying transactors underprovide — a Coasean resolution of the security-budget externality that need not flow through the protocol at all.

The paper's exclusive focus on protocol-side revenue engineering (base fee, fee floor, block size) therefore addresses only one of two available levers. Its mechanisms make the *visible base load* economically self-sustaining, which is valuable — it sets the price of admission for attacks and reduces how often the reserve layer is probed. But it leaves unexamined the elastic, demand-driven, stakeholder-funded stratum of security that a thin fee regime would plausibly call into existence. The two approaches are complements, not rivals, and a complete treatment of post-subsidy security should model both.

It must be acknowledged that the stratified model carries its own difficulties, which any formalization would need to confront: idle ASIC capital depreciates whether or not it runs, creating pressure to deploy or sell reserve capacity; difficulty adjustment calibrated to a thin base load lowers the attacker's entry cost along with the defender's; each exercised defense leaks a lower bound on reserve size, inviting probing attacks; the sponsor coalition faces a free-rider problem, mitigated but not eliminated by its small size and repeated interaction; and deterrence through cost-uncertainty does not bind an attacker indifferent to profit. These are open research questions — but they are questions the field should be asking, and the paper's static framing prevents it from asking them.

## 3. Witnessing as the Precondition of Elastic Defense

It is essential to be precise about the logical relationship between the mechanisms introduced above, because they do not stand in parallel: they form a strict dependency chain, and the chain's first link is epistemic, not economic.

A reactive defense is *conditional on detection*. Dark capacity of unknown size deters nothing if it never learns that an attack is underway; the defenders can only surge to protect a chain they know the victim is trusting. Detection, in turn, is supplied by witnessing — the victim's counterparties and the broader network observing, in real time, the blocks on which settlement is being predicated. It follows that an eclipse or Sybil attack does not need to contend with the elastic reserve at all: it *routes around it*, by ensuring the attack is never seen. An attacker who isolates a victim's network view can feed it a cheap privately mined chain, and from the perspective of the honest network nothing anomalous is occurring — no conflicting history is visible, no trigger fires, no surge is mounted. The attack defeats both security strata simultaneously: the thin base-load work is inexpensive to outpace in private, and the reserve capacity — the entire deterrent — is blind.

This is why the eclipse attack, a bounded nuisance in the subsidy era, becomes the *dominant attack vector* in the late game. Under a large subsidy, work is so expensive that an eclipsed victim can be defrauded only of amounts small relative to the cost of the private chain; eclipse resistance can therefore live quietly in peer-selection heuristics, invisible to the trust calculation. In a fee-only regime with a minimal base load, the cost of fabricating a plausible-looking chain collapses along with X_t — this is the paper's own central result, viewed from the network layer — while the cost of an eclipse attack, which is a network-position attack rather than a hashpower attack, does not scale with the security budget at all. The weaker the blocks, the more damaging the eclipse.

A reactive defense is furthermore only as good as its response window, and here the paper's implicit settlement model — fast, fixed-depth finality — works against any dynamic scheme. Two mechanisms close the gap, and both must be understood as serving the witnessing requirement identified above.

**Value-scaled confirmation depth.** Confirmation requirements should scale with the value at risk, so that the attacker's cost grows with depth while the prize remains fixed — and, equally important, so that the detection-and-response window widens for exactly the transactions worth attacking. This practice already exists informally at every major exchange; formalizing it acknowledges that base-layer Bitcoin is evolving into a settlement layer for large, patient transactions, with speed provided by higher layers.

**Witnessed propagation as a validity condition.** The more consequential proposal is a requirement that blocks be demonstrably relayed to the network — witnessed by a threshold of independent relays at approximately the time of mining — before they are accepted, i.e., a resubmission or attestation requirement mediated by a relay network. The effect on the paper's own threat model is decisive: a privately mined chain, however much work it embodies, is invalid because it was never witnessed. Withholding-based double spends, selfish mining, and fee sniping are not merely re-priced; they are removed from the strategy space. In the paper's notation, this does not raise φ(w) — it renders the withholding strategy inadmissible, forcing any attack into the open where a reactive defense can observe and answer it block by block.

The costs of this mechanism should be stated with equal candor. A relay attestation requirement reintroduces a trusted component into consensus: relay operators can censor, and observation-based validity is inherently subjective — two honest nodes with different network views may permanently disagree, and newly syncing nodes must trust a record of what was witnessed. These costs can be distributed (threshold attestation across a jurisdictionally diverse federation) but not eliminated; this is the known, fundamental price of trading reorg-ability for finality in a permissionless setting. A softer deployment — in which economically significant nodes and the sponsor coalition adopt witnessed-propagation as *policy*, refusing to credit unwitnessed blocks or honor deep reorgs of witnessed history, while consensus itself remains objective — captures most of the security benefit while confining trust to the settlement endpoints where it already resides.

## 4. The Underlying Reframing: Confirmation as Two Conjunctive Conditions

Taken together, these mechanisms amount to a reformulation of what finality in a post-subsidy Bitcoin actually consists of, and it is worth making the reformulation explicit, because the paper's framework implicitly assumes the original, fused model.

The central claim is that confirmation is irreducibly *two* things — not accumulated work with an optional sanity check, but two co-equal, conjunctive conditions, the failure of either of which voids finality:

1. **The cost of the blocks.** Accumulated proof-of-work: evidence that the history was expensive to produce and would be expensive to rewrite. This is the economic component, and it is the quantity the paper's threshold condition governs.
2. **Knowledge that the network is aware of the blocks.** Verified common knowledge: assurance that the history one is trusting is the history one's counterparties, the relay network, and the elastic defense are also watching. This is the epistemic component, and it is what arms the surge capacity of Section 2.

In the subsidy era these two conditions travel together so reliably that the second is invisible: work is expensive enough that any sufficiently heavy chain one observes is almost certainly the public chain. In the fee-only era that coupling breaks. Chain weight becomes cheap to fabricate, and a heavy-looking chain shown to an isolated observer proves nothing. The second condition must then be established *independently* — and it cannot be established from chain data, because it is not a property of the chain. It is a property of the observer's position in the network.

Proof-of-work answers one question: *was this history expensive to produce?* It is a costliness proof, and its assurance is forward-looking and economic — a bet that the work embodied in the chain will remain valuable enough that rewriting it stays irrational. The paper itself is, at bottom, an analysis of the conditions under which that bet stops paying. But work alone has never identified *which* expensive history is authoritative; it identifies only the most expensive history a given observer has seen — which is precisely why eclipse attacks succeed, and why the community's actual crisis resolutions (the 2010 overflow incident, the 2013 fork) were social decisions that work subsequently ratified.

The second question — *is this history common knowledge?* — is answered not by work but by witnessing: verification that the network at large, including the parties one settles with, has seen the blocks in which one is placing trust. A double spend is, structurally, an attack on common knowledge; a witnessing requirement makes the attack impossible rather than expensive.

The proposal, then, is a deliberate re-weighting. In the subsidy era, work supplies effectively all of finality and the social layer is an unspoken backstop. In the post-subsidy era, work should be understood as the *price signal* — the entry fee that keeps attacks costly and disciplines miners at the margin, exactly the quantity the paper's threshold condition governs — while witnessed common knowledge becomes the *finality mechanism* for transactions of consequence. Settlement assurance decomposes into (i) trust in the future relative value of the work securing the chain, and (ii) verification that one's counterparties and the broader network agree on that chain. Each component is then provisioned by the party best suited to supply it: miners sell costliness; the stakeholder and relay network provides common knowledge.

### 4.1 The New Security Assumption, Stated Negatively

This yields what is proposed here as the explicit, late-game security assumption of a post-subsidy Bitcoin: **a client is secure only if it is well connected to the Bitcoin network — and security fails if it is eclipsed.** The assumption is deliberately stated in its negative form, because that is how the client must operationalize it. Nakamoto consensus treats connectivity as a *liveness* assumption: a partitioned node falls behind but never accepts a false history it could not detect upon reconnection, because work is too costly to counterfeit. The model proposed here necessarily promotes connectivity to a *safety* assumption: an eclipsed client in a low-work regime can be fed a finalized-looking false history at low cost, so well-connectedness becomes load-bearing for correctness, not merely for freshness.

The corresponding design requirement is that the client must *model this assumption in its trust calculation* rather than rely on it silently. Conceptually, the trust a client assigns to a block becomes a conjunction of the form

> trust(block) = f(accumulated work) × P(client is well connected and not eclipsed),

with the essential property that no quantity of observed work compensates for a connectivity estimate near zero. A client unable to establish that it is genuinely embedded in the honest network should decline to finalize *regardless of the weight of the chain it observes*. Operationally, the connectivity term must be actively estimated, not assumed: diversity of peers across autonomous systems, network paths, and jurisdictions; out-of-band comparison of chain tips against independent relays and counterparties; attestation from the witnessing federation of Section 3; and alarm conditions when the client's view diverges from, or cannot be corroborated by, independent channels. Eclipse resistance thereby moves from an implementation detail of peer selection to a first-class, quantified input to settlement — which is precisely where it must sit once the work term alone can no longer carry finality.

The new critical surface of this model must also be named: the security of the witnessing layer is exactly the security of the witnessing graph. If witness selection remains open, plural, and jurisdictionally diverse, corrupting common knowledge requires corrupting a visible plurality of the ecosystem — arguably harder and more detectable than renting hashpower. If it drifts toward a handful of default providers, a trusted third party has been rebuilt by habit. Resisting that drift is a design and governance problem in its own right — and under the negative assumption above, it is now a *safety-critical* one.

## 5. Recommendations

In light of the above, the following extensions are respectfully suggested, either for the present line of work or for future research building on it:

1. **Endogenize hashrate.** Extend the model to allow entry, exit, and latent capacity, so that the security level responds to the fee regime rather than being fixed by assumption. The deviation threshold should be re-derived in a setting where the attacker's required expenditure is bounded by unobservable reserve capacity rather than by X_t.
2. **Model the attacker–defender game.** The paper's threat model — individually rational per-block deviation — should be complemented by an explicit reorg/double-spend adversary and a defender reaction function, including detection lags, activation costs, and the information revealed by each defended attack.
3. **Treat confirmation depth as a policy variable.** The interaction between value-scaled settlement depth, the detection window, and the deviation threshold is analytically tractable within the paper's own Poisson framework and would materially strengthen the policy section.
4. **Evaluate witnessed-propagation regimes.** Both the consensus-rule variant and the settlement-policy variant of relay attestation deserve formal treatment, including their failure modes (censorship, subjectivity, relay availability attacks) alongside their elimination of the withholding strategy space.
5. **Model the eclipse adversary as the binding constraint.** In the low-work regime, the cost-minimizing attack is plausibly not hashpower acquisition but network isolation of the victim, which blinds the witnessing layer and thereby disarms any reactive defense. The threshold analysis should be extended with an adversary who chooses between out-working the base load and eclipsing the target, with the attack cost taken as the minimum of the two.
6. **Specify connectivity-aware clients.** Formalize the client-side trust calculation in which finality requires both sufficient work and a verified, quantified estimate of the client's own connectedness — including concrete estimators (peer diversity across autonomous systems and jurisdictions, out-of-band tip comparison, witness attestation) and the refusal rule under which no observed chain weight is accepted when the connectivity estimate falls below threshold.
7. **Analyze the hybrid.** The most promising architecture is plausibly the combination: the paper's base fee and fee floor sustain the visible base load and set the attack entry price, while stakeholder-funded elastic capacity, deep confirmations for high-value settlement, witnessed propagation, and connectivity-aware clients handle the tail risk. No existing work formalizes this combination; the paper's framework is well positioned to do so.

## 6. Conclusion

The paper convincingly establishes that a fee-only Bitcoin, secured by a static population of always-on miners and unmodified consensus rules, is fragile — attackably so, at private gains of a fraction of a percent of block value. The reply offered here accepts that diagnosis and questions the implicit conclusion that the remedy must be protocol-side revenue engineering alone. If the security budget becomes thin, security itself will not remain static: it will stratify into a cheap base load and an elastic, deliberately opaque reserve funded by those with the most to lose; settlement expectations will lengthen in proportion to value at risk; and the network will come to rely explicitly on what it has always relied on implicitly — common knowledge of which history is real, verified among the parties who matter.

The core of the proposal is the recognition that these elements form a single dependency chain whose first link is epistemic: the elastic defense only functions if attacks are seen, attacks are only seen if blocks are witnessed, and witnessing is only meaningful to a client that can verify it has not been eclipsed. Confirmation therefore becomes two conjunctive conditions — the cost of the blocks, and knowledge that the network is aware of the blocks — and the late-game security assumption must be stated in its negative form: security fails if the client is eclipsed, and the client must model that possibility explicitly in deciding whether to trust what it sees. Work, in that world, is the price of admission; witnessed common knowledge is the finality; and verified connectivity is the precondition of both. The author's threshold framework is an excellent instrument for pricing the first component. The invitation of this reply is to extend it to the other two.

*Edited to make sure the core connection between the Sybil attack and the Dynamic Mining is more clear.*

---

*Thank you J. Lee for publishing your preprint on arXiv. It is good to have proper analysis of the dynamics of the end-game system. The above is my thoughts on the matter: elaborated and little with the help of the AI to make it more accessible.*

*Good Regards,*

*Cameron.*

-------------------------

xodn348 | 2026-07-14 15:48:08 UTC | #5

Howdy, Cameron.

Thank you for your opinion on my paper. By the way, the feedback is quite too long and verbose because of AI. It would be convenient for me to read if the feedback comment. The comment looks like an plain AI's suggestion.

-------------------------

da2ce7 | 2026-07-14 22:31:14 UTC | #6

Hello Junhyuk,

Well let’s me rewrite it in bullet points. (Of course I did read the AI output, and did comprehend every word of it, so alas.)

* Your paper makes fundamental assumptions that are not realistic of end-game game theory:
* You assume a static model of mining. But this maximises cost and minimises security.
* You assume that the secure is only from the difficulty of creating blocks. But this ignores the low cost of Sybil attack when the blocks are cheap to make.

My recommendations are:

1. Use a model where the stake of the system is a also a contributing factor to keeping it secure.
2. Move to a dynamic model of mining capacity, with dark capacity that is used only as much and when is needed.
3. Take into account that blocks will need to have two qualities to be considered confirmed in the late game:
   a. Costly to make.
   b  Known by your friends (i.e. do not trust blocks if you are cut off by Sybil).

Kind Regards,
Cameron.

-------------------------

da2ce7 | 2026-07-14 23:02:13 UTC | #7

Hello again Junhyuk,

I just wanted to say that you are probably finding out the problem space is much larger than you expected.

Even my AI made a slob of the argument. I got it to correct it on second reading. (Now updated above).

Your paper as-it-is is still good, with some re-scoping.

1. Change the tile. It massively over claims.
2. Qualify the assumptions: say this is a worst case pessimistic model.
3. Reduce the confidence in the recommendations: as this is a worst case analysis: reality may be more favourable!

And if you do want to do the serious work to make a paper I would defend. Then you are welcome to work through my critique; and post results here.

Also, best to put an AI text into AI for summary if you do not wish to read it. - extract the essence and then reply to it. - A generic AI response should state why: „everything critiqued was noted in the limitations, and or, is obvious and uninteresting”.

Time is expensive, and you cannot be expected to respond to ever AI slop. - But at least engage with the arguments, (or lack thereof).

Cam.

-------------------------

