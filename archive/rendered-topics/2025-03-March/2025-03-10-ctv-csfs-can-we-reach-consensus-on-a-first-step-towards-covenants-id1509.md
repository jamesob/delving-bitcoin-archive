# CTV+CSFS: Can we reach consensus on a first step towards covenants?

stevenroose | 2025-03-11 09:44:00 UTC | #1

First of all, it is not easy to write about covenants without upsetting at least some people, given the various different leanings on the topic. Don’t worry, I won't be announcing an activation client, nor pointing fingers or proposing yet another opcode.

Recently I have noticed a small uptick in positive sentiment on the topic of covenants; at least within my own bubble of bitcoin developers. When I noticed some people were sharing their ideal covenants roadmap on Twitter, [I did the same](https://x.com/stevenroose3/status/1893206459602870583). While I didn’t put a whole lot of thought into this roadmap (you’re not supposed to do that on Twitter), it was nonetheless based on two years of pondering on the topic. To my surprise, I received a lot of positive feedback, both publicly and privately. And even more to my surprise, I heard almost no voices of objection. So I’m here to share a little bit of the thinking process behind this roadmap.

![|579x301](upload://n8KmzdAyvu07AuUQ2Uf8Twmljbl.png)

# Consensus upgrades

Bitcoin protocol developers have only two duties: maintain and protect the decentralized and permissionless nature of the bitcoin network; and provide functionality for app developers to build the product that is bitcoin as digital money.

Immediately, two challenges come to mind:

* Changing consensus code to provide new functionality always comes with a certain risk. From buggy code to chain splits to changes in the network’s incentive structures, it is our duty to take the utmost care when evaluating changes to bitcoin’s consensus before proposing them to users.
* Since bitcoin users don’t use the protocol directly, but app developers do when they are building the applications on top, it is hard to evaluate the utility of different proposals.

Because the design space of application developers is so vast and bitcoin users so varied, I think the best approach is to make the bitcoin protocol as expressive as possible, while focussing on maintaining the security of the network and not endangering bitcoin’s core properties.

# On generalized covenants

In light of this approach, I specifically like the outlook of the [Great Script Restoration](https://rusty.ozlabs.org/2024/01/19/the-great-opcode-restoration.html) project of Rusty Russel. It aims to re-instante much of the originally intended expressivity of bitcoin’s Script and at the same time safeguarding validation requirements by introducing the VarOps budget for Script execution. The basic opcodes the GSR re-enables can then be extended with a combination of opcodes specifically tailored to provide transaction introspection (covenants) in an efficient, practical and safe way.

While no one will deny that covenants are a useful tool to build a wide variety of second-layer protocols, I feel that there is still hesitancy in the wider bitcoin user community on how the merits of the functionality covenants and second-layer protocols enable outweigh the risks of introducing them.

# A careful first step

Given the above arguments, I feel like we have both not yet figured out how to best design a practical protocol for fully generalized covenants, nor fully figured out their implications for the network’s incentives.

Technically however, I do feel that we can already sufficiently envision at least how the basis of any more generalized solution will look. For example, any proposal currently being designed, will benefit from an opcode to succinctly assert certain transaction properties. This is why I took on the work to spec out and implement the [OP_TXHASH opcode](https://covenants.info/proposals/txhash/) according to Russel O’Connor’s ideas. While TXHASH as it stands is far from being mature enough to be deployed, it can function as an upgrade path for the similar but more restricted opcode [OP_CTV](https://covenants.info/proposals/ctv/). CTV has been known to the developer space for several years, has been deployed on various [test networks](https://github.com/bitcoin-inquisition/bitcoin) and has been used to develop [several use cases](https://utxos.org/uses/).

While CTV alone is quite a powerful tool for various second-layer protocols, it might on its own not be compelling enough to perform a soft-fork upgrade. That’s why I think it makes sense to bundle it with another simple and mature opcode, [OP_CHECKSIGFROMSTACK](https://github.com/bitcoin/bips/blob/050d422b2ac24d8221edab0ff0053e0f585409f7/bip-0348.md). This opcode is also active on some test networks and has been [deployed to the Liquid network for years](https://github.com/ElementsProject/elements/commit/c35693257ca59b80659cfc4a965311f028c2d751#diff-a0337ffd7259e8c7c9a7786d6dbd420c80abfa1afdb34ebae3261109d9ae3c19R1328).

What attracts me to this combination of opcodes, CTV & CSFS, is that both of them stay useful in the scenario of further upgrades towards generalized covenants. Templating transactions is a useful tool in itself, and CTV can be upgraded with TXHASH to make the templating a lot more powerful. CSFS on the other hand, enabling simple signature verification, is such a basic tool that it will likely always have some use cases.

Besides, CTV and CSFS enable only the most restrictive version of covenants possible. This both gives us more time to design technical solutions for generalized covenants and better study their implications for the network.

# CTV + CSFS, what do we actually get?

Summarizing, CTV & CSFS as a combination both enhances existing protocols and makes some entirely new protocols possible, while still being a very careful addition to bitcoin’s expressiveness.

Some examples of new protocol possibilities:

* In general terms, CTV simplifies any protocol that is based on pre-signed transactions by removing the need for both the interactivity of the pre-signing and the requirement to store the signatures. This in itself can benefit both existing protocols and new protocols yet uninvented.
* Since CTV+CSFS functions as an equivalent for [SIGHASH_ANYPREVOUT](https://covenants.info/proposals/apo/) (APO), which enables Lightning Symmetry (formerly Eltoo). LN Symmerty is argued to be a better and more user-friendly implementation of Lightning channels, while it also makes multi-party channels possible and simplifies the implementation of PTLCs, further improving the Lightning Network.
Greg Sanders’ [deep dive summary on LN Symmetry](https://delvingbitcoin.org/t/ln-symmetry-project-recap/359) mentions additional benefits of CTV specifically. Various Lightning devs have stated their intention to implement LN Symmetry if CTV+CSFS would be available.
* More generally, CTV + CSFS as an APO replacement enabling re-bindable signatures can have further usage beyond LN Symmetry.
* I am obviously biased here, but CTV is a game-changer for [Ark](https://ark-protocol.org/), the younger second-layer protocol. While both Ark implementations are currently being built within bitcoin’s actual capabilities, the [benefits of CTV](https://x.com/stevenroose3/status/1865144252751028733) to the user experience are undeniable, and it is without doubt that both implementations will utilize CTV as soon as it is available.
* Protocols involving [DLCs](https://dci.mit.edu/smart-contracts), which have been growing in popularity, can be [significantly simplified](https://gnusha.org/pi/bitcoindev/CAH5Bsr2vxL3FWXnJTszMQj83jTVdRvvuVpimEfY7JpFCyP1AZA@mail.gmail.com/) using CTV.
* [BitVM](https://bitvm.org/) would be able to drastically reduce its script sizes by replacing their current use of Lamport signatures implemented in Script by simple CSFS operations, while CTV could eliminate the need for some of the pre-signed transactions in its constructions.
* [Very limited vaults](https://github.com/jamesob/simple-ctv-vault) have been built with CTV.

# Where do we go from here?

I mostly wanted to put this out there as a first step to see how people feel about this minimal package of opcodes as a first step on the covenants roadmap and hear out any feedback.

James O’Beirne and Rearden, who authored the latest implementations of the relevant BIPs ([ctv](https://github.com/bitcoin/bips/blob/050d422b2ac24d8221edab0ff0053e0f585409f7/bip-0119.mediawiki), [csfs](https://github.com/bitcoin/bips/blob/050d422b2ac24d8221edab0ff0053e0f585409f7/bip-0348.md)), are working on updating their pull request to the Bitcoin Core repository so that they can be re-opened. While both the implementations are not new, they will obviously have to be thoroughly reviewed by other developers with knowledge of the codebase.

Additionally, I have been talking with the authors of alternative covenant proposals to ask them how they feel about this idea. Surprisingly, I have received almost exclusively positive responses, some of them publicly announcing their opinion on Twitter. But then again, my own personal bubble of developers is obviously biased in favor of expanding Script with more functionality. That's why I'd like to gather some feedback from a broader audience.

-------------------------

1440000bytes | 2025-03-10 22:58:38 UTC | #2

I have added the link to this post as 'rationale' in the covenants support wiki page for your row.

https://en.bitcoin.it/w/index.php?title=Covenants_support&diff=prev&oldid=70610

-------------------------

moonsettler | 2025-03-10 23:35:27 UTC | #3

If I wanted to arrive there I would probably go with:

1. CTV + CSFS + IKEY [+ PC]
2. TXHASH + CCV [+ CAT]
3. GSR + ECops

TXHASH and CCV together make it extremely unlikely that CAT would be used for wonky inefficient introspection.

If you have PC you can delay CAT, but you need multi-commitments and Merkle stuff eventually. Alternatively you can go for CAT in step 2.

STARKs become a possibility at step 2 with CAT and afaik pretty much a certainty at step 3.

-------------------------

AntoineP | 2025-03-11 22:08:19 UTC | #4

Thanks for writing this. My [personal view](https://antoinep.com/posts/softforks) on the topic is that we should change Bitcoin's consensus rules if we have a good reason to, but **only if** we do. I would like to offer some push backs and explain why i don't think a `CTV`+`CSFS` does not meet the bar we should set for ourselves for a soft fork.

[quote="stevenroose, post:1, topic:1509"]
In general terms, CTV simplifies any protocol that is based on pre-signed transactions by removing the need for both the interactivity of the pre-signing and the requirement to store the signatures.
[/quote]

This is actually the main selling point of CTV in my opinion. Interactivity reduction is often underappreciated.

[quote="stevenroose, post:1, topic:1509"]
Various Lightning devs have stated their intention to implement LN Symmetry if CTV+CSFS would be available.
[/quote]

It would be useful to get some statement on the matter from active contributors to the Lightning specifications. What i heard from some people there is that Eltoo might be nice for (far) future stuff, like multiparty channels. But would not be close to a priority for the moment, where all resources are currently allocated to features demanded by users that can be implemented today (including some made possible by the last soft fork but not yet implemented).

[quote="stevenroose, post:1, topic:1509"]
More generally, CTV + CSFS as an APO replacement enabling re-bindable signatures can have further usage beyond LN Symmetry.
[/quote]

This is pretty vague. What are concrete examples of applications leveraging those that have any realistic chance of bringing value to Bitcoin end users, should such a soft fork be activated?

[quote="stevenroose, post:1, topic:1509"]
CTV is a game-changer for [Ark](https://ark-protocol.org/), the younger second-layer protocol. While both Ark implementations are currently being built within bitcoin’s actual capabilities, the [benefits of CTV](https://x.com/stevenroose3/status/1865144252751028733) to the user experience are undeniable, and it is without doubt that both implementations will utilize CTV as soon as it is available.
[/quote]

If Ark becomes widely used and CTV makes it significantly better, then i think this would be a very convincing argument in favor of a CTV soft fork.

Could you list the benefits of CTV for Ark here, instead of linking to Twitter? This is better for discussion here, future reference, people who don't have an account there, and for when it's down like it's been in the past couple days.

You say both implementations would use CTV as soon as possible beyond any doubt, ~~but it seems the other team stated otherwise on Twitter. (Reference will have to wait until Twitter is back up.)~~ *edit: couldn't find a reference*

[quote="stevenroose, post:1, topic:1509"]
Protocols involving [DLCs](https://dci.mit.edu/smart-contracts), which have been growing in popularity, can be [significantly simplified](https://gnusha.org/pi/bitcoindev/CAH5Bsr2vxL3FWXnJTszMQj83jTVdRvvuVpimEfY7JpFCyP1AZA@mail.gmail.com/) using CTV.
[/quote]

Bitcoin users don't seem too interested in DLCs, [at all](https://10101.finance/blog/10101-is-shutting-down).

[quote="stevenroose, post:1, topic:1509"]
[BitVM](https://bitvm.org/) would be able to drastically reduce its script sizes by replacing their current use of Lamport signatures implemented in Script by simple CSFS operations, while CTV could eliminate the need for some of the pre-signed transactions in its constructions.
[/quote]

Has BitVM seen any adoption from Bitcoin users, either directly or indirectly?

[quote="stevenroose, post:1, topic:1509"]
[Very limited vaults](https://github.com/jamesob/simple-ctv-vault) have been built with CTV.
[/quote]

This is misleading to qualify such constructions as "vaults". Vaults were introduced as a way for Bitcoin users to receive coins on a script such as external payments may be canceled, by enforcing an unvault period whereby external spend are delayed for a configurable amount of time.

The construction described there simply defines a precomputed chain of transaction to spend a UTxO which requires using a hot wallet before making a payment, which kills the purpose of using a ""vault"" if your intention was to protect yourself from your hot wallet being compromised. In addition, this transaction chain requires committing to the amount which is by itself a gigantic footgun. Send too little to the address? Your funds are locked forever. Send too much? The excess is burned to fees. Which makes it so you need to first receive funds on your hot wallet before sending it to this utxo-with-precomputed-chain-of-transactions, and even then make really sure you are not going to burn your funds in doing so.

CTV [may be useful](https://gnusha.org/url/https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-April/020337.html) to reduce interactivity in some actual vault constructions, but those don't seem to have much traction in the first place.

---

In conclusion, from the presented motivations for soft-forking `CTV`+`CSFS` today, only one was clearly [demonstrated](https://delvingbitcoin.org/t/ln-symmetry-project-recap/359?u=antoinep). Reduced interactivity in protocols by replacing presigned transactions with `CTV` commitments. This seems pretty light on its own for a Bitcoin soft fork.

-------------------------

1440000bytes | 2025-03-11 16:22:24 UTC | #5

[quote="AntoineP, post:4, topic:1509"]
You say both implementations would use CTV as soon as possible beyond any doubt, but it seems the other team stated otherwise on Twitter. (Reference will have to wait until Twitter is back up.)
[/quote]

Other team will use CTV if its available although prefer TXHASH which is explained as the next step by OP. 

There are lot of things that are discussed privately and other team is also willing to issue a grant for a proof of concept on signet that uses CTV for Ark.
[quote="AntoineP, post:4, topic:1509"]
Bitcoin users don’t seem too interested in DLCs, [at all](https://10101.finance/blog/10101-is-shutting-down).
[/quote]

This is misleading. [Atomic Finance](https://atomic.finance/) and [Lava](https://www.lava.xyz/) use discreet log contracts. Failure of one project does not show anything about demand for discreet log contracts and improvements with covenants.

[quote="AntoineP, post:4, topic:1509"]
Has BitVM seen any adoption from Bitcoin users, either directly or indirectly?
[/quote]

Yes multiple projects are based on BitVM.

[quote="AntoineP, post:4, topic:1509"]
This is misleading to qualify such constructions as “vaults”. Vaults were introduced as a way for Bitcoin users to receive coins on a script such as external payments may be canceled, by enforcing an unvault period whereby external spend are delayed for a configurable amount of time.
[/quote]

Its not misleading. Vaults can even be built using pre-signed transactions.

[quote="AntoineP, post:4, topic:1509"]
Send too little to the address? Your funds are locked forever. Send too much? The excess is burned to fees. Which makes it so you need to first receive funds on your hot wallet before sending it to this utxo-with-precomputed-chain-of-transactions, and even then make really sure you are not going to burn your funds in doing so.
[/quote]

The demo uses a simple CTV vault with single hop. There are several ways to setup a vault with CTV.

[quote="AntoineP, post:4, topic:1509"]
In conclusion, from the presented motivations for soft-forking `CTV`+`CSFS` today, only one was clearly [demonstrated](https://delvingbitcoin.org/t/ln-symmetry-project-recap/359). Reduced interactivity in protocols by replacing presigned transactions with `CTV` commitments. This seems pretty light on its own for a Bitcoin soft fork.
[/quote]

It wasn't light in [2022](https://gnusha.org/pi/bitcoindev/p3P0m2_aNXd-4oYhFjCKJyI8zQXahmZed6bv7lnj9M9HbP9gMqMtJr-pP7XRAPs-rn_fJuGu1cv9ero5i8f0cvyZrMXYPzPx17CxJ2ZSvRk=@protonmail.com/) when you were interested to activate APO and built a [vault demo](https://github.com/darosior/simple-anyprevout-vault) using it. [Co-author of APO](https://xcancel.com/Snyke/status/1895880013796556818) supports CTV+CSFS as a soft fork.

-------------------------

reardencode | 2025-03-11 16:19:25 UTC | #6

[quote="AntoineP, post:4, topic:1509"]
[quote="stevenroose, post:1, topic:1509"]
Various Lightning devs have stated their intention to implement LN Symmetry if CTV+CSFS would be available.
[/quote]

It would be useful to get some statement on the matter from active contributors to the Lightning specifications.
[/quote]
![image|690x330](upload://z097XjR135aqxq9BtuG1oNBawVc.png)
https://x.com/rusty_twit/status/1865546920779022434

-------------------------

instagibbs | 2025-03-11 16:26:19 UTC | #7

[quote="stevenroose, post:1, topic:1509"]
Given the above arguments, I feel like we have both not yet figured out how to best design a practical protocol for fully generalized covenants, nor fully figured out their implications for the network’s incentives.
[/quote]

Agreed. I am very wary of efforts to lock people into a "roadmap" when we can barely see what's in front of us. It's very unclear to me what is possible and desire-able longer term, but my personal preference is to bite the bullet and build/use something from the ground up that people can actually reason about.

That's clearly a long way out and there's no assurance that anything will pan out for practical and reasonable reasons, so thinking shorter term improvements is the best we can do, whether or not we decide to adopt the most mature proposals.

[quote="stevenroose, post:1, topic:1509"]
ince CTV+CSFS functions as an equivalent for [SIGHASH_ANYPREVOUT](https://covenants.info/proposals/apo/) (APO)
[/quote]

Caveat here being it only emulates a form of `SIGHASH_ALL` style APO. This is somewhat limiting and comes with downsides / additional roadblocks to adoption as a slot in replacement. e.g., it will heavily rely on CPFP instead of single tx exo fees which is a bit cheaper and reliably from relay PoV.

[quote="AntoineP, post:4, topic:1509"]
But would not be close to a priority for the moment, where all resources are currently allocated to features demanded by users that can be implemented today (including some made possible by the last soft fork but not yet implemented).
[/quote]

It depends on a number factors such as timing with other updates(one update of commitment tx is easier than two), difficulty of swapping out parts, and synergies with other changes to protocols.

I do think it would accelerate updating to PTLCs, f.e., even if symmetry per-se is never adopted. Re-bindable signatures just makes life so much less of a headache when stacking protocols...

I would caution on an overall effort to gate a softfork on a single group's public declaration of using it promptly. If they're good, powerful, modular primitives that seem to improve many existing constructions (with proper integration testing!), I think that's the signal we're looking for for "utility".

[quote="AntoineP, post:4, topic:1509"]
Has BitVM seen any adoption from Bitcoin users, either directly or indirectly?
[/quote]

It's still in development by [multiple companies](https://bitvm.org/#about-bitvm-alliance) from my understanding, so it's hard to judge whether or not it will lead to usage, or what the end usage will be *for*. I am not interesting in jpegs, but I find trust-minimized bridges to a Shielded CSV-like system quite attractive.

What I'd like on this claim is concrete numbers on the size of improvement. CSFS doesn't change the security/trust model of BitVM, when something like TXHASH could. And the efficiency claims would have to be weighed against alternative improvements e.g., bignum support alone supposedly brings down the disprove tx from 4MWU to <400kWU. 

[quote="1440000bytes, post:5, topic:1509"]
There are lot of things that are discussed privately
[/quote]

Unfortunately this is not an effective strategy to gather industry feedback. It is not incumbent on all reviewers to chase down motivations and validation for changes.

Overall, I think CTV+CSFS is an incrementalist improvement, but it *is* an improvement, built on modular pieces that likely won't become wholly deprecated in the future if and when we rewrite script or decide we're not worried about more MEVil. In other words, ignoring the social/technical costs of deployments which are very non-trivial, I believe it would be a positive outcome with limited maintenance burden.

-------------------------

stevenroose | 2025-03-11 16:42:06 UTC | #8

I think @1440000bytes already responded with some things I wanted to mention as well. I'm not going to re-iterate them.

I think my main response here is something that I already tried to emphasize in the OP. **Bitcoin users don't use the bitcoin protocol.** They use applications that application developers built on the protocol and it is these application developers that use the bitcoin protocol. I was there when users knew whether their wallet was "p2shwpkh" or "p2pkh" etc, the time when all users were developers. That time is over, because bitcoin is growing up. This basic principle inspires a lot of my responses.

[quote="AntoineP, post:4, topic:1509"]
Bitcoin users don’t seem too interested in DLCs, [at all](https://10101.finance/blog/10101-is-shutting-down).
[/quote]

Bitcoin users shouldn't be interested in DLCs. You mention a failed company building on DLCs, while @1440000bytes mentions one that raised 30M in funding last week. Only mentioning one or the other is misleading.

[quote="AntoineP, post:4, topic:1509"]
Has BitVM seen any adoption from Bitcoin users, either directly or indirectly?
[/quote]

Again, no. Users won't use BitVM, but developers will. And they are. BitVM is obviously a very young and complex project, but multiple teams with ample funding have been [building an implementation](https://github.com/BitVM/BitVM) in the last year.

In fact, BitVM is a good example of the result of the lack of expressiveness in bitcoin's protocol. It is essentially a bad version [MATT](https://merkle.fun/), but built as a workaround around the lack of covenants in bitcoin. To me this shows that there is great demand for this expressiveness and developers are going to extreme lengths to obtain equivalent behavior in the ugliest possible ways.

[quote="AntoineP, post:4, topic:1509"]
CTV [may be useful](https://gnusha.org/url/https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-April/020337.html) to reduce interactivity in some actual vault constructions, but those don’t seem to have much traction in the first place.
[/quote]

I find is quite disingenuous when people point to the lack of implementations as an argument against a protocol upgrade. First of all, there were 0 implementations of taproot when it was deployed, but more generally, we cannot expect our developer ecosystem to be building on features we are still pondering to even add or not. Most of us can't afford to spend months building something that in the best case will have to be shelved for another two years and worst case will be thrown out entirely.

There is an enormous difference in time commitment between brainstorming how certain primitives can be combined to achieve certain functionality, and actually implementing them. I think people with sufficient experience don't need the latter to be convinced of the former.

-------------------------

reardencode | 2025-03-11 17:19:12 UTC | #9

I support this effort, as reflected in my [pinned post](https://x.com/reardencode/status/1871343039123452340
) on X.

![image|690x353](upload://m7pNR3NmzBQGDpvYTnXZdFdxAhX.png)

To slightly expand on why specifically CTV+CSFS(and IKEY+PC?):

IKEY: [INTERNALKEY](https://github.com/bitcoin/bips/blob/050d422b2ac24d8221edab0ff0053e0f585409f7/bip-0349.md)
PC: [PAIRCOMMIT](https://github.com/bitcoin/bips/pull/1699)

There's a tension between "only change bitcoin consensus in gravest extremity" and "upgrade bitcoin whenever there's a clear improvement to be had". Team Slow and Steady (as promoted by @ajtowns, @moneyball, and others) is a proposed middle of the road that aligns well with the CTV+CSFS upgrade.

While there are many, @AntoineP included, who are in a fourth camp of "only change bitcoin consensus when there is a strong and clear reason", that standard is simply not realizable while navigating the delicately balanced space between "only in gravest extremity" and "slow and steady". Any upgrade that would add sufficient capabilities to show such a strong and clear reason will necessarily also raise the specter of potentially weakening the incentives which are necessary for bitcoin's continued success. While I personally am confident that an upgrade such as CAT, Simplicity, or Lisp would not harm the incentive structure, that is difficult (I think impossible) to defend to Team Slow and Steady.

Therefore, we are left not with a choice of "CTV+CSFS now or something with stronger justification later" but rather "CTV+CSFS at some point, or nothing until solving the 2106 timestamp overflow".

----

Lastly, I'd like to expand on why I think CTV+CSFS alone may need IKEY and PC to reach activation.

There was some debate during the development over the exclusion of a keyless spend. This was resolved with the justification that nearly all protocols have a cooperative n-of-n case that will dominate over whatever uncooperative cases. While this is broadly true, for some of these protocols, success would mean that the cooperative case itself is rarely broadcast which would then imply that uncooperative cases are the main spend seen in blocks. For those protocols, IKEY largely mitigates the cost of the mandatory key spend. Without IKEY, those potentially dominant spends are 32WU larger for no reason.

Thanks to @JeremyRubin's [key laddering concept](https://rubin.io/bitcoin/2024/12/02/csfs-ctv-rekey-symmetry/), we know that CSFS alone enables Merkle-tree-ish commitments by using sigops to ladder keys which then commit to items. Given this, and the usefulness of such multi-commitments, I think it would be poor stewardship to add CSFS to bitcoin without also enabling multi-commitments directly. If we do so then protocol developers will use sigops instead, pushing up validation costs (not above current worst cases but simply unnecessarily). When I was developing the CSFS BIP, I briefly explored the idea of using magic keys of small sizes to specify additional items from the stack to sign over. In the end, I concluded that that functionality belongs in a separate opcode. @moonsettler developed PC as that opcode.

---

To tie this all together: While CAT would also enable multi-commitments for CSFS, CAT also violates the "slow and steady" ethos by introducing many capabilities with difficult to reason about incentive implications. PC is ideally situated to be everything we need to maintain incentive alignment between nodes and application protocol developers in a post-CSFS world, without introducing additional capabilities beyond those already added by CSFS itself.

-------------------------

instagibbs | 2025-03-11 18:39:51 UTC | #10

[quote="stevenroose, post:8, topic:1509"]
I find is quite disingenuous when people point to the lack of implementations as an argument against a protocol upgrade. First of all, there were 0 implementations of taproot when it was deployed, but more generally, we cannot expect our developer ecosystem to be building on features we are still pondering to even add or not. Most of us can’t afford to spend months building something that in the best case will have to be shelved for another two years and worst case will be thrown out entirely.
[/quote]

Appealing to history means you can only convince people perhaps that taproot wasn't vetted enough. Valid argument to be made (I'm sympathetic to it, really), but I don't think it will persuade people the integration testing bar should be the same 7 years later.

With regards to the latter point, there's a lot of potential distance between "blog posts" and "fully production ready integrations". Pushing this effort forward probably involves getting people to demonstrate in specification and end to end PoC code which is firmly between the two ends.

-------------------------

AntoineP | 2025-03-11 18:41:38 UTC | #11

[quote="stevenroose, post:8, topic:1509"]
**Bitcoin users don’t use the bitcoin protocol.** They use applications that application developers built on the protocol and it is these application developers that use the bitcoin protocol.
[/quote]

I mean.. sure? What's your point here?

[quote="stevenroose, post:8, topic:1509"]
Bitcoin users shouldn’t be interested in DLCs.
[/quote]

I am interpreting this charitably as "Bitcoin users should be interested in applications enabled by DLCs, not by DLCs themselves". I wholeheartedly agree and this has been a point i've raised a few times already: for consensus changes we should consider what is enabled to Bitcoin users down the road, and should only consider tools made available to app developers insofar as it may bring benefits to end users.

[quote="stevenroose, post:8, topic:1509"]
You mention a failed company building on DLCs, while @1440000bytes mentions one that raised 30M in funding last week. Only mentioning one or the other is misleading.
[/quote]

I am sorry if this came across as misleading, this was absolutely not my intention. I am merely pointing out that we've been hearing about DLCs for years, companies were built to bring applications it enabled to market, and the market shrugged. Others also [pointed out](https://gist.github.com/jonasnick/e9627f56d04732ca83e94d448d4b5a51#dlcs) the lack of traction did not seem to be due to the performance limitations that would be lifted by CTV.

If DLCs are going to be used as a motivation for a CTV soft fork, it seems appropriate to ask how it's going to 1) actually improve an application that 2) users actually care about.

[quote="stevenroose, post:8, topic:1509"]
[quote="AntoineP, post:4, topic:1509"]
Has BitVM seen any adoption from Bitcoin users, either directly or indirectly?
[/quote]

Again, no. Users won’t use BitVM, but developers will. And they are. BitVM is obviously a very young and complex project, but multiple teams with ample funding have been [building an implementation](https://github.com/BitVM/BitVM) in the last year.
[/quote]

Obviously my question about indirect users of BitVM was about whether anyone was using an application that a developer leveraged BitVM to build.

And if we are going to be uncharitable in interpreting each other's points here let me be clearer. Handwavy claims about speculative applications of a not-yet-existing protocol are not appropriate to justify a Bitcoin soft fork in the near term.

You've also mentioned funding a few times now. Funding is good, and is a signal to take into consideration as presumably people that are putting money on a table have an incentive to properly research the market fit of a product. But this can't be blindly followed in decisions about changing Bitcoin's consensus rules, for obvious reasons.

[quote="stevenroose, post:8, topic:1509"]
To me this shows that there is great demand for this expressiveness and developers are going to extreme lengths to obtain equivalent behavior in the ugliest possible ways.
[/quote]

There is some demand for more expressiveness from application developers for sure, that's why we are here. But again, what should be demonstrated to motivate a soft fork is whether there is demand from Bitcoin users for the applications this expressiveness would enable.

[quote="stevenroose, post:8, topic:1509"]
I find is quite disingenuous when people point to the lack of implementations as an argument against a protocol upgrade.
[/quote]

I don't see how it relates to your quote of my message, but let me reply to your point in general. I think it's reasonable to demonstrate usecases if they are going to be used as a motivation for a soft fork proposal. Demonstrating a usecase does not mean building a production-ready application. It merely means showing there is demand for the usecase and that the proposed change does indeed enable the usecase.

I guess what i'm describing is something akin to the "power user exploration phase" in @ajtowns' [Bitcoin Forking Guide](https://ajtowns.github.io/bfg/power.html).

[quote="stevenroose, post:8, topic:1509"]
First of all, there were 0 implementations of taproot when it was deployed,
[/quote]

Huh, what? What does Taproot has to do with all this?

[quote="stevenroose, post:8, topic:1509"]
we cannot expect our developer ecosystem to be building on features we are still pondering to even add or not
[/quote]

I think my previous comment addresses this, nobody expects a full production application to be built to motivate a soft fork.

[quote="stevenroose, post:8, topic:1509"]
There is an enormous difference in time commitment between brainstorming how certain primitives can be combined to achieve certain functionality, and actually implementing them. I think people with sufficient experience don’t need the latter to be convinced of the former.
[/quote]

Maybe i don't have enough experience, as you seem to imply, but i would argue the contrary. You seldom grasp all the ramifications of a protocol until you've actually implemented it.

-------------------------

stevenroose | 2025-03-11 20:27:40 UTC | #12

Allow me to wind back a little bit here.

The core of the motivation of the OP was that it should be our goal to make the bitcoin protocol better, but that we as a community are both not sure of the best complete package for covenants and not entirely convinced enabling generalized covenants will not have unforeseen negative side-effects.

The argument I make in OP is that "covenants" doesn't have to be an all-or-nothing question, and can be achieved in multiple steps.

[quote="AntoineP, post:11, topic:1509"]
Huh, what? What does Taproot has to do with all this?
[/quote]

I think it is a bit unfair to claim that we can't draw on examples from the past. Bitcoin soft-forks are clearly a unique technical and political challenge, and we have not much more than our own past to learn lessons from. Taproot, our previous soft-fork, seems to me like it came about with the same motivation I am citing as a motivation for covenants: it makes the bitcoin protocol better. At the time that seems to have been sufficient motivation. Of course things can change, but I never had the impression that we as a community felt that the taproot soft-fork was a mistake.

[quote="AntoineP, post:11, topic:1509"]
If DLCs are going to be used as a motivation for a CTV soft fork, it seems appropriate to ask how it’s going to 1) actually improve an application that 2) users actually care about.
[/quote]

[quote="AntoineP, post:11, topic:1509"]
I think it’s reasonable to demonstrate usecases if they are going to be used as a motivation for a soft fork proposal.
[/quote]

To come back to your main point of critique: I am not using the new use cases I mentioned as a motivation for this soft-fork proposal. What I am using as motivation is that it is a clear improvement to the protocol that falls cleanly in a larger roadmap towards covenants, while the opcodes in question are well-studied so the risk of unforeseen side-effects is small. And since we are far from close to reaching consensus on what the entirety of this roadmap looks like, it seems a reasonable technical decision to start with deploying a first step. 

I took great care in selecting (and demonstrating) that these are two opcodes that would not be "lost" if we would later execute further steps on a roadmap, and excluded other opcode candidates that could (like PAIRCOMMIT and INTERNALKEY).

I used the mentioned use cases as a added bonus of protocols that could be built of enhanced with *just* this starting package of opcodes. Obviously these benefits will be somehow limited when compared to benefits that generalized covenants would imply.

Again drawing on the past, taproot similarly didn't "enable" a whole lot of new *use-cases*, especially not compared to what a way simpler `OP_CHECKSCHNORRSIG` could do. It was nonetheless a worthwhile technical improvement of the protocol, as I think most developers at the time agreed on, even without having actually built any applications using it.

To finish, I obviously agree that as bitcoin matures and grows more important on the world stage, we should definitely raise the bar for changes to our consensus. I just get the feeling that for some people the height of this bar has become a fixation as some kind of proxy for bitcoin's political maturity, while on the technical side I hear almost no voices claiming that these opcodes are either endangering the network or will turn out useless after subsequent upgrades. I am afraid that this mentality is ultimately holding us back. 

Where should the bar realistically be? Is bitcoin a done project? Is it working perfectly and should it only be changed if its security is endangered? Or should we realize that bitcoin is still a work-in-progress and our goal is to incrementally extend its functionality while maintaining its security? This is obviously a false dichotomy, but I hope you can see the point I'm trying to make.

-------------------------

