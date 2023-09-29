# Covenant tools softfork

jamesob | 2023-09-28 18:38:38 UTC | #1

I'd like to propose a softfork deployment that activates the consensus changes outlined in

- [BIP-118](https://github.com/bitcoin/bips/blob/7004ad1a825a0422b78bbf1a96bf748d5e380569/bip-0118.mediawiki) (`SIGHASH_ANYPREVOUT` for Taproot Scripts)
- [BIP-119](https://github.com/bitcoin/bips/blob/7004ad1a825a0422b78bbf1a96bf748d5e380569/bip-0119.mediawiki) (`CHECKTEMPLATEVERIFY`)
- [BIP-345](https://github.com/jamesob/bips/blob/4aae726be9610a675b362e66f539ce0d5f903a5f/bip-0345.mediawiki) (`OP_VAULT`)

These changes make possible a number of use-cases that are broadly beneficial to users of Bitcoin, including

- [vaults](https://bitcoinops.org/en/topics/vaults/) (reactive custodial security),
- [LN-Symmetry](https://bitcoinops.org/en/topics/eltoo/),
- efficient implementations of [DLCs](https://bitcoinops.org/en/topics/discreet-log-contracts/),
- [non-interactive channel openings](https://utxos.org/uses/non-interactive-channels/),
- [congestion control](https://utxos.org/uses/scaling/),
- decentralized mining pools (via [CTV compression in coinbase payouts](https://utxos.org/uses/miningpools/)),
- various [Lightning efficiency improvements](https://twitter.com/roasbeef/status/1692589689939579259),
- using [covenant based timeout-trees](https://bitcoinops.org/en/newsletters/2023/09/27/) to scale Lightning, and more generally enabling [channel factories](https://bitcoinops.org/en/topics/channel-factories/).

We also see that many speculative scaling solutions (e.g. [Ark](https://arkpill.me), [Spacechains](https://gist.github.com/RubenSomsen/c9f0a92493e06b0e29acced61ca9f49a#spacechains)) require locking coins to be spent to a particular set of outputs without any additional authorization (i.e. CTV, or APO's emulation of it).

---

The patches for BIP-118 and BIP-119 have long been stable and well scrutinized; they each haven't changed in some time.

BIP-345 (OP_VAULT) is the newest of the three by a good margin, and originally I was going to exclude it from this proposal. But after a number of discussions, it became clear that BIP-345 may be the most immediately usable of the three in terms of a major use-case: 
- LN-symmetry (the major use of ANYPREVOUT) will require a good amount of time to coordinate deployment, and 
- CTV, while an important primitive, has mostly niche direct uses (DLC efficiency, possible decentralized mining pools, congestion control, ...) outside of BIP-345-style vaults.

Vaults with BIP-345, on the other hand, would be delayed only by the speed of wallet implementers. [Example wallet implementations](https://github.com/jamesob/opvault-demo/) already exist, and the appetite for safer modes of custody is almost universally present among both industrial and individual users.

The implementation for these consensus changes is only about [7,000 lines](https://github.com/bitcoin/bitcoin/compare/master...jamesob:bitcoin:2023-09-covtools-softfork?), and that includes comprehensive testing. The limited nature of these changes relative to the last two softforks that we've had in Bitcoin give me some comfort in proposing the relatively young code for BIP-345 -- with focused effort, the limited line count and tight scope will make it a tractable deployment to review.

I will be opening this branch as a draft PR in the Core repo.

The activation mechanism is currently drafted as the same modified version of BIP-9 [described in BIP-341](https://en.bitcoin.it/wiki/BIP_0341#Deployment), but I am agnostic in this department. There is currently no signaling period specified - we will wait until further indications of consensus before choosing specific dates.

What do you think?

-------------------------

jamesob | 2023-09-28 18:43:36 UTC | #2

The related Bitcoin Core pull request is here: https://github.com/bitcoin/bitcoin/pull/28550

-------------------------

sjors | 2023-09-29 09:46:38 UTC | #3

One of the things I remember from the Taproot days is that it was impossible for most mortals, including myself, to review the entirety of the proposal. But it made sense to combine all its ingredients in a single fork. E.g. adding Schnorr signatures and/or MAST to regular SegWit script would have added complexity, which Taproot neatly avoided.

These three proposal however are much more suitable for independent deployment. In particular I see no reason to make `ANYPREVOUT` dependent on the less mature `OP_VAULT`, but I also don't see why `CTV & OP_VAULT` should be held back by `ANYPREVOUT`.

BIP9 introduced the ability to activate multiple forks in parallel, so I suggest using one bit for each. This doesn't preclude the possibility to _bundle them in the activation phase_ (and of course BIP-345 can't activate if BIP-119 doesn't). Bundling activation can make sense because it takes a lot of effort to convince miners to signal.

I would rather be in a situation where only one of these proposals makes it through the review filter and gets activated, then having them all stuck.

As each of these proposals gets added to Bitcoin Core, we may at some point feel great about all of them. In that case we recommend miners to set flags `ANYPREVOUT & CTV & OP_VAULT`. But maybe for some reason `CTV` isn't getting enough review, but  `ANYPREVOUT ` has been merged, tested and fuzzed to death in the master branch for years. In that case we could wait even longer, or recommend just the `ANYPREVOUT` flag and keep the rest for later (presumably leaving activation params for these entirely unset).

The latter approach does raise the question as to how much unactivated potential softfork code we want in the main codebase. Since even unactivated code can lead to bugs. That bar should be high imo, but I can see some grey area between mosty-done and ready-to-activate.

-------------------------

jamesob | 2023-09-29 13:41:37 UTC | #4

@ariard writes on GitHub:

> If we can have an end-to-end proof-of-concept implementation of each use-case brought as a justification to the proposed soft-forked opcodes. Otherwise it’s quite impossible to provide a sound technical review of the primitives robustness and trade-offs and state what they enable exactly. And it sounds we’re good to repeat the loop of the last 3 or 4 years of covenants discussions.
> 
> As a reminder, just to take the last example of channel factories, most of the folks who have done *real* research on the subject still disagree on the security model and fundamental trade-off of the proposed design of channel factories.
> 
> As one of my technical peer challenged on the [mailing list](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-September/020921.html) few months ago:
> 
> "So I think that means that part of the "evaluation phase" should involve
> implementing real systems on top of the proposed change, so that you
> can demonstrate real value from the change. It's easy to say that
> "CTV can enable vaults" or "CTV can make opening a lightning channel
> non-interactive" -- but it's harder to go from saying something
> is possible to actually making it happen, so, at least to me, it
> seems reasonable to be skeptical of people claiming benefits without
> demonstrating they're achievable in practice.”
> 
> I’m fully sharing this opinion.

It's worth noting that two of the major use-cases I mention in the original post have test implementations:

- Vaults
  -  `opvault-demo`: https://github.com/jamesob/opvault-demo/
  - `simple-ctv-vault`: https://github.com/jamesob/simple-ctv-vault
- LN-symmetry
  - @instagibbs' CLN implementation: https://github.com/instagibbs/lightning/tree/eltoo_support
  - Richard Meyers' test implementation: https://yakshaver.org/2021/07/26/first.html

In my opinion, if all this fork did was enable these two use-cases (especially vaults), it would probably be worthwhile. But we know that indeed isn't the case, and some of the applications e.g. DLC efficiency improvements ([spec](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-January/019808.html)), are  so straightforward that a demo implementation doesn't seem prerequisite to seeing the value of CTV as a primitive.

Another more speculative use that has a demo implementation is spacechains (https://github.com/nbd-wtf/simple-ctv-spacechain).

-------------------------

fiatjaf | 2023-09-29 13:44:59 UTC | #5

There is also this fully working (with rough edges) Spacechain (with the actual parallel blockchain code too and not only the Bitcoin-BMM part) that works on signet: [https://github.com/fiatjaf/soma](https://github.com/fiatjaf/soma) (although there are no nodes online anymore anyone can always start a new chain).

-------------------------

jamesob | 2023-09-29 14:44:58 UTC | #6

> I would rather be in a situation where only one of these proposals makes it through the review filter and gets activated, then having them all stuck.

This is a point worth consideration.

My goal here is not to have an omnibus softfork - I hope it's clear the contents here seem much more limited in scope and depth than either segwit or taproot. But I think we can't ignore the practical realities of the fixed overhead that comes with the activation process, both social and technical. If I've learned anything from the assumeutxo development cycle, it's that slicing changes too granularly, even when in some abstract software-engineering sense seeming like the "clean" thing to do, actually hampers the overall process because extensive review cannot be consolidated on a single package. I think there is an analogue here.

At our current clip, it's looking like we navigate the softfork process about once every four years. That means if we did these efforts independently, we might have many years go by before having usable vaults in Bitcoin and/or either CTV or APO. I don't think this is a good outcome.

Yes, in the abstract miners may be able to elect to signal independently for each proposal, but will they really show that level of involvement while simultaneously having neglected the review process up until the point of deployment? I think they'll likely just continue to default to whatever Core ships with (short of an egregious change).

I was originally going to propose APO and CTV together without vault, but after speaking with a number of involved contributors it became clear that people felt
1. OP_VAULT is a major "realization" of the use of CTV. The criticism among involved contributors a few years ago was that CTV doesn't go "far enough" to fully realize any one use-case, which OP_VAULT solves.
2. Vaults are essential enough to fundamental bitcoin use that the "dead weight loss" of not having them available for another few years is significant.

When I saw the relatively modest size of the patch for all three together (7,000 lines added including tests, of which there probably need to be more), I became convinced that it's worth, as a community, amortizing the painful effort of consensus review and deployment over the three of them.

On a lighter note, there are uses that involve both CTV and APO - @instagibbs notes that his version of ln-symmetry makes use of CTV, or could be made to very easily.

---

It is my hope that people will agree
1. CTV and APO have been extant long enough that there is broad consensus among most that both are safe changes, and they're both easy to review and reason about, and
2. Vaults are a vital enough use-case that taking the effort to focus review for a few months is a good priority for those interested in matters of consensus.

I think having all three together in the same package attracts many with varying interests and motivations. This seems important in summoning the "activation energy" necessary to perform a thorough review process and navigate the hazards of a consensus change.

-------------------------

ursuscamp | 2023-09-29 15:37:15 UTC | #7

What if omnibus soft forks are the only thing has any hope of running the gauntlet anymore? Changing Bitcoin is becoming more political as the tent gets larger. It is only natural. In order to get enough people on board for a change, you have to give something to everyone.

-------------------------

sjors | 2023-09-29 18:08:28 UTC | #8

> Yes, in the abstract miners may be able to elect to signal independently for each proposal

I think we should strongly suggest one set of proposals (bits) to signal for. There would be a Bitcoin Core point release with certain bits turned on, which miners (or some advocating an alternative binary) should only tweak if they have a good reason. What I'm arguing for is that we decide that set when the proposals are merged and ready to go, rather than upfront.


The technical case for combining `CTV` with `OP_VAULT` seems stronger than the argument for combining `APO` with `CTV`. The narrow use case of `OP_VAULT` gives direction to `CTV`, in a similar way ln-symmetry gives direction to `APO`.

Lines of code is one metric, and easy to measure, but with a soft-fork it's important to think through all the ways it can be used, its worst case resource consumption, etc. For that you probably want to look at a lot more stuff than just the code.

In order to review APO well, I'd probably want to do a deep dive into ln-symmetry, which means brushing up on my lightning knowledge.

For CTV I might instead choose to _only_ consider the OP_VAULT use case, since that's the only one I'm currently interested in. But I'd still want to feel comfortable that enough other people reviewed the plethora of other use cases you mentioned.

That creates a dilemma if I had to choose between them: `CTV` + `OP_VAULT` is probably easier to grok than ANY_PREVOUT + ln-symmetry, but I'm not super comfortable ignoring all the other CTV use cases during review. Now, having to review _both_ just seems too daunting. In practice I would just end up approving some subset of the proposal. Hard to predict if review coverage over the whole proposal would end up equally distributed.

Both proposals require thinking through the dilemma of whether we want to restrict them to an enumerated set of use cases, make them potentially flexible but without any guarantee they'll be useful (we're not running out of new op codes anytime soon), or thoroughly work out a broad set of conceivable use cases. Or some other way out of the dilemma. It's just hard to quantify how much thinking work that is (for starters, going though piles of mailinglist archives).


Regarding the 4 year soft fork cycle: it used to be much quicker. And if you only consider the time between when a soft-fork proposal was reasonably fleshed out and it's time of activation, it's more like 1 or 2 years. Taproot was simpler than SegWit, but both were orders of magnitude more complicated than earlier proposals. I don't know how much we should attribute the recent slow deployments to drama, lack of energy or their complexity.



> CTV and APO have been extant long enough that there is broad consensus among most that both are safe changes

I never looked at CTV code and only know APO at a high level (despite having done two technical podcast episodes on it). I'm skeptical that "enough" people have looked at these proposals in enough nitty-gritty detail and actually tried to build things on top of it. At the same time I also don't really know how many people is "enough".

Are the opvault-demo and CLN implementation enough? I guess I'd want to see two more. But then, Bitcoin Core didn't even have a SegWit wallet before that was deployed and activated. And how far along were lightning prototypes back then? So in that sense these two proposals may indeed be relatively mature.

In other words, maybe one clearly useful demo project per proposed softfork _is_ enough. Personally I'd just have to more thoroughly review the proposals to determine if these demo's really exercise them well.

-------------------------

ajtowns | 2023-09-29 19:51:55 UTC | #9

Couple of comments:

[quote="jamesob, post:1, topic:98"]
I’d like to propose a softfork deployment that activates the consensus changes outlined in
[/quote]

Personally, I think the "activation" part is probably the least interesting/useful thing to talk about.

[quote="jamesob, post:1, topic:98"]
The patches for BIP-118 and BIP-119 have long been stable and well scrutinized;
[/quote]

I don't really think that's quite true: there's a couple of outstanding changes to BIP 118 that seem pretty likely to be desirable ([inq#19](https://github.com/bitcoin-inquisition/bitcoin/issues/19)); and I don't really think either 118 or 119 have really been super well scrutinized. Meanwhile, 345 hasn't had been officially published yet.

[quote="jamesob, post:1, topic:98"]
Vaults with BIP-345, on the other hand, would be delayed only by the speed of wallet implementers. [Example wallet implementations](https://github.com/jamesob/opvault-demo/) already exist,
[/quote]

The link there goes to a project whose first commit was two weeks ago. To me, that's an exciting first step -- I'd rather read about how that works, see demo transactions, and think about if either it's broken, or if it could be improved, either with better use of the proposed features, or with alternative proposals. But using a two week old demo as an argument for activating a consensus change just seems way over-eager.

[quote="jamesob, post:1, topic:98"]
The implementation for these consensus changes is only about [7,000 lines ](https://github.com/bitcoin/bitcoin/compare/master...jamesob:bitcoin:2023-09-covtools-softfork?), and that includes comprehensive testing.
[/quote]

I wouldn't call the bip 118 code "comprehensively tested" -- implementing the proposed change mentioned above doesn't cause any breakage in the unit tests, and there aren't test vectors included with the BIP, eg.

For me, I'd rate the next steps as:
1. make it easy for people to try out demos and see what value they have (the vault stuff, ln-symmetry, spacechains?), ideally being able to run a live demo in public, in a way that allows people to attempt to attack it
2. improve tooling for all this stuff -- we still don't really have all the tooling for *taproot*, after all (eg musig2, adaptor signatures, descriptors to specify specific tapscripts...)
3. once we've demonstrated that these really are good idea, get more thorough review of everything, including:
   * more tests
   * check if there's any tangible improvements to be had from tweaking the API (`op_txhash` etc)
   * do our best to make sure that people can understand the proposed changes and judge them on their merits
4. only then see about activating them

-------------------------

ajtowns | 2023-09-29 20:22:41 UTC | #10

Oh, also: ln-symmetry likely needs work on package relay and fees paid via ephemeral transactions and ways to avoid pinning (which may also need consensus changes, if done via an assertion "this signature is invalid if the tx is larger than X vbytes")

-------------------------

harding | 2023-09-29 20:31:57 UTC | #11

I don't oppose any of these proposals, individually or together, and I'll be happy to see any or all of them included in Bitcoin (as long as no major problems are found during review).  However, I continue to feel like adding specific features to Bitcoin for unproven usecases is a suboptimal approach.  Looking at @jamesob's list:

[quote="jamesob, post:1, topic:98"]
* [vaults ](https://bitcoinops.org/en/topics/vaults/) (reactive custodial security),
* [LN-Symmetry ](https://bitcoinops.org/en/topics/eltoo/),
* efficient implementations of [DLCs ](https://bitcoinops.org/en/topics/discreet-log-contracts/),
* [non-interactive channel openings ](https://utxos.org/uses/non-interactive-channels/),
* [congestion control ](https://utxos.org/uses/scaling/),
* decentralized mining pools (via [CTV compression in coinbase payouts ](https://utxos.org/uses/miningpools/)),
* various [Lightning efficiency improvements ](https://twitter.com/roasbeef/status/1692589689939579259),
* using [covenant based timeout-trees ](https://bitcoinops.org/en/newsletters/2023/09/27/) to scale Lightning, and more generally enabling [channel factories](https://bitcoinops.org/en/topics/channel-factories/).
[/quote]

- **Vaults** can be done today with presigned transactions.  The presigned versions are a lot harder to implement correctly than with `OP_CTV` + `OP_VAULT`, they can't receive payments without interaction with the payer, and the vault user needs to store more data.  However, it seems to me that if there were high demand for the general ability to have hot spends announced onchain with a cancellation window, a significant number of people would be using presigned vaults---but I'm not aware of that happening.

- **LN-Symmetry:** has not received a significant amount of work from people who usually work on LN and discussions about it often indicate that some LN developers think a different protocol is needed, such as one that continue to provide penalties.  It also seems to me personally that John Law's tunable penalties protocol provides the primary advantage of LN-Symmetry (proper assignment of penalties when >2 parties are involved) without requiring any consensus change.

- **More efficient DLCs:** if this is referring to [what I think](https://bitcoinops.org/en/newsletters/2022/02/02/#improving-dlc-efficiency-by-changing-script), this efficiency boost is just in the amount of offchain operations DLC users need to generate and store; it doesn't significantly change the onchain footprint of DLCs.  DLCs have been operational for several years but have not seen any major use that I'm aware of; I believe this is partly due to their poor scalability characteristics.  Every DLC-based trade needs to be anchored onchain; it's possible to use the same anchor for multiple trades, but only if you use the same trading partner(s) for all of them.  By comparison, a centralized trading clearinghouse (e.g. an exchange) requires trust but can pair any seller with any buyer.  AFAIK, there are no proposed solutions for the scalability problem of DLCs.

- **Non-interactive channel openings:** I haven't looked into this in detail, but this looks to me like a version of [swap in potentiam](https://bitcoinops.org/en/newsletters/2023/01/11/#non-interactive-ln-channel-open-commitments), which is possible today and with (I'd guess) about the same onchain cost as using CTV.

- **Congestion control:** to do congestion control, you need to be able to do payment batching, but payment batching remains underused by popular custodians (although, happily it is used more than when Jeremy Rubin first posted CTV).

- **Decentralized mining pools:** I haven't looked at this in detail, but we already have a mechanism (Stratum v2) that allows individual hashers to get pretty close to decentralized pooling, but very few hashers seem to be using those features of Stratum v2.

- **Various LN efficiency improvements:** it's hard to tell what's being proposed from a tweet-level of text here, but simplification of existing highly used code is something I'd find to be an effective argument for CTV.

- **Using covenant based timeout trees:** Law's previous proposals, e.g. hierarchical channels and the Fully Factory Optimized Watchtower Free (FFO-WF) protocols provide channel factories that are more capital efficient and more compatible with casual user behavior than previous factory designs, and they don't require any consensus changes to implement.  The timeout tree design can use those previous protocols to obtain much greater efficiency, so the first step would be implementing hierarchical channels and FFO-WF in LN nodes.

I feel like (with the possible exception of Osuntokun's tweet), if these were really great ideas that people really wanted and that application developers were eager about, wouldn't we see more work on, and adoption of, the versions of those protocols which can be deployed now without consensus changes?

Another way to look at it, using vaults as an example, if `OP_VAULT` will give thousands of custodians and potentially millions of casual users a 10x better experience keeping their funds safe, why is it that so few developers are working on the 2x-5x improvement that presigned vaults provides and few users seem excited about presigned vaults?

One reason I can think of for this incongruity between the enthusiasm for CTV/APO/VAULT and the lack of development or use of immediately-deployable versions is that most developers are waiting for CTV/etc. to become available before building the their version of vaults, improved channels, and other features.  But, if that's the case, how many other great ideas are being held up waiting for the entire Bitcoin userbase to agree on a very specific set of consensus changes?

I feel like a better approach would be to enable a really general set of features that would allow anyone to create and deploy scripts that work like CTV/APO/VAULT and more, even if not in the most efficient way, without having to wait for permission.  If we see a lot of onchain use (or otherwise demonstrated offchain use) of a generalized feature for something that would be more efficient if it was specialized, we can soft fork in that specialized feature and everyone who previously used the generalized version can immediately switch to the specialized version---an instant win with hopefully no controversy.

The stub proposal I've used for this in the past is `OP_CSFS` + `OP_CAT`, or some variation on it (such as the SHA-based version used in Elements).  That's a really small consensus change and yet it allows implementation of any sort of transaction introspection we might want.  Of course, maybe Simplicity, BTC Lisp, TXHASH, MATT, or something else would be better.  The risks of this approach are recursive covenants and the creation of MEV scenarios, but it seems to me that there's also a risk of stifling development if we won't build anything until a new limited opcode is added for it in an event that might only happen once every four years, plus the risk of adding specialized opcodes or sighashes to Bitcoin that we will need to support forever but which might never see any significant use.

-------------------------

