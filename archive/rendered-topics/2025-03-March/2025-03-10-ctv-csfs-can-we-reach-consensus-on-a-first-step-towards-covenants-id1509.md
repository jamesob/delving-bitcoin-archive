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

AntoineP | 2025-03-12 18:14:32 UTC | #4

Thanks for writing this. My [personal view](https://antoinep.com/posts/softforks) on the topic is that we should change Bitcoin's consensus rules if we have a good reason to, but **only if** we do. I would like to offer some push backs and explain why i don't think a `CTV`+`CSFS` meets the bar we should set for ourselves for a soft fork.

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

ariard | 2025-03-12 01:20:13 UTC | #13

Thanks Steven for introducing this subject.

This is indeed a really compelling question if the community of bitcoin technical experts is in measure or not to reach consensus on covenants. Even if I disagree with a lot of the positions asserted within, this is always valuable to have that kind of conversation with the most dispassionate and disinterested minds.

*First and foremost*, I strong believe we should abstain ourselves to use expressions like "*we as a community felt*", who is we ? who is that community ? and if you go to read any treatise on psychology, biological emotions and as such expressing "feelings" is one of the most subjective experience that one can have as a human being. The domain of subjective psychology is not necessarily the most compelling lexical fields when you have to argue on objective criterias for an audience who is geographically spread around the world.

*Secondly*, I strongly also believe we should abstain ourselves to use expressions like "*Bitcoin soft-forks are clearly a unique technical and political challenge*,”.

As said in reply to James on the mailing list which is still pending moderation, **science and engineering != politics**. The old bitcoin motto is "*Vires in numeris*”, which for the ones less used with Latin means "*Trust the Numbers*", i.e all the cryptographic assumptions backing Bitcoin as a protocol and a network (Discreet Log problem, preimage-collision resistance for SHA256, Hashcash as a cryptographic protocol, etc).

That changing the Bitcoin consensus involves discussions, arguments and palavers with other human beings, yes this is a reality and otherwise there would be no one else with whom you could exchange few scarce sats of your stack. However, this is a social activity, e.g like a group of people who would play the different instruments of a musical orchestra, but it's not a political activity, as it’s understood in humanities (e.g in postmodern theories, science has always been a unique object of its own).

If my memory is correct, myself I've only used the "political" adjectives 2 in my more than half-a-decade Bitcoin career, and the 2 times it was to denounce the kind of horse-trading show we often see on bitcoin core, where contributor A goes to review PR #1 in exchange of another contributor B going to review PR #2 (— they mutually benefit of merged PRs for their funding renewal, afterall...), under all very questionable technical criteria. Horse-trading show more or less documented in IETF RFC 7282, pointing this is not technically sound "rough consensus”.

Of course, that historically in Bitcoin some people involved at the crux of the development process of `bitcoind`, the original client, have tacken at times, or been deliberately ambiguous with trying to establish any sound guidelines, while at the same time having massive commercial interests on a specific technical outcome has been seriously detrimental for the *credibility* of the process of consensus changes.

Let's remember the example of Blockstream, where folks have gone to raise venture capitals for $21M, publishing a sidechain with a proposed Bitcoin Script (e.g `OP_SIDECHAINPROOFVERIFY`) while the names on the paper had work experience with Mozilla and few other non-profits. At Mozilla, the Foundation is dissociated from Mozilla Corporation, and this was already true at the time in 2014.

I don't remember that Satoshi ever announced the venture capitalist funding of the Bitcoin whitepaper on TechCrunch.

Instead the Blockstream people (Sipa, Gmax, Adam Back, Austin Hill), especially the engineers could have instead focus their time nurturing initiatives like Bitcoin Optech and its online-ressources and in-person workshops, or ensuring the old bitcoin mailing list hosted by the Linux foundation was moderated on neutral grounds, or building up on the work of Amir Taaki in maintaining the BIPs. All community activities building bridges in matters of consensus changes…

But no what we had instead is a [kick-out](https://laanwj.github.io/2016/05/06/hostility-scams-and-moving-forward.html) of Gavin Andresen of its maintenance rights in 2016 (cf. laanwj’s blog post). Gavin Andresen and Mike Hearn, whatever all others judgements we can have on them, were technical builders and building communities. Somehow it turns out that Mike Hearn's bitcoinj micropayment channels, which was Satoshi's idea and of which the Lightning Network is only a sophisticated iteration, is what is certainly the most decentralized used second-layers, compared to "federated" side-chains.

All that said I wasn't active in Bitcoin protocol development during the early days of the Block Size War. I wasn't in Blockstream folks shoes and I might judge them harshly retrospectively without all the facts in mind. However, as said the adage "*Error humanum est, perseverare diabolicum*”.

So I understood Jeremy's "hot take" at the time of CTV 1st activation attempt in 2022 when he was jooking on one of his blog post (I can't remind which one exactly…) that current Bitcoin consensus change is like getting the blessing of a "*Grey Beards Committee*". One thing about unaccountable committee, you'll always find people to join them and _do nothing_, just here to take the periodic financial reward associated with the committed seat…

*Thirdly*, saying proposal OP_MNO or proposal OP_IJK has "funding", be it for-profit, non-profit, commercial venture, whatever is very short-sighted. Bitcoin as a protocol is first a complex scientific endeavor and let reminds [Theranos](https://en.wikipedia.org/wiki/Theranos) in matters of results on a highly-funded, scientifically deficient venture (cf. Theranos wikipedia page). W.r.t to non-profit sources of funding which can be driven with a more scientific perspective, if we exclude Jeremy the only independent names I'm aware off I've never contributed to bitcoin script code, themselves, even less on covenants, as far as I can remember.

*Fourthly*, on the statement that "*while the opcodes in question are well-studied so the risk of unforeseen side-effects is small*", this is factually wrong.

As Gleb Naumenko researched in the past and I pointed out this research on the mailing list, extending the bitcoin script can break the current UTXO model and enable to have tx-withhold risks of time-sensitive transactions.

Here the blog post, drawing from Ethereum academic literature actually:
https://blog.bitmex.com/txwithhold-smart-contracts/

To put it plainly, an attacker could (1) set a tx-withhold UTXO with a time-sensitive txid and (2) have this UTXO paying out miners for each block or sequence of blocks the txid is not confirmed in the UTXO set and (3) double-spend the funding UTXO of a LN channel as "honest" lightning transactions have been withheld.

*Fifthly*, *CheckTemplateVerify* and its iterations has been around since more than ~6 years, it has been broken few times at least there was a change to integrate input in the template to avoid ***"half-spend”*** issues, as Gmax pointed out. Very strict template is easier to reason on if it does enable some kind of change in the UTXO model.

*Sixthly*, for the Taproot there was an extended evaluation phase, with more than ~ 2 years and half between the proposition of BIP116 (`OP_MERKLEBRANCHVERIFY`) and BIP117 (Tail Call Execution Semantics), the original Taproot proposition in 2018 and the open-to-all on IRC review process over a span of weeks. This does not make Taproot perfect, as few people noticed after the code was merged for payment pools, there was no commitment to the script-path internal pubkey oddness *in the control block c*. Everyone is free to come to point out defects that could arise from Taproot consensus code, existing consensus code even made for Satoshi has its own issue, as the dummy element for `OP_CHECKMULTISIG` taught it.

*Seventhly*, on the "*Is it working perfectly and should it only be changed if its security is endangered?*". See the point above about ***tx-withholding risk*** and the ***"half-spend” issue*** that was found in the past about CTV. If bringing covenants in bitcoin is a real security risk, we should do no soft-fork, or delay until they have been more studied. Personally, if bitcoin dies next week, I'm lucky to have access to a wide range of other options to preserve my financial autonomy, even better than the average fiat billionaire.

This is not the case of a lot of people in war zones, in developing countries or who cannot access the mainstream 24 / 7 banking system because they are discriminated due to a social factor (e.g they have an exotic name), etc. *Bitcoin is an alternative for the ones who need it.*

*edited: to correct my own english and add missing link.*

-------------------------

ajtowns | 2025-03-12 03:11:53 UTC | #14

[quote="stevenroose, post:1, topic:1509"]
has been deployed on various [test networks](https://github.com/bitcoin-inquisition/bitcoin) and has been used to develop [several use cases](https://utxos.org/uses/).
[/quote]

I like the aphorism "trust, but verify" (of course, some Bitcoiners take it further with "don't trust, verify" -- but either way, the verification part is important). When I try to verify statements like these about CTV, they usually come up pretty lacking.

Taking the specific examples you list for CTV+CSFS, rather than the ones on the utxos page:

 * "CTV+CSFS functions as an equivalent for SIGHASH_ANYPREVOUT (APO), which enables Lightning Symmetry (formerly Eltoo)"

CTV+CSFS isn't equivalent to APO, it's somewhat more costly by requiring you to explicitly include the CTV hash in the witness data. The [TXHASH](https://gnusha.org/pi/bitcoindev/CAMZUoK=pkZuovtifBzdqhoyegzG+9hRTFEc7fG9nZPDK4KbU3w@mail.gmail.com/) approach is (in my opinion) a substantial improvement on that.

The TXHASH approach would be a taproot-only solution (as is CSFS), which would have the benefit of both simplifying the change (removing policy considerations for relay of bare-CTV txs, not having to consider the CTV in P2SH being unsatisfiable), and removing the need for segwit-related hashes to be precalculated for non-segwit transactions, preventing [the potential for IBD slowdown](https://github.com/bitcoin/bitcoin/pull/21702#pullrequestreview-859718084), and not requiring complicated workarounds. I believe the only cost would be adding 70 witness bytes to txs in the "congestion control" use case, however that use case doesn't seem viable in the first place to me.

(Strictly, "TXHASH EQUAL" is perhaps an extra witness byte compared to "CTV", though "TXHASH EQUALVERIFY" is the same as "CTV DROP". "[sig] TXHASH [p] CSFS" is probably a byte cheaper than APO, since you add "TXHASH" but save both a SIGHASH byte and a public key version byte. TXHASH also has the benefit that you can just run the script "TXHASH" through a script interpreter to get the TXHASH you want to commit to, rather than having to construct it out of band)

Beyond that, the exploration into ln-symmetry has been both fairly preliminary (to the best of my knowledge, nobody has been able to reproduce Greg's results; I've tried, and I believe cguida has as well -- ie, the "verify" part hasn't succeeded in general). It also used APO, the annex, and some custom relay rules, not CTV or CSFS or the current TRUC/ephemeral anchor model, so even if it did provide good evidence that eltoo was possible with APO, someone would still need to do the work to redesign it for CTV/TXHASH/CSFS before it was good evidence for that approach. I suspect TXHASH, CAT and CSFS would actually be a good combination here, as that would allow for a single signature to easily commit to both the tx and publication of some data while minimising wasted bytes and without needing to allow relay of random data in the annex. (Doing an adaptor signature rather than forcing publication of data would be more efficient on-chain, but seems to still be hard to convert from a whiteboard spec to an actual implementation)

Unfortunately, eltoo/ln-symmetry doesn't seem to be a very high priority in the lightning space from what I can see -- eg, see the [priorities from LDK](https://x.com/TheBlueMatt/status/1859070516956389704).

I think having both CAT and CSFS would make implementing PTLCs a bit easier -- you could replace "<preimage> SIZE 32 EQUALVERIFY SHA256 <x> EQUALVERIFY" with "<y> SIZE 32 EQUALVERIFY <x> TUCK CAT SWAP DUP CSFS" where "y" is the the s-part of the signature, calculated as preimage/(1+H(x,x,x)). Not simple per se, but separates out the PTLC calculation for the signing of the tx as a whole (avoiding the need for adaptor signatures), and only needs to be calculated once, not once per channel update. I can't see a way to get the same result from just CSFS alone.

 * "I am obviously biased here, but CTV is a game-changer for Ark"

When I looked into this, [as far as I could tell](https://x.com/ajtowns/status/1856635064941166753) Ark has been built based on the elements introspection opcodes and on pre-signed transactions, but there has been no practical exploration into actually using it with CTV. For something that was announced as being implementable either with BIP 118 (APO) or BIP 119 (CTV), never having actually done that comes across as a huge red flag for me, particularly when people are willing to implement crazy things like the "purrfect" vaults.

 * "Protocols involving DLCs, which have been growing in popularity, can be significantly simplified using CTV."

Doesn't having CSFS available on its own give you equally efficient and much more flexible simplifications of DLCs? I think having CAT available as well would probably also be fairly powerful here.

 * "BitVM would be able to drastically reduce its script sizes by replacing their current use of Lamport signatures implemented in Script by simple CSFS operations"

I think CAT would also be a significant win here.

 * "Very limited vaults have been built with CTV."

I don't believe the CTV-only "vaults" are at all interesting or useful in practice; I'm not sure if adding CSFS into the mix changes that in any way. Fundamentally, I think you want some ability to calculate an output scriptPubKey as "back to cold storage, or spend here after some delay", and CTV and the like only give the "spend here" part of that. Features like that are very complicated, and are very poorly explored. I think [they're worth exploring](https://delvingbitcoin.org/t/flexible-coin-earmarks/1275), but probably not worth rushing.

[quote="stevenroose, post:1, topic:1509"]
While no one will deny that covenants are a useful tool to build a wide variety of second-layer protocols, I feel that there is still hesitancy in the wider bitcoin user community on how the merits of the functionality covenants and second-layer protocols enable outweigh the risks of introducing them.
[/quote]

I mean, personally [I have denied that covenants are a useful tool](https://gnusha.org/pi/bitcoindev/20220719044458.GA26986@erisian.com.au/) and continue to do so. Covenants are a bad idea, and misusing that term to also cover things that are useful was also a bad idea. Jeremy may have had a good excuse for making that mistake in 2019 or so, but there's no reason to continue it.

If you're seriously trying to establish consensus about activating CTV and CSFS simultaneously, I would expect the first step would be to revise the CTV BIP so that its motivation/rationale are actually consistent with such an action.

[quote="stevenroose, post:1, topic:1509"]
This is why I took on the work to spec out and implement the [OP_TXHASH opcode](https://covenants.info/proposals/txhash/) according to Russel O’Connor’s ideas. While TXHASH as it stands is far from being mature enough to be deployed, it can function as an upgrade path for the similar but more restricted opcode [OP_CTV](https://covenants.info/proposals/ctv/).
[/quote]

There's no fundamental reason for TXHASH to be particuarly more flexible than CTV; it could be implemented precisely as "push the BIP-119 hash for this tx onto the stack". If you wanted slightly more flexibility, you could use a SIGHASH-like approach where there's a handful of values that will hash different parts of the tx, which could also precisely cover all the BIP-118 sighash values. It's not immediately clear to me what variants would actually be useful though; I think having CAT available would cover most of the cases where committing to the OP_CODESEPARATOR position or the script being executed as a whole would be very useful though, which are the main differences between the two APO variants. I don't think upgradability considerations here are even particularly necessary; just introducing OP_TXHASH2 in future seems roughly fine. (For general tx introspection, an OP_TX that doesn't automatically hash everything is perhaps more useful, but general tx introspection is a much wider design space)

[quote="stevenroose, post:8, topic:1509"]
I find is quite disingenuous when people point to the lack of implementations as an argument against a protocol upgrade. First of all, there were 0 implementations of taproot when it was deployed, but more generally, we cannot expect our developer ecosystem to be building on features we are still pondering to even add or not. Most of us can’t afford to spend months building something that in the best case will have to be shelved for another two years and worst case will be thrown out entirely.
[/quote]

So much for "I won't be .. pointing fingers", I guess? In any event, I would personally argue this was a serious flaw in how we deployed taproot, and one that we shouldn't repeat. While we did have some practical experimentation (the [optech taproot workshops](https://bitcoinops.org/en/schnorr-taproot-workshop/) explored schnorr, alternative tapscripts, and degrading multisig eg), and certainly had plenty of internal tests, that was still pretty limited. In particular, there were a variety of problems we didn't catch particularly early, but likely could have with more focus on "does this work in practice, or just in theory?":

 * going from an abstract design for musig to an actual implementation was quite complicated, and perhaps would have been simplified if the x-only/32-byte point changes could have been reconsidered. The optech workshops predated this change, so didn't have a chance to catch the problem; and the taproot review sessions afterwards were mostly focussed on the theory, with any failures at not being able to actually use the features in practice not raising huge alarms.
 * Neutrino had a [bug](https://gnusha.org/pi/bitcoindev/CAO3Pvs-Jazo27Vmi3Rforke++T5rkboS=2PK9CSSTtCk4enF2w@mail.gmail.com/) regarding compact block filters impacting blocks with taproot spends, that was only discovered a little over a week before taproot activated on mainnet.
 * Likewise, there were [multiple](https://bitcoinops.org/en/newsletters/2022/10/19/#block-parsing-bug-affecting-btcd-and-lnd) [bugs](https://bitcoinops.org/en/newsletters/2022/11/09/#block-parsing-bug-affecting-multiple-software) in btcd related to parsing of transactions that were only caught long after taproot was in active use. Really, these were holdover bugs since segwit, however taproot at least made transactions that trigger the first bug relayable. Either way, more practical experimentation on public test networks, rather than just in our internal test framework, would likely have allowed these bugs to have been discovered earlier, and potentially fixed prior to it causing downtime for users on mainnet, with risk of loss of funds.

From a social point of view this outcome probably shouldn't be surprising -- there were plenty of people pushing for taproot to be activated ASAP, and nobody saying that we should slow down and spend more time testing things and demonstrating that they're useful.

As far as "most of us can't afford to spend months building" goes, there's two things you should consider. One is that with open source, you only need one person to build something, after which point everyone can reap the benefits. If you can't find even one person willing to spend time on a moonshot project, it's probably not actually much of a moonshot. Second, is that, [reportedly](https://x.com/tierotiero/status/1899406106851463530) these things only take a few hours, not months.


Personally, I think the biggest blocker to progress here continues to be CTV's misguided description of "covenants", and its misguided and unjustified concern about "recursive covenants". Particularly given those concerns are incompatible with co-activation of CSFS, I would have thought a simple first step would be updating BIP 119 to remove/correct them, which might then also allow some rational discussion of CAT and similar features. I and others have already tried talking to Jeremy both publicly and privately about those issues with no success, but maybe you'll have some. Otherwise, discarding BIP 119 entirely and starting from scratch with an equally expressive and more efficient simplified TXHASH BIP could be a good option.

-------------------------

1440000bytes | 2025-03-12 03:32:38 UTC | #15

[quote="ajtowns, post:14, topic:1509"]
When I looked into this, [as far as I could tell](https://x.com/ajtowns/status/1856635064941166753) Ark has been built based on the elements introspection opcodes and on pre-signed transactions, but there has been no practical exploration into actually using it with CTV. For something that was announced as being implementable either with BIP 118 (APO) or BIP 119 (CTV), never having actually done that comes across as a huge red flag for me, particularly when people are willing to implement crazy things like the “purrfect” vaults.
[/quote]

https://codeberg.org/ark-bitcoin/bark/src/branch/ctv

-------------------------

ajtowns | 2025-03-12 03:38:25 UTC | #16

It's great that Steven has apparently been working on this in the last hour, but maybe it's even better to let the code bake a little while before trying to use it in a debate.

-------------------------

stevenroose | 2025-03-12 14:50:44 UTC | #17

[quote="ajtowns, post:14, topic:1509"]
CTV+CSFS isn’t equivalent to APO, it’s somewhat more costly by requiring you to explicitly include the CTV hash in the witness data. The [TXHASH](https://gnusha.org/pi/bitcoindev/CAMZUoK=pkZuovtifBzdqhoyegzG+9hRTFEc7fG9nZPDK4KbU3w@mail.gmail.com/) approach is (in my opinion) a substantial improvement on that.
[/quote]

I think it's fair to call something equivalent even if it's a little more costly but achieves the same functionality. (Of course I wouldn't go as far to make the same argument for things like CAT where you have to dump the entire tx on stack including several dozen opcodes.) A better argument would be that CTV+CSFS can only emulate APO|ALL and not the other APO flags. Though it seems that the APO|ALL variant of APO has the most interest.

[quote="ajtowns, post:14, topic:1509"]
The TXHASH approach would be a taproot-only solution (as is CSFS), which would have the benefit of both simplifying the change (removing policy considerations for relay of bare-CTV txs, not having to consider the CTV in P2SH being unsatisfiable)
[/quote]

The TXHASH BIP explicitly also specifies to enable CHECKTXHASHVERIFY in legacy and segwit context and outlines how hashes should be calculated for those contexts. This is specifically done so to enable "bare CTXHV" which is about a 70 witness bytes saving over p2tr and 30-something bytes over p2wsh. Having implemented TXHASH, I would definitely not say that it "simplifies the change". The difference in both technical debt and potential for bugs is an order of magnitude bigger for TXHASH than for CTV. (Not to say that I don't think TXHASH would be worthwhile, but I will definitely say that it has not received the attention I had expected, so I would definitely not want to put it on the table anytime soon.)

When it comes to IBD slowdown, I believe that was before the CTV cache was made lazy? All TXHASH caches are already implemented to be lazy, so they shouldn't affect IBD or any tx validation without TXHASH opcodes.

[quote="ajtowns, post:14, topic:1509"]
It also used APO, the annex, and some custom relay rules, not CTV or CSFS
[/quote]

It actually explicitly mentions some additional benefits from CTV on top of its use of APO:

[quote="instagibbs, post:1, topic:359"]
CTV(emulation) ended up being useful! It removed the necessity of round-trips from the payment protocol, allowing for “fast forwards” that are extremely simple, and would likely reduce payment times if widely adopted.
[/quote]

[quote="ajtowns, post:14, topic:1509"]
Unfortunately, eltoo/ln-symmetry doesn’t seem to be a very high priority in the lightning space from what I can see – eg, see the [priorities from LDK](https://x.com/TheBlueMatt/status/1859070516956389704).
[/quote]

An argument could be made that the lack of prioritization of eltoo in Lightning land is no proxy for a lack of enthusiasm for it. Lightning is not a moonshot industry, it has actual products and users, so they have to be pragmatic and build what can improve user experience tomorrow and not in several years.

[quote="ajtowns, post:14, topic:1509"]
When I looked into this, [as far as I could tell](https://x.com/ajtowns/status/1856635064941166753) Ark has been built based on the elements introspection opcodes and on pre-signed transactions, but there has been no practical exploration into actually using it with CTV.
[/quote]

Like already mentioned, I did the experiment [last night](https://x.com/stevenroose3/status/1899651611385057519) and it was remarkably straightforward.

[quote="ajtowns, post:16, topic:1509, full:true"]
It’s great that Steven has apparently been working on this in the last hour, but maybe it’s even better to let the code bake a little while before trying to use it in a debate.
[/quote]

I wouldn't go so far as to dismiss this experiment based on its age. The code is obviously not to be deployed in the state it is in, because I hackishly used conditional compilation to swap out our musig-cosigned transaction trees with ctv-based trees and for deployment you'd just pick one and not have the conditional compilation.

But we have been working on this implementation for over 6 months, it is working on bitcoin's vanilla signet, we have ample integration tests that test various unilateral exit scenarios and all of these are passing for the ctv-based trees. From the looks of it, it seems like we could net remove about 900 lines of code if we would delete the cosigning code in favor of ctv and reduce our own round protocol to just two rounds of interactions instead of three. (Not to speak of the bandwidth improvement for not having to pass around signing nonces and partial signatures.)

Anyway, while this experiment shows that ctv is a practical primitive to use as a developer, I think it is still mostly off topic and I'd prefer to stick to the topic at hand.

[quote="ajtowns, post:14, topic:1509"]
Doesn’t having CSFS available on its own give you equally efficient and much more flexible simplifications of DLCs? I think having CAT available as well would probably also be fairly powerful here.
[/quote]

I would assume so, but they would make the protocols a lot less "discreet", I guess. The whole "hide the tweak inside the output key" feature is I think one of the main draws to DLCs and why they're called "discreet" log contracts. Obviously not arguing against CAT adding a lot of value in most protocols :slight_smile:

-------------------------

instagibbs | 2025-03-12 15:36:31 UTC | #18

[quote="ajtowns, post:14, topic:1509"]
suspect TXHASH, CAT and CSFS would actually be a good combination here, as that would allow for a single signature to easily commit to both the tx and publication of some data while minimising wasted bytes and without needing to allow relay of random data in the annex.
[/quote]

Yes, getting the tx hash you want on the stack, sticking it to whatever data you want published onto it, and signing it does the trick.

[quote="ajtowns, post:14, topic:1509"]
(Doing an adaptor signature rather than forcing publication of data would be more efficient on-chain, but seems to still be hard to convert from a whiteboard spec to an actual implementation)
[/quote]

This probably needs to be rewritten on a whiteboard because I don't remember it saving much and being much harder to reason about in multiparty settings vs 32 WU. I'm guessing this is getting a bit off topic though :) 

[quote="ajtowns, post:14, topic:1509"]
Not simple per se, but separates out the PTLC calculation for the signing of the tx as a whole (avoiding the need for adaptor signatures), and only needs to be calculated once, not once per channel update. I can’t see a way to get the same result from just CSFS alone.
[/quote]

One signature per PTLC also falls out of using APO/(CTV/TXHASH+CSFS), just to be clear. Not needing adaptor signatures (and binding it to a specific sighash?) is kinda cool.

[quote="ajtowns, post:14, topic:1509"]
Otherwise, discarding BIP 119 entirely and starting from scratch with an equally expressive and more efficient simplified TXHASH BIP could be a good option.
[/quote]

Current TXHASH BIP has "sepcial" TxFieldSelector options, with 0-length argument being CTV mode, and 1-byte allowing BIP118 emulation. I think it's pretty straight forward to investigate that direction.

[quote="stevenroose, post:17, topic:1509"]
This is specifically done so to enable “bare CTXHV” which is about a 70 witness bytes saving over p2tr and 30-something bytes over p2wsh.
[/quote]

I've found this use-case the least compelling part, especially at the cost of having to reason about legacy scripting. If it's really important one idea I'd suggest it be included it as a new segwit output type as a separate feature that lives and dies in a proposal on its own merits. I ended up doing this with P2A for the exact same two reasons.

-------------------------

stevenroose | 2025-03-12 18:14:51 UTC | #21

My purpose with this thread is to avoid doing an independent activation client which will undoubtedly upset people, but instead hear everyone out and see if we can gather enough momentum to get some version of these changes deployed through the regular Core release process.

-------------------------

jamesob | 2025-03-12 19:51:09 UTC | #23

[quote="ajtowns, post:14, topic:1509"]
I don’t believe the CTV-only “vaults” are at all interesting or useful in practice
[/quote]

I know firsthand that a large custodian would make use of CTV to replace presigned transactions specifically for custodial operations.

Beyond that, as you should recall from your `VAULT` days, CTV (or an equivalent) is a necessary prerequisite to doing any kind of "better" vault. It's tablestakes. Rob Hamilton recently [substantiated the industry demand for good vaults, using `VAULT` or similar](https://x.com/Rob1Ham/status/1897781338796966103), and once again I can corroborate firsthand there.

So to downplay the demand for vaults is very strange to me. It's an obvious use that people keep asking for in various forms, and CTV is a required primitive.

The best argument you can make against CTV on this count is that TXHASH's CTV mode would serve the same purpose, but scrutinizing the TXHASH implementation for validity is probably an order of magnitude harder than for CTV's given [its complexity](https://github.com/stevenroose/bitcoin/blob/599020daaa3deab922229adf41f5021ee9f4ca3b/src/script/txhash.cpp).

[quote="ajtowns, post:14, topic:1509"]
Fundamentally, I think you want some ability to calculate an output scriptPubKey as “back to cold storage, or spend here after some delay”, and CTV and the like only give the “spend here” part of that.
[/quote]
I'm not understanding you here - CTV vaults, and [my implementation in particular](https://github.com/jamesob/simple-ctv-vault), allows you to either do a time-delayed spend, or sweep immediately to cold. That much is shown in the first state diagram on the page.

[quote="ajtowns, post:14, topic:1509"]
I believe the only cost would be adding 70 witness bytes to txs in the “congestion control” use case, however that use case doesn’t seem viable in the first place to me.
[/quote]
I'm not sure what your basis for saying this is. Aside from the "thundering herd use" potential, congestion control can be used by miners to compress payouts in the coinbase transaction during times of elevated feerates. I spoke to LukeJr about this last night, who runs Ocean Mining, and he said that miners would make use of this - although possibly on the basis of firmware limitations rather than fee smoothing.

The point stands that there are many uses for what's referred to as "congestion control," and to dismiss them all casually seems presumptuous.

The "bare legacy" mode for CTV is especially important in these cases because it's the most succinct first-stage commitment (`<32byte-hash> OP_CTV`) that can serve to lock in a series of `n` payments on bitcoin.

-------------------------

jamesob | 2025-03-12 19:57:23 UTC | #24

Oh, and I forgot to mention on the subject of vaults/presigned-txns: I've spoken to two Blockstream engineers in the past couple months who both say that CTV could be used to drastically improve the Liquid timelock-fallback script that requires coins to be rolled on a periodic basis. This may apply to Liana as well.

-------------------------

instagibbs | 2025-03-12 21:24:45 UTC | #25

[quote="jamesob, post:24, topic:1509"]
I’ve spoken to two Blockstream engineers
[/quote]

For the sake of discussion getting these things on the record by the claimants somewhere public with details is helpful. I think I know the upsides/downsides of the approach but without details it's hard to engage.

-------------------------

AntoineP | 2025-03-12 22:34:34 UTC | #26

[quote="jamesob, post:24, topic:1509"]
that CTV could be used to drastically improve the Liquid timelock-fallback script that requires coins to be rolled on a periodic basis. This may apply to Liana as well.
[/quote]

I am skeptical of this claim.

First of all, it's vague. What does "drastically improve" even means, concretely? Following [Liquid's documentation](https://docs.liquid.net/docs/technical-overview#emergency-recovery-procedure), i assume you are talking about the peg-in script. TL;DR for everyone here: Bitcoin users who want to onboard to the Liquid sidechain can pay to this Script. Coins sent to this Script by users onboarding may later be spent by 2/3 of the Liquid watchmen (custodians) when a Liquid users wants to peg-out to Bitcoin. This Script contains a timelock clause, such that those coins may be spent using three emergency keys [after 4032 blocks](https://help.blockstream.com/hc/en-us/articles/900001408623-How-does-Liquid-Bitcoin-L-BTC-work).

Using such a timelocked spending path directly in the receiving Script presents the same trade-off as for Liana: if they don't want the emergency recovery to become available then the Liquid watchmen need to spend every coin within 28 days (4032 blocks) of its reception. This presents an inescapable trade-off between funds availability in case the recovery is needed and the security margin to avoid the weaker spending path from being available unless absolutely necessary.

More [interesting covenants provide a way out of this](https://delvingbitcoin.org/t/using-op-vault-for-recovery/150?u=antoinep), by delaying the timelock to only be triggered through a second stage similar to that of vault constructions. I think claiming `CTV` can achieve as much is misleading, as in this case you would have to commit to the second-stage transaction at the time of receiving the coins. Since the receiver crafts the address to request funds on, this means the Liquid watchmen would need to both 1) know the amount before giving away the address and 2) trust that the user will for sure use the **exact same amount** they said they would in the previous round of communication, or the funds may be **locked forever** or have the excess burned to fees. In addition they need to trust the address will never be reused with a different in the future, something that is infamously hard to get users to do.

This scheme would probably do (much) more harm than good, and this is why i'm skeptical either Liquid or Liana engineers would ever put such a footgun in the hand of their users. Therefore, i do not think it is a valid motivation for a CTV soft fork.

-------------------------

ariard | 2025-03-13 00:28:28 UTC | #27

### Lightning Eltoo

ajtowns:
> "CTV+CSFS isn’t equivalent to APO, it’s somewhat more costly by requiring you to explicitly include the CTV hash in the witness data. The TXHASH approach is (in my opinion) a substantial improvement on that.”

stevenroose:
> "I think it’s fair to call something equivalent even if it’s a little more costly but achieves the
> same functionality. (Of course I wouldn’t go as far to make the same argument for things like CAT where you have to dump the entire tx on stack including several dozen opcodes.) A better argument  would be that CTV+CSFS can only emulate APO|ALL and not the other APO flags. Though it seems that  the APO|ALL variant of APO has the most interest.”

I don't believe we can say things are equivalent when the marginal on-chain witness cost can fluctuates in the range of two-digits bytes. In Lightning, we already have to trim the outputs
out of the commitment transaction, if the outputs scriptpubkeys + claiming input is superior to
the ongoing mempool feerates (very imperfect heuristic...). This is a safety issue if you go to open LN chan with a miner, that you can never be sure of.

Going for the more expensive Eltoo, i.e the one where the script stack has to provide `<pubkey>` `<message>` `<signature>`, where message size is equal to 32 bytes those 32 bytes compared to the `ANYPREVOUT` sighash approach that might make some chan unable to uncooperatively force-close, at a time of fee spikes.

Note this concern on marginal channel or off-chain payment is something that very likely affects Ark too. It's even hard to compare the cost of a LN chan marginal payment vs the cost of a Ark marginal payment, as with ARK you have an ASP and you have to come with some probabilistic estimations for the interactivity of the ASP.

If my memory is correct, the efficiency approach of logically equivalent primitive was already discussed in the `OP_CHECKMERKLEBRANCHVERIFY` vs check-if-this-is-a-PTR2 templated
approach (i.e BIP341).

### Discreet Log Contracts

ajtowns:

>  "Doesn’t having CSFS available on its own give you equally efficient and much more flexible
> simplifications of DLCs? I think having CAT available as well would probably also be fairly 
> powerful here".

See this [thread](https://github.com/bitcoinops/bitcoinops.github.io/pull/806) on Optech Github for the trade-offs on the usage of CTV of Discreet Log Contracts.

tl;dr: With adding a hash for each CTV outcome, there is a logarithmic growth of the witness script size (i.e `<hash1_event>` `<OP_CTV>` `<hash2_event>` `<OP_CTV>`), if the bet is logarithmic in its structure. Evaluating what primitive is the best for a Discreet Log Contract is very function of (1) what is the marginally value "bet on" and (2) what is the probabilistic structure of the bet (i.e are you betting on a price where equal chance among all outcomes or a bet sport where ranges scores can be approximated).

### Push-Based Approach Templating

stevenroose:
> The TXHASH BIP explicitly also specifies to enable CHECKTXHASHVERIFY in legacy and segwit
> context and outlines how hashes should be calculated for those contexts. 

See the old Johnson Lau [idea](https://github.com/jl2012/bips/blob/ce4a6980c89859b8f5c9074b6f219f973e4d9128/bip-0ZZZ.mediawiki) with `OP_PUSHDATADATA` for another variant of push-approach.

My belief on the push-vs-implicit-access-with-sigs digest, it's all up to wish semantic you wish to check on the spending transaction of your constrained UTXO, and if it's not a templated approach, what is shortest path for stack operations among a N number of transactions fields.

Of course, there can be numerous low-level details of primitive implementation to make that more efficient, like bitvector, special opcodes to push target input or assumptions on the "most-likely" fetched transaction fields.

I don't know if it's a programming model we wish to move towards wish...This would start to be very likely to ASM where you have to program CPU registers at the bit-level. If you think Bitcoin Script programming is already low-level and error-prone, this is an order of magnitude worst. Complexity for the use-case programmer at the benefit of more on-chain efficiency.

### On the Taproot Rush

ajtowns:
> So much for “I won’t be … pointing fingers”, I guess? In any event, I would personally argue this was a serious flaw in how we deployed taproot, and one that we shouldn’t repeat.

I share the opinion, that we could have spent more time doing experimentations of the use-case enabled by Schnorr / Taproot. There was a [research page](https://github.com/BlockstreamResearch/scriptless-scripts/blob/fd2000d2c30cc8d9125ecd85b0dc14edf32266a3/md/multi-hop-locks.md) at the time listing all the ideas enabled by Schnorr. I did an [experiment](https://github.com/lightningdevkit/rust-lightning/issues/605) to implement PTLC+DLC in early ~2020 for Discreet Log Contract. The learning I’ve come from it that we would have to seriously re-write the LN state machine. As far as I can tell, this has been confirmed by the more recent research of other LN folks.

On the more conceptual limitations of the Taproot, the lack of commitment in the control block of the oddness of an internal pubkey is a limitation to leverage a Schnorr signature as mutable cryptographic accumulator for payments pools. This limitation was known before the activation of Taproot, and it has been discussed few times on the mailing list and [documented](https://bitcoinops.org/en/newsletters/2021/09/15/#covenant-opcode-proposal) by Optech.

On the merge of the Taproot feature, let's remember that [the PR implementing it](https://github.com/bitcoin/bitcoin/pull/19953) was merged the latest day of the feature freeze for 0.21.0, which I don’t believe I was the only one to find it was a bit of a rush…

One can see the names who have ACKed the merged commit at the time on the Github pull
request, as I think seriously reviewing and testing code for a consensus change is always more expensive than talking about it:
- instagibbs
- benthecarman
- kallewoof
- jonasnick
- fjahr
- achow101
- jamesob (post-merge)
- ajtowns (post-merge)
- ariard (post-merge)
- marcofalke (post-merge)

### On the usage of the "Covenant” word

ajtowns:
> Personally, I think the biggest blocker to progress here continues to be CTV’s misguided
> description of “covenants”, and its misguided and unjustified concern about “recursive covenants”.

To be fair here the usage of the word covenant in Bitcoin is not Jeremy's initiative. I think it comes with Gmax "[CoinCovenants using SCIP signatures, an amuingly bad idea](https://bitcointalk.org/index.php?topic=278122.0)” bitcoin talk org article in 2013, and it was in the aftermath also used by folks like Roconnor, Johnson Lau, Roasbeef or even by myself as early as 2019 when OP_CTV was still called OP_SECURETHEBAG.

I'm not aware if Satoshi herself / himself has used the word covenant in its public writing. However the idea to use Script for many use-cases beyond payments, that’s Satoshi, there is quote in the sense somewhere talking about escrow and having to think carefully the design of Script ahead.

The problem of "recursive covenants" is also layout in Gmax's article of 2013, as a basic question, if malicious "covenants" could be devised or thought at lot, and it was abundantly commented at the time on bitcoin talk org.

### On the lack of enthusiasm for Lightning / Eltoo

1440000bytes:
> We see an arrogance and non sense being repeated here by developers who are misusing their reputation in the community. Some of these developers have no reasons to block CTV and been writing non sense for years that affects bitcoin.

To bring more context on why there is a lack of enthusiasm for Eltoo Lightning, during the year of 2022, Greg Sanders have worked on a fork of core-lightning with eltoo support and this was reviewed multiple times by AJ Towns and myself.

This is during the review of this Lightning-Eltoo and considering hypothetical novel attacks on eltoo Lightning ("Updates Overflow" Attacks against Two-Party Eltoo ?"), that I found was is (sadly) known today as Replacement Cycling Attacks.

This is for a very experimental point, if you believe that reviewing complex Bitcoin second-layers is shamanism or gatekeeping. I still strongly believe that end-to-end PoC'ing, testing and adversarial review is a good practice to get secure protocols, and no not all second-layers issues can be fixed "in flight" like "that", especially if the fixes commands themselves for serious engineering works at the base-layer (e.g better replacement / eviction policy algorithms).

### Ark + CTV

stevenroose:
> But we have been working on this implementation for over 6 months, it is
> working on bitcoin’s vanilla signet, we have ample integration tests that
> test various unilateral exit scenarios and all of these are passing for
> the ctv-based trees.

If I'm understanding correctly, Ark is argued as an example of a near-production or production-ready use-case that would benefit from a hash-chain covenant like CTV. Given Ark is relying on a single "blessed party" the ASP, I'm still curious how an ASP client can be sure that he can redeems its balance on-chain in a collaborative on-chain.

Namely, how do you generalize the fair exchange of a secret to a N number of parties, where among the set of N there is blessed party M, how do you avoid collusion between the N-1 parties + the blessed party M against the N party. Fair exchange of secret is quite studied the 90's distributed litterature. There were papers also few years ago on Lightning, analyzing unsafe update mechanism for many parties.

Of course, you can do a congestion control tree embedded in the on-chain swap UTXO coming the ASP, but now there is no efficiency gain remained for the usage of CTV (nVersion / nLocktime fields penalty for each depth of the tree).

### Vault + CTV / better primitives

jamesob:
> Beyond that, as you should recall from your VAULT days, CTV (or an equivalent)
> is a necessary prerequisite to doing any kind of “better” vault. It’s tablestakes.
> Rob Hamilton recently substantiated the industry demand for good vaults, using
> VAULT or similar, and once again I can corroborate firsthand there.

The issue, with vault, is of course dynamics fees for time-sensitive transactions, and if I remember correctly the emergency path, which is a time-sensitive path you have to be sure dynamic fees works well. Even if you pre-sign at some crazy rate, there is no guarantee that you won't have a nation-state sponsoring hacking group going to engage in a feerate race to delay the confirmation (e.g costless bribes to the miners), until the compromised withdrawal can confirm.

This is not paranoia, if one follows smart contract exploits in the wider cryptocurrencies world (free to check rekt.news), you often see hacks in the $100M - $500M range. So an attacker going to burn 10% of the targeted value in miner bribing fees do not seem unrealistic or unreasonable to me. If you assume that attacker has already keys for the withdrawal or unvault target "hot” wallet.

To be frank, fixing dynamics fess, it's very likely going to be someting in the line of “[fee-dependent timelocks](https://bitcoinops.org/en/newsletters/2024/01/03/#fee-dependent-timelocks)". And here everyone is free to believe or not (don't trust, verify), under all technical info and hypothesis I'm aware off, eventually those are not going to be simple consensus changes…

Now, on the more precise question of CheckTemplateVerify and its usage for vaults, the best piece of information I'm aware of for the key-sig-vs-hash-chain is based on this Phd Thesis, section 4 "[Evolving Bitcoin Custody](https://arxiv.org/pdf/2310.11911)”.

Of course, while CheckTemplateVerify introduces _immutability_ of a chain of transactions, where intermediary vault transactions do not have to be pre-signed and corresponding privates key deleted, there is still the issue of key ceremony to be dealt with. After reorg-delay, though this a novel property if one goes to design bitcoin second-layers.

If you're software engineer with know-how on the difference between userspace and kernelspace or familiarity with core Internet protocols (...yes all those talks on bitcoin consensus changes can be very technical in the raw sense), keys ceremonies and overall corresponding operational guidelines can be an incredibly hard thing to get right. All depends the threat model considered, though it’s hard thing to do, and hard to do it repeatedly in production.

So what is key ceremony for bitcoin vaults ? This is dealing with the transition of the cold wallet to the hotter wallets, though for bitcoin this is not a only a "blind signature", it's verifying that the spent utxo exists, that the unvaulting outputs `scriptPubkeys` are valid or at byte-for-byte equal to the one that are going to be verified by the Script by bitcoin full-nodes at run-time.

As people who are familiar with the [Validating Lightning Signer](https://vls.tech/), series of custom checks and the situation there are to have re-write a LN state machine embeddable for the constraint of a secure enclave, being sure that your vault is the "correct" vault in production is a bit more complex safety-wise than is this a P2WSH yes or no.

So I strongly believe the bottleneck we have to evaluate a CTV-enabled vault is a vault proof-of-concept specified enough that all the logic of the vault can be described with chain headers, UTXO proofs and outputs descriptors that they can be given through a carefully-designed interface to a HW or secure enclave and have the vault setup verification done there.

**Do we have any bitcoin HW vendors or secure enclave vendors**
**ready-to-extend their interfaces for a simple [2-steps](https://github.com/jamesob/simple-ctv-vault) CTV vault protocol ?**

Not only with output script support though also with any efficient proving of the UTXO set, which can be challenging programming-wise as secure enclave RAM and cache memory is limited, by design. And as far as I researched so far, constrained templating like CTV do not comes with [tx-withhold risks](https://blog.bitmex.com/txwithhold-smart-contracts/) and do not alter the UTXO model, though my thanks if you prove me wrong here.

-------------------------

sanket1729 | 2025-03-13 01:00:11 UTC | #28

[quote="instagibbs, post:25, topic:1509"]
For the sake of discussion getting these things on the record by the claimants somewhere public with details is helpful. I think I know the upsides/downsides of the approach but without details it’s hard to engage.
[/quote]

I spoke with @jamesob about this. I am no longer affiliated with Blockstream, but  I do feel that liquid script can benefit from CTV. I worked on research team, maybe @stevenroose and @instagibbs who worked more closely with liquid can correct me if I am in the wrong here. 

[quote="AntoineP, post:26, topic:1509"]
1) know the amount before giving away the address and 2) trust that the user will for sure use the **exact same amount** they said they would in the previous round of communication, or the funds may be **locked forever** or have the excess burned to fees. In addition they need to trust the address will never be reused with a different in the future, something that is infamously hard to get users to do.
[/quote]

You’re right. A straightforward implementation would be highly prone to issues for the same reasons you mentioned. However, this can be addressed as follows: users deposit funds using a standard liquid peg-in script. These peg-in funds are then consolidated into a separate CTV address managed by the watchmen. This shifts the responsibility of correct CTV usage from users to the liquid engineering team.

While this approach requires careful engineering, the potential fee savings could make it well worth the effort.

I don't think Liana can achieve something similar since it is non-custodial. In this case, once the funds are in the watchmen's custody, they can optimize spending as needed.

Overall, I agree with the sentiment that general-purpose vaults are better and less error-prone for this use case, for the same reasons you listed. However, as mentioned above, CTV remains useful on its own for avoiding recurring fees.

-------------------------

ajtowns | 2025-03-13 05:18:58 UTC | #29

[quote="ariard, post:27, topic:1509"]
To be fair here the usage of the word covenant in Bitcoin is not Jeremy’s initiative. I think it comes with Gmax "[CoinCovenants using SCIP signatures, an amuingly bad idea](https://bitcointalk.org/index.php?topic=278122.0)”
[/quote]

Greg used the term precisely and correctly -- his post describes taking a general [zero-knowledge proof](https://bitcointalk.org/index.php?topic=277389.0) feature and using that to produce actual covenant constructions where a coin and all possible spends of that coin are permanently constrained in a particular way, creating a burn address that allows the burnt coins to be burnt again and again:

> A particular sort of rule could take the form of requiring any output scriptpubkey to be of the form `THIS_VALIDATION_KEY && {whatever rules you want}` and by doing so you have effectively created a coin which is forever subject to a [covenant](http://en.wikipedia.org/wiki/Covenant_%28law%29) which will run with the coin and forever constrain the use of it and its descendants degrading and its fungibility.

-------------------------

reardencode | 2025-03-13 14:45:03 UTC | #30

On not ignoring the importance of script cost for contested settlement:

[quote="ariard, post:27, topic:1509"]
Note this concern on marginal channel or off-chain payment is something that very likely affects Ark too. It’s even hard to compare the cost of a LN chan marginal payment vs the cost of a Ark marginal payment, as with ARK you have an ASP and you have to come with some probabilistic estimations for the interactivity of the ASP.
[/quote]

I've made a comparison of many alternatives for developing eltoo. Briefly, APO+CTV+standardized_annex is the most efficient of known alternatives. Compared to this:

| Method | uncontested | once contested |
| --- | --- | --- |
| LNHANCE | +8vB | +16vB |
| @instagibbs APO+annex | +16vB | +32vB |
| CTV+CSFS | +58vB | +109vB |

As in other protocols CTV consistently reduces cost and sigops even when used with other mechanisms.

edit: corrected some figures and made into a table

-------------------------

stevenroose | 2025-03-13 16:40:26 UTC | #31

Eh, I seem to somehow have missed the second half of AJ's first response. My bad.

[quote="ajtowns, post:14, topic:1509"]
So much for “I won’t be … pointing fingers”, I guess? In any event, I would personally argue this was a serious flaw in how we deployed taproot, and one that we shouldn’t repeat.
[/quote]

I wasn't trying to point fingers at whomever was involved in the taproot deployment. More rather trying to indicate that the bar for the taproot soft fork was met on perceived technical merit only, while today it seems that practical usage is the only bar upheld. Is this because people actually don't see technical merit in CTV and CSFS and hence want convincing by usage, or because we believe purely technical merit is no longer enough for a consensus upgrade?

[quote="ajtowns, post:14, topic:1509"]
If you can’t find even one person willing to spend time on a moonshot project, it’s probably not actually much of a moonshot. Second, is that, [reportedly](https://x.com/tierotiero/status/1899406106851463530) these things only take a few hours, not months.
[/quote]

Arguably CTV so closely resembles presigned transactions yes, one can [swap presigned txs to CTV in a few hours](https://x.com/stevenroose3/status/1899651611385057519), but that is only possible because we have spent the presigned txs project for several months while keeping the potential of swapping to CTV in mind while designing API. This strategy wouldn't apply for any project that really needs a new primitive to be built.

[quote="ajtowns, post:14, topic:1509"]
Personally, I think the biggest blocker to progress here continues to be CTV’s misguided description of “covenants”, and its misguided and unjustified concern about “recursive covenants”. Particularly given those concerns are incompatible with co-activation of CSFS, I would have thought a simple first step would be updating BIP 119 to remove/correct them
[/quote]

I'm confused. Are you talking about a technical amendment to CTV? Or are you (again, like in your recent mailing list post) whining about the Motivation section of the BIP? Excuse my French, but if your "biggest blocker" in what is trying to be a pragmatic evaluation of a technical change is some wording in a BIP's motivation section, I can't help but question whether you are actually trying to constructively participate in this conversation.

[quote="ajtowns, post:14, topic:1509"]
Otherwise, discarding BIP 119 entirely and starting from scratch with an equally expressive and more efficient simplified TXHASH BIP could be a good option.
[/quote]

FWIW, I wouldn't mind this outcome of course, I implemented TXHASH exactly because it's a more powerful version of CTV that offers a lot more flexibility. Though when I published the BIP and implementation, I barely received any feedback, so I figured there was not much interest. Given your own remarks on the topic, I suspect you also haven't read the BIP text yourself.


[quote="AntoineP, post:26, topic:1509"]
Since the receiver crafts the address to request funds on, this means the Liquid watchmen would need to both 1) know the amount before giving away the address and 2) trust that the user will for sure use the **exact same amount** they said they would in the previous round of communication, or the funds may be **locked forever** or have the excess burned to fees
[/quote]

It's true that CTV isn't suitable for a scheme where you want to present the user with an address that they can deposit any sort of funds into. However, when these transactions are managed by software, I think it's ok. Liquid could just abandon the "pegin address" concept and have wallets implement pegins internally.

We do the same with Ark: when onboarding you craft an exit tx and send it to the server for cosigning. With CTV you could do away with this round of interaction and since the amount is visible in the tx, you could also re-generate the template hash and recover funds using a mnemonic, which in the current system is impossible because loss of the cosign signature means you can't recover the exit tx.

(For completion, TXHASH is able to assert that input amount equals output amount, so that would solve this particular problem if you only use a single input. Can't do input sum amount equals output sum amount unfortunately.)

-------------------------

jamesob | 2025-03-13 17:30:22 UTC | #32

[quote="instagibbs, post:25, topic:1509"]
For the sake of discussion getting these things on the record by the claimants somewhere public with details is helpful.
[/quote]

[quote="AntoineP, post:26, topic:1509"]
I am skeptical of this claim.
[/quote]

Andrew Poelstra is the second person I spoke with about this. Today he gave me permission to affirm here that he thinks CTV could be used for this purpose with Liquid.

He writes

>  What the watchmen need is super simple. "These keys allow moving the coins to a special timelocked staging area, from which the original keys can still pull them back" It's basically a one-step vault.
> 
> In fact we literally implemented it in terms of unsigned transactions in an early version of liquid, but it was too difficult to keep them synced up and invalidated.

Given that he is short time (and a Delving Bitcoin account), he graciously allowed me to republish here.

So we have two independent attestations that CTV could be used to good effect on Liquid -- hopefully that's sufficient to put the matter to rest.

-------------------------

moneyball | 2025-03-13 17:59:58 UTC | #33

I still don't see any quantification of benefit to Liquid so seems premature for you to put the matter to rest.

-------------------------

jamesob | 2025-03-13 18:25:04 UTC | #34

What has been put to rest is that two highly qualified Liquid devs have said, "yeah, we'd use CTV for Liquid."

-------------------------

moneyball | 2025-03-13 18:57:58 UTC | #35

You're the one saying you're putting things to rest. I'm curious to learn of the magnitude of the advantage.

-------------------------

ariard | 2025-03-13 20:15:10 UTC | #36

[quote="ajtowns, post:29, topic:1509"]
Greg used the term precisely and correctly – his post describes taking a general [zero-knowledge proof](https://bitcointalk.org/index.php?topic=277389.0) feature and using that to produce actual covenant constructions where a coin and all possible spends of that coin are permanently constrained in a particular way, creating a burn address that allows the burnt coins to be burnt again and again:
[/quote]

Yes, I'm familiar with the idea that you have a SNARK verifier as a replacement for the script interpreter, so you get as an input stack `<input public data>` `<verification public key>` `<signature>` `<hash_masked_tx>`, where you can assert properties on the spending tx (e.g is this output scriptpubkey size 32 bytes).

By setting the verification rule that an output redeem script must be of the form `THIS_VALIDATION_KEY && {whatever rules you want}`, in my understanding you're introducing recursivity of the verification rules.

The recursivity can be bounded (i.e using the tx nVersion field as a counter) or unbounded, by generating a correct and custom `<verification public key>`.

In my understanding, and here with in mind Roconnor's FC'17 paper and Jeremy's talk at Standford's 2017, the idea of constraining and recursively applying a set of rules on a UTXO and any of spent tx, is what probably mistakenly understood as a covenant nowadays. At the very least, I gave talks and used the term in that sense in my own writing on bitcoin covenants.

I don't disagree that "covenant" is an imprecise terminology. After re-checking the translation in my native tongue (i.e "une convention legale"), the term designates more wider legal constructions than what is understood as a "covenant" in the English real estate law. Saying contracting primitives or Script opcodes sounds indeed better.

[quote="reardencode, post:30, topic:1509"]
I’ve made a comparison of many alternatives for developing eltoo. Briefly, APO+CTV+standardized_annex is the most efficient of known alternatives. Compared to this:
[/quote]

So the original idea of Eltoo was to have a new sighash flag for the signature of state transaction, that makes them re-bindable on any previous state, removing the constraint to have to store revoked scripts / amounts, for each previous state, and opening the door to >= 2 parties off-chain constructions.

Efficiency-wise, yes I can see how you can have the chan constraint in an "anyprevout_flag" tx template in the commitment_tx output, the per-state counter in the annex fields which could be saving the signature cost size, though I'm not sure we're talking about the same data layout. And I believe you might need one more opcode to push the annex field on the stack.

[quote="stevenroose, post:31, topic:1509"]
I wasn’t trying to point fingers at whomever was involved in the taproot deployment. More rather trying to indicate that the bar for the taproot soft fork was met on perceived technical merit only, while today it seems that practical usage is the only bar upheld.
[/quote]

As I pointed out on the mailing list more powerful opcode primitives can open the door to `TxWithhold` risks, by allowing to introspect the status of another UTXO. E.g a basic tx-withhold contract would be someone promising to any miner that if target LN commitment tx is not confirmed until N, a native bitcoin bounty is paid. The N picked up can be the safety timelock of a LN commitment tx.

At the very least CSFS, with CSFS you can have the `<message>` being the commitment transaction, for which you know the public key (e.g if you're a LN counterparty), but you don't know the `<signature>`.

If you combine it with an UTXO set oracle (e.g `<message=UTXO_123` signed), you've already a rudimentary yet powerful tx-withhold contract. I don't believe CTV allows in any fashion to do more powerful malicious tx-withhold contract, though I believe if it can affect LN, it can certainly affect Ark too.

If I'm correct here, the lemma is that you certainly needs to have any LN chan UTXO be veiled with some kind of consensus-level semantics, "this outpoint cannot be referenced by the Script execution of another UTXO spend". That can be very touchy to implement in bitcoin Script interpreter…

[quote="jamesob, post:32, topic:1509"]
What the watchmen need is super simple. “These keys allow moving the coins to a special timelocked staging area, from which the original keys can still pull them back” It’s basically a one-step vault.

In fact we literally implemented it in terms of unsigned transactions in an early version of liquid, but it was too difficult to keep them synced up and invalidated.
[/quote]

Thanks, this is interesting to get Andrew's opinion here. While I disagree with him on the simplicity (lol) of translating "these keys allow moving the coins..." in a protocol and well-designed cryptographic API, I believe this is backing the point I was raising in my previous comment that you should get support for any opcode at the HW-level or within the secure enclave.

Otherwise, how can you be sure that the "cold keys" are authorizing the spend to the correct hash of a transaction, if you do not re-verify the hash computation on the enclave ? This is the same problem that the Validating Lightning Signer has already today to verify *all* the state transition of the LN protocol to be secure in face of a "compromised" main LN node.

In my opinion, this as much the community of bitcoin protocol experts to convince than HW vendors of all kinds, that a said given opcode should be supported.

—————————————————————————————————————————-

Speaking for myself, I can be supportive of the most minimal opcode improvement that improves self-custody for the lambda bitcoin user, at condition it doesn't introduce tx-withholding risk or extend DoS surface area for full-nodes. If you go in the street to ask to 10 bitcoiners "do you wish to make the self-custody of your coins, *stronger* and *easier*", I genuinely believe the number of positive answer is going to be equal to 10. I don't expect the same level of positive answer for Ark, DLC or payment pool, I think it's just either early or far too complex to be understood by a lambda bitcoin user. In comparison coins self-custody has always been an
area of focus of bitcoin development since people have started to develop lightweight wallets for their own usage in  ~2010 / 2011.

Liquid, no opinion. I've already met many Liquid devs in real-life but I've never met a L-BTC user, which it doesn't mean Liquid users don't exist.

-------------------------

ajtowns | 2025-03-14 03:19:50 UTC | #37

[quote="stevenroose, post:31, topic:1509"]
I’m confused. Are you talking about a technical amendment to CTV? Or are you (again, like in your recent mailing list post) whining about the Motivation section of the BIP? Excuse my French, but if your “biggest blocker” in what is trying to be a pragmatic evaluation of a technical change is some wording in a BIP’s motivation section, I can’t help but question whether you are actually trying to constructively participate in this conversation.
[/quote]

The confusion Jeremy has created with this has been continually used to block discussion of other comparable approaches in this design space, including combinations of CTV with other opcodes, variations on introspection approaches, and other opcodes entirely. Thanks at least for confirming that this is as much of a waste of time as every other CTV discussion has been.

-------------------------

stevenroose | 2025-03-14 14:55:01 UTC | #38

Jeremy has not at all participated in this discussion and you are the first one to bring up both him and the BIP text. Or even CTV history in general. Most other comments to focus on the actual technical merit of the proposal, whether it be evidence or lack of evidence thereof. 

[quote="ajtowns, post:37, topic:1509"]
has been continually used to block discussion of other comparable approaches in this design space
[/quote]

You are the only one bringing up historical artifacts and calling them "blockers".

I am aware I said I wouldn't be pointing fingers, but I tried to start a discussion here that focuses on where we are right now and how we want to move forward and I don't appreciate attempts to derail the conversation into old fights that have been fought before and add absolutely no value to the topic.

Let's yield the space so others have a chance to weigh in...

-------------------------

instagibbs | 2025-03-14 17:01:34 UTC | #39

[quote="instagibbs, post:18, topic:1509"]
I’ve found this use-case the least compelling part, especially at the cost of having to reason about legacy scripting. If it’s really important one idea I’d suggest it be included it as a new segwit output type as a separate feature that lives and dies in a proposal on its own merits. I ended up doing this with P2A for the exact same two reasons.
[/quote]

My technical suggestion for a post-segwit/taproot world:
1. switch to OP_SUCCESSx CTV that pushes to stack (tascript++ only)
2. P2CTV softfork which is defined as a spk of exactly:  
	a) <32 bytes data> <OP_NOP5> # for upgrade hook ensuring no one intended to use this otherwise, no corresponding address. Or it could be any other non-segwit template.  
	b) OR <2> <32 bytes data> # unknown witness version, has address
	
Either one would be required to have empty witness data / scriptSig on spend and does otherwise identical hashing checks on spending tx.

Either bare CTV is important or it's not; it would be pretty trivial to do (2.b)
and allows the reviewer to never think about legacy script opcodes at all.

-------------------------

ariard | 2025-03-14 18:19:37 UTC | #40

[quote="stevenroose, post:38, topic:1509"]
I am aware I said I wouldn’t be pointing fingers, but I tried to start a discussion here that focuses on where we are right now and how we want to move forward and I don’t appreciate attempts to derail the conversation into old fights that have been fought before and add absolutely no value to the topic.
[/quote]

So in my view, of someone who has been involved or have followed the conversation on "covenants” / "contracting primitives" on the last 3 / 4 years, we're more or less doing circles as there is not real convergence on the use-case worthiness, that are improved by covenants.

Following Jeremy's 1st initiative to activate CTV in early 2022, there have been people for who have take a step back and try to improve the development process of consensus changes, be it AJ Towns with bitcoin-inquisition, myself with the Bitcoin Contracting Primitives WG on IRC, Optech with a collection of all covenant ideas, some people who have attempted to reboot in-person Scaling Bitcoin with op.next, etc.

From the last 3 / 4 years, I don't think there have been major technical breakthrough on the conceptual side (i.e finding new cryptographic primitives that allows to do *more* with *less*), and all the conceptual advances have been either lines on opcode-level efficiency tricks or ideas like BitVM or ColliderScript. Those latest novels ideas are interesting in themselves, though of course, there are not achieving the same in terms of performance or expressivity than their logical equivalent, at the consensus-level.

So I think, we're still halting on the lack of social consensus on specific use-cases, that are strong technical and economical motivation enough to warrant consensus changes of Bitcoin Script.

If there is an inability as a community to come to consensus, there is always the path to fork bitcoin core and do an activation client. While I strongly believe more competing clients in the bitcoin space is a good thing on the long-term (yes - you can outcompete bitcoin core on so many technical dimensions), I don't believe this is the way to go when we're talking about consensus changes. There is a value in itself in the stability of the bitcoin network, which is not worth to play on a coin toss, or bargain with brinksmanship.

My position, as expressed in the last paragraph of my previous post, is that I can be supportive of a minimal Script change improving bitcoin coins self-custody, at the condition it doesn’t introduce tx-withholding risks or extend the DoS surface of full-nodes. If it has a side-effect to improve other use-cases, cool, but not something which is ranking high in my list of bitcoin technical items I'm interested to contribute on.

In a spirit to move the conversation forward, we had 11 developers expressing on this thread so far, eventually each one could express which use-cases they can be interested by, and if there is any primitive they think will improve the use-case, or the technical reasons they think a primitive have weakness, limitation, is not suiting their particular use-case, etc.

Expressing technical motivations why you're against an idea (e.g me against CSFS due to tx-withholding risk), is I think a good development practice as objective elements can be discussed on, rather than information-poor marks like "thumbs up" or "like" on gh or twitter.

If the conservation stalls, I'm fine with the status quo.

-------------------------

jamesob | 2025-03-14 18:22:17 UTC | #41

[quote="instagibbs, post:39, topic:1509"]
Either bare CTV is important or it’s not; it would be pretty trivial to do (2.b) and allows the reviewer to never think about legacy script opcodes at all.
[/quote]

There is very little marginal complexity in the implementation that is due to bare/legacy CTV. On the other hand, there is significant complexity in an additional fork deployment - both in terms of added code and process.

What is the concrete reason that CTV as `OP_NOP4` is a sticking point for you?

-------------------------

jamesob | 2025-03-14 18:28:01 UTC | #42

[quote="ariard, post:40, topic:1509"]
we’re more or less doing circles as there is not real convergence on the use-case worthiness, that are improved by covenants.
[/quote]

Except that in the last week, we've had
- 2 Blockstream engineers go on record as saying CTV would be valuable for Liquid,
- the CEO of one of Bitcoin's most popular hardware wallets saying [he'd implement vaults](https://x.com/nvk/status/1899918763728003420),
- the CEO of Bitcoin's only custodial insurance product saying [he wants vaults](https://x.com/Rob1Ham/status/1897781338796966103), and
- a prototype of Ark based on CTV passing tests and showing a substantial reduction in the interactivity required (mentioned above).

I wouldn't call that "doing circles!"

-------------------------

instagibbs | 2025-03-14 19:00:11 UTC | #43

[quote="jamesob, post:41, topic:1509"]
There is very little marginal complexity in the implementation that is due to bare/legacy CTV.
[/quote]

IIUC, the NOP/verify pattern is primarily being justified due to bare usage. I am suggesting an alternative (at handwave height) which fits the same hole, while converting the rest of the functionality to something that is more efficient for the use cases I care about, signing hashes that reside on the stack.

I admit this is speaking off the cuff, but I'd like to know why this is not a good idea.

[quote="jamesob, post:41, topic:1509"]
On the other hand, there is significant complexity in an additional fork deployment - both in terms of added code and process.
[/quote]

If consensus arrives that the bare case is worthwhile, just bundle them.

[quote="jamesob, post:41, topic:1509"]
What is the concrete reason that CTV as `OP_NOP4` is a sticking point for you?
[/quote]

If reviewers (me) don't have to think about legacy script, it's a win.

-------------------------

AntoineP | 2025-03-14 21:11:30 UTC | #44

[quote="jamesob, post:42, topic:1509"]
the CEO of one of Bitcoin’s most popular hardware wallets saying [he’d implement vaults](https://x.com/nvk/status/1899918763728003420),
[/quote]

The post you are linking to mentions `OP_VAULT`. It seems inappropriate to use as motivation for a proposal, a statement in favour of a separate proposal.

[quote="jamesob, post:42, topic:1509"]
the CEO of Bitcoin’s only custodial insurance product saying [he wants vaults](https://x.com/Rob1Ham/status/1897781338796966103), and
[/quote]

This post does not mention `OP_CTV` either. Also, AnchorWatch is great but it's a company that launched less than 3 months ago. 

Meanwhile Bob McElrath [stated](https://x.com/BobMcElrath/status/1900445574626693258) that from his experience researching vaults for years at [Fidelity Digital Asset](https://www.fidelitydigitalassets.com), one of the largest Bitcoin custodians in the world, vaults sounded like a good idea but were not in practice. The CEO of [Nunchuk](https://nunchuk.io/), a popular Bitcoin wallet in operation since 2021, [stated](https://x.com/hugomofn/status/1899998739512832377) that after investigating vaults he did not believe those were practical for end users ("plebs"). The CEO of [Wizardsardine](https://wizardsardine.com/), who spent years developing vaults and otherwise developing advanced custody products also [stated](https://x.com/KLoaec/status/1897694745050194392) that in practice the revealed preferences of people is that they'd rather not take on the burden that comes with managing a vault. The reasons stated by every single of these persons (as well as myself) for their skepticism of the value of vaults in practice is all the infrastructure that comes with it (watchtowers, etc). These opinions were informed by their experience in actually building all that comes with a product targeting real-life end users, and this would also apply to an ideal vault architecture with an opcode dedicated to it[^0].

Now, all these discussions you brought up are about vaults that the proposal at hand does not enable. So i hope we can "put this matter to rest" and come back on topic to usecases actually enabled by the combination of `OP_CTV` and `OP_CSFS`. So far i see three:
- Make LN-symmetry a possible option for Lightning developers
- Make it [easier for Lightning to upgrade to PTLCs](https://delvingbitcoin.org/t/ctv-csfs-can-we-reach-consensus-on-a-first-step-towards-covenants/1509/7?u=antoinep) (regardless of LN-symmetry usage)
- Improve Ark (@stevenroose could you lay down the exact improvements?)
- Reduce interactivity in various (potentially future or not yet known) protocols and applications
- In theory be helpful for UTxO management for Liquid watchmen, which would be fair to generalize to custodial sidechains in general

[^0]: Something like `OP_VAULT` seems to have other interesting usecases beyond vault, but again, this is off-topic for this thread.

-------------------------

jamesob | 2025-03-14 21:37:02 UTC | #45

[quote="AntoineP, post:44, topic:1509"]
The post you are linking to mentions `OP_VAULT`. It seems inappropriate to use as motivation for a proposal, a statement in favour of a separate proposal.
[/quote]

A prerequisite for any vault is unambiguously locking the coins to a certain destination during the trigger stage. CTV is the most basic way of doing this; in fact, anything that is capable of accomplishing this is either CTV or a generalization. So an argument for the value of vaults is in essence an argument for CTV (or something more general that encompasses it as one mode).

[quote="AntoineP, post:44, topic:1509"]
Meanwhile Bob McElrath [stated](https://x.com/BobMcElrath/status/1900445574626693258) ...
[/quote]

One (or three) custodians saying "I'm not going to use taproot, it doesn't make sense for me" isn't an argument against taproot. Especially if you have a number of other custodians (who are more current and not working on competing products) saying, "yes, I'd really like this, it would make a difference for my business."

-------------------------

AntoineP | 2025-03-14 21:44:33 UTC | #46

[quote="jamesob, post:45, topic:1509"]
A prerequisite for any vault is unambiguously locking the coins to a certain destination during the trigger stage. [...] So an argument for the value of vaults is in essence an argument for CTV
[/quote]

I disagree. If people are proposing to change Bitcoin in X manner, they should make the case for X. Not for a future theoretical Y change that X combines well with. Y should only matter to argue that X won't be rendered obsolete in the near future (which Steven did pretty convincingly in OP), to convince people it is worth going through a soft fork for X without waiting for Y.

-------------------------

1440000bytes | 2025-03-14 22:03:45 UTC | #47

[quote="AntoineP, post:44, topic:1509"]
Meanwhile Bob McElrath [stated](https://x.com/BobMcElrath/status/1900445574626693258) that from his experience researching vaults for years at [Fidelity Digital Asset](https://www.fidelitydigitalassets.com), one of the largest Bitcoin custodians in the world, vaults sounded like a good idea but were not in practice. The CEO of [Nunchuk](https://nunchuk.io/), a popular Bitcoin wallet in operation since 2021, [stated](https://x.com/hugomofn/status/1899998739512832377) that after investigating vaults he did not believe those were practical for end users (“plebs”). The CEO of [Wizardsardine](https://wizardsardine.com/), who spent years developing vaults and otherwise developing advanced custody products also [stated](https://x.com/KLoaec/status/1897694745050194392) that in practice the revealed preferences of people is that they’d rather not take on the burden that comes with managing a vault.
[/quote]

There are several other developers and users who need vaults with covenants. This includes companies with successful products and services.

What do others achieve by blocking someone from using different types of vaults?

-------------------------

jamesob | 2025-03-14 22:18:13 UTC | #48

[quote="instagibbs, post:43, topic:1509"]
IIUC, the NOP/verify pattern is primarily being justified due to bare usage.
[/quote]

VERIFY is also important for upgradeability; if you don't have the <32-byte-hash> parameter, you can't make the opcode upgradeable.

Now maybe in a tapscript context that's okay because you've got so much unused opcode space.

[quote="instagibbs, post:43, topic:1509"]
I’d like to know why this is not a good idea.
[/quote]
I'm not averse to it, I'm just trying to figure out why you want it, since all the uses of CTV I'm familiar with are VERIFYs - I gather it's to make ln-symm more efficient or something? I'm not trying to be glib here, I just think the motivation for what you suggested wasn't explicitly mentioned.

I'm totally fine with having adding a tapscript-only `OP_CHECKTEMPLATEHASH` that pushes the hash to the stack if it helps someone, that's not a lot of marginal complexity. But I think the idea that it would replace CTV as-is is sort of a different proposal.

[quote="instagibbs, post:43, topic:1509"]
If reviewers (me) don’t have to think about legacy script, it’s a win.
[/quote]
Again, not trying to be glib here but is there a more specific reason than "each reviewer has to review about 15 marginal lines of not-substantially-novel code?" What about legacy is so complicated that it really moves the needle?

I understand the argument that we only have so much NOP-space left, but given the bare CTV case is "most compact way possible to commit to `n` future spends," I think it probably justifies the allocation.

Please grant me the goodwill to admit it's a reasonable thought that if chainspace ever does become massively short, congestion control is a good escape hatch to have on hand - even outside of the use for miners today.

-------------------------

Rob1Ham | 2025-03-15 17:32:28 UTC | #49

[quote="AntoineP, post:44, topic:1509, full:true"]
[quote="jamesob, post:42, topic:1509"]
the CEO of Bitcoin’s only custodial insurance product saying [he wants vaults](https://x.com/Rob1Ham/status/1897781338796966103), and
[/quote]

This post does not mention `OP_CTV` either. Also, AnchorWatch is great but it's a company that launched less than 3 months ago. 

[/quote]

Making my first post at Delving (only because I'm specifically referenced, I just lurk otherwise) to say the post does not mention `OP_CTV` specifically, but the need to make a commitment to a future transaction like the primitive of what `OP_CTV` enables would be a necessary part of the vaulting structure from my view.

Most of my reading of `OP_VAULT` is from the official [BIP 345](https://github.com/bitcoin/bips/blob/00c13baff0dc4a3a250d9725129b0d2c8d0be6a9/bip-0345.mediawiki) which makes several references to `OP_CTV`.

I'm in support of CTV + CSFS.

-------------------------

stevenroose | 2025-03-17 17:24:27 UTC | #50

**EDIT: To avoid derailing this conversation with questions and suggestions on the below reply, I created a [separate topic here](https://delvingbitcoin.org/t/the-ark-case-for-ctv/1528) where the post is copied and reactions can be posted :pray:**

Because it was requested, let me make the [Ark case for CTV](https://x.com/stevenroose3/status/1865144252751028733) in a bit more detail, because I think it is significant. One can also refer to [ark-protocol.org](https://ark-protocol.org/intro/vtxos/index.html) for more detail.

TL;DR: send-to-others, automatic vtxo reissuance, lightning receive and mass payouts

For those that don't know, in Ark users hold off-chain utxos that we call virtual utxos or vtxos. In the original design, which uses CTV, these vtxos are constructed using a tree of transactions where each transaction uses a CTV covenant to get into the next transaction. Much like congestion control.

Generalized, a vtxo can be any construction that has:

1. a chain or one or more txs with policy `ctv OR (pk(S) AND after(T))` (`S` being the Ark server's key and `T` being the vtxo's expiry time). Each `ctv` comitting to the next tx in the chain

2. followed by a tx with the following policy `(pk(A) AND pk(S)) OR (pk(A) AND older(delta))` (`A` being the key of the vtxo's owner and `delta` being a relative timelock which gives the other policy enough time to react (something like 24h or 28h).

Using CTV, this construction can be constructed entirely non-interactively, meaning that knowing the Ark's parameters `S` and `delta`, anyone can issue their own vtxo that expires at time `T`. This is how "onboarding" works, i.e. entering the Ark.

When we developing `clArk` (covenant-less Ark), we replace the `ctv` policy with a MuSig2 pre-signature of `S` and all the users of the leaves below that node. The intermediate nodes from step 1. above become then `pk(S+A+B+C+..) OR (pk(S) AND after(T))`. Apart from additional signing and storage overhead, this has one strong limitation:

**Co-signed (clArk) vtxos cannot be issued without the presence of the eventual owner**, otherwise the construction is not safe. This has led our team (at [Second](second.tech)) to only designate Ark rounds to be used for users that have expiring vtxos and need them to be refreshed, in effect "sending them to themselves". Because senders always have to be present in either Ark construction (they need to sign [forfeit txs](https://ark-protocol.org/intro/connectors/index.html#forfeit-transactions)), if the receivers are also the senders, they are already present during the process.

But being able to issue vtxos for others has several significant benefits:

- The most obvious is simply being able to **send vtxos to others without requiring the receiver to participate in Ark rounds**. We currently only support users sending to each other using [out-of-round (Arkoor)](https://ark-protocol.org/intro/oor/index.html) transactions, which has an additional security assumption. With CTV we could again support sending vtxos to others in rounds.

- It allows the server to issue vtxos for users. We currently want to do this in two occasions:
  - A good server will want to **re-issue expired vtxos automatically**. This is obviously not secure, but the server can continuously provide proofs that it hasn't claimed any expired vtxos and for some smaller-value vtxos, this can be enough security for certain users.
  - **Receiving Lightning payments** in Ark (without having virtual channels) is not easy. One way it could be done is by having the user notify the server that it is expecting an inbound payment, the server will accept the HTLC as a hodl HTLC and issue an HTLC for the user in a vtxo in the next round. The user can then reveal the preimage which grants him the vtxo fully and allows the server to claim the hodl HTLC. (With various anti-abuse measures in place.) Without CTV the receiver would have to participate in a round in order to receive his HTLC.

- It would allow non-interactive onboards, but since for an onboard, the onboarder is usually the receiver anyway this seems quite uninteresting, but it equivalently **allows any party to non-interactively issue any number of vtxos with a single on-chain output**. We think this can be a very powerful tool for exchanges or DCA providers that want to regularly payout amounts to their users. Using CTV, they could independently construct a tree of vtxos (using the Ark's parameters), fund the root tx and inform all their users. Since these vtxos are no different than vtxos created in Ark rounds, the recipients can use them as if they were any other type of vtxo in this Ark.

At Second we believe these benefits are quite significant for both the user experience more broadly and the viability of participating in Ark rounds a the mobile setting in the first place. We have little data currently to back up this latter claim, but we are working hard to put up our implementation of co-signed Ark to the test on mobile clients.

-------------------------

instagibbs | 2025-03-17 14:43:03 UTC | #51

[quote="jamesob, post:48, topic:1509"]
VERIFY is also important for upgradeability; if you don’t have the <32-byte-hash> parameter, you can’t make the opcode upgradeable.

Now maybe in a tapscript context that’s okay because you’ve got so much unused opcode space.
[/quote]

Good point, but you're correct I've basically stopped thinking about these kinds of details because we have so many upgrade hooks. Let's use them!

[quote="jamesob, post:48, topic:1509"]
I’m just trying to figure out why you want it, since all the uses of CTV I’m familiar with are VERIFYs
[/quote]

The use-cases I'm thinking about that involve the CTV + CSFS combo means we can elide bringing the hash into the witness data, saving ~32WU. State channels, payment channels, statechains, pre-signed HTLC transactions, PTLC transactions. Probably more, but probably most of the CTV + CSFS intersection of functionality. 

The tradeoff here is 1WU(?) in the tapscript CTV case, which would apply to things like Ark, Timeout Trees, settlement path in ln-symmetry(et al.), anywhere where authorization isn't required, just precommitment. I don't find this a compelling savings if we have to choose just one.

And if VERIFY behavior is still desired, you could still softfork a NOP, just in tapscript circumstances.

[quote="jamesob, post:48, topic:1509"]
But I think the idea that it would replace CTV as-is is sort of a different proposal.
[/quote]

I'm much more interested in CTV + CSFS than CTV alone, so there lies some design bias.

[quote="jamesob, post:48, topic:1509"]
Again, not trying to be glib here but is there a more specific reason than “each reviewer has to review about 15 marginal lines of not-substantially-novel code?” What about legacy is so complicated that it really moves the needle?
[/quote]

I'm thinking about it from another direction: Designing BIP119 from scratch, how should we build it? Legacy script (short-hand for non-segwit, not NOPs) is a dumpster fire, and it shouldn't be touched unless necessary. 

If it saves no vbytes, makes no operations safer, doesn't make the changes easier to understand for reviewers, why would we build it this way? 

[quote="jamesob, post:48, topic:1509"]
Please grant me the goodwill to admit it’s a reasonable thought that if chainspace ever does become massively short, congestion control is a good escape hatch to have on hand - even outside of the use for miners today.
[/quote]

I think bare CTV in any form is one of the least motivated proposals I've seen in a long while, but if it's going to happen, I want it to happen in the way that's least damaging in terms of maintenance of Bitcoin script. I believe my proposal is one way to do it without changing any known capabilities from the original.

-------------------------

ariard | 2025-03-17 22:28:38 UTC | #52

[quote="jamesob, post:42, topic:1509"]
Except that in the last week, we’ve had

* 2 Blockstream engineers go on record as saying CTV would be valuable for Liquid,
* the CEO of one of Bitcoin’s most popular hardware wallets saying [he’d implement vaults](https://x.com/nvk/status/1899918763728003420),
* the CEO of Bitcoin’s only custodial insurance product saying [he wants vaults](https://x.com/Rob1Ham/status/1897781338796966103), and
* a prototype of Ark based on CTV passing tests and showing a substantial reduction in the interactivity required (mentioned above).

I wouldn’t call that “doing circles!”
[/quote]

Okay so yes to be precise, it was more on the theoretical / primitive design space technical options, in the sense that I don't think we have seen primitives breakthrough since OP_TLUV, OP_EVICT, MATT or PT2R merkle tree editors since few years ago. The only recent breakthrough have been constructions like BitVM or ColliderScript, to attempt to achieve the same for some use-cases, without consensus changes.

Though yes, getting industry's opinions on how they would see vaults in practice, and not only on whiteboard is a notable move forward, I do not disagree. IMHO, the more interesting question, I was laying is if HW / secure enclave vendors would be ready to have CTV-like support on the enclave-side, otherwise it's "blind signing" for the key ceremonies (cf. "[Blind Signing Considered Harmful](https://medium.com/@devrandom/blind-signing-considered-harmful-ac82e5852853)").

[quote="AntoineP, post:44, topic:1509"]
Meanwhile Bob McElrath [stated](https://x.com/BobMcElrath/status/1900445574626693258) that from his experience researching vaults for years at [Fidelity Digital Asset](https://www.fidelitydigitalassets.com), one of the largest Bitcoin custodians in the world, vaults sounded like a good idea but were not in practice.
[/quote]

In my perspective, the only security property improvement, that one can argue on is *immutability*, that once the coins are deposited on-chain beyond reorgs, there is no more reliance on correct and secure key-deletion for the intermediary unvaulting transactions.

From browsing the Nunchuk, McElrath and Wizardsardine viewpoints, if I'm understanding correctly they're saying either (a) vault it's hard because *reactive model* (so chain monitoring or watchtower) or (b) vault it's hard because duplicate set of keys. To have researched in the past extensively second-layers threats models (cf. [L2 Transaction-Relay Workshops](https://github.com/ariard/L2-zoology)), vault security model is not more complex than Lightning, and while LN is plagued by security vulnerabilities and limitations, courageous wallet vendors have succeeded to integrate it on mobile and desktop wallets. Of course, key ceremonies is a hard thing for vaults, i.e dealing well the transition from cold storage, though it's that harder than managing seeds transfers on multiple platforms for LN, and inadvertently broadcasting an already revoked state ? I don't think so...

Contrary to LN, vaults are not stateful, in the sense of off-chain update with a counterparty. This is already far less complexity, imo.

[quote="AntoineP, post:44, topic:1509"]
Now, all these discussions you brought up are about vaults that the proposal at hand does not enable. So i hope we can “put this matter to rest” and come back on topic to usecases actually enabled by the combination of `OP_CTV` and `OP_CSFS`. So far i see three:
[/quote]

Fwiw, if we wish to have LN-Symmetry / Eltoo, we should just do ANYPREVOUT. I think OP_CSFS enhances the range of [TxWithhold](https://blog.bitmex.com/txwithhold-smart-contracts/) contract, as you can pass now a signature for a tx, you're not a counterparty to (i.e not a pubkey in a 2-of-2 P2WSH), which can be concerned for already deployed L2s like Lightning.

[quote="instagibbs, post:51, topic:1509"]
I’m thinking about it from another direction: Designing BIP119 from scratch, how should we build it? Legacy script (short-hand for non-segwit, not NOPs) is a dumpster fire, and it shouldn’t be touched unless necessary.

If it saves no vbytes, makes no operations safer, doesn’t make the changes easier to understand for reviewers, why would we build it this way?
[/quote]

I do not disagree that we should avoid to go for legacy script, avoids making the DoS surface worst (e.g worst-block case). I'm re-reading BIP119 though there is no footnote to explain why `OP_NOP4` vs another upgrade path (e.g tapleaf version or `OP_SUCCESS`).

Doing an OP_PUSH `<template>` instead of a OP_CHECK `<template>`, that means now you have a comparison on the Script stack to be done between `<pushed_template>` and `<reference_template>`. I do no think something in terms of `<template>` *bad malleability* it's an issue (hmm...), however it's more memory allocated among the different script ops on the stack by each full-nodes. Not sure it's a concern.

-------------------------

instagibbs | 2025-03-17 22:45:20 UTC | #53

[quote="ariard, post:52, topic:1509"]
I’m re-reading BIP119 though there is no footnote to explain why `OP_NOP4` vs another upgrade path (e.g tapleaf version or `OP_SUCCESS`).
[/quote]

I believe for the very understandable reason that it was prior to the taproot BIP being written.

-------------------------

moonsettler | 2025-03-18 00:52:54 UTC | #54

Also bare CTV seems pretty useful and economic for certain cases. Has no "address" to send to by accident from wallets which could result in burned funds.

-------------------------

harding | 2025-03-18 15:59:31 UTC | #55

[quote="instagibbs, post:53, topic:1509"]
I believe for the very understandable reason that it was prior to the taproot BIP being written.
[/quote]

I believe the original idea was indeed developed before publication of the taproot and tapscript BIPs, but drafts of those BIPs were [published](https://gnusha.org/pi/bitcoindev/CAPg+sBg6Gg8b7hPogC==fehY3ZTHHpQReqym2fb4XXWFpMM-pQ@mail.gmail.com/) about two weeks before [OP_COSHV](https://mailing-list.bitcoindevs.xyz/bitcoindev/CAD5xwhgHyR5qdd09ikvA_vgepj4o+Aqb0JA_T6FuqX56ZNe1RQ@mail.gmail.com/), the predecessor for CTV.  Versions of both the
[COSHV](https://github.com/JeremyRubin/bips-archive/blob/87ae732da28346515be9049ea2d2b548f81812e8/bip-coshv.mediawiki#deployment) and [STB](https://github.com/JeremyRubin/bips-archive/blob/e2b34bd59986bd2b173a89ac969b4728170fa8fa/bip-secure-the-bag.mediawiki#deployment) BIPs linked to the draft Tapscript BIP and depended on it making `OP_RESERVED1` (0x89) opcode into an `OP_SUCCESSx` in tapscript.

-------------------------

instagibbs | 2025-03-19 16:39:12 UTC | #56

I suppose this alternative is more likely: If you're not considering composibility with other future opcodes, there's not much to be done with a "push to stack" variant except equality checks.

-------------------------

ariard | 2025-03-21 00:13:23 UTC | #57

[quote="instagibbs, post:53, topic:1509"]
I believe for the very understandable reason that it was prior to the taproot BIP being written.
[/quote]
[quote="harding, post:55, topic:1509"]
I believe the original idea was indeed developed before publication of the taproot and tapscript BIPs, but drafts of those BIPs were [published](https://gnusha.org/pi/bitcoindev/CAPg+sBg6Gg8b7hPogC==fehY3ZTHHpQReqym2fb4XXWFpMM-pQ@mail.gmail.com/) about two weeks before [OP_COSHV](https://mailing-list.bitcoindevs.xyz/bitcoindev/CAD5xwhgHyR5qdd09ikvA_vgepj4o+Aqb0JA_T6FuqX56ZNe1RQ@mail.gmail.com/), the predecessor for CTV. Versions of both the [COSHV](https://github.com/JeremyRubin/bips-archive/blob/87ae732da28346515be9049ea2d2b548f81812e8/bip-coshv.mediawiki#deployment) and [STB](https://github.com/JeremyRubin/bips-archive/blob/e2b34bd59986bd2b173a89ac969b4728170fa8fa/bip-secure-the-bag.mediawiki#deployment) BIPs linked to the draft Tapscript BIP and depended on it making `OP_RESERVED1` (0x89) opcode into an `OP_SUCCESSx` in tapscript.
[/quote]

I always thought the original author of BIP119  (@JeremyRubin) had or did a version on top of Taproot. But didn’t read the BIP recently.

Anyway, try a quick-didn’t-check-it-compiles-because-im-too-lazy version of OP_CHECKTEMPLATEVERIFY on top of bitcoin-inquisition 28.x (cf. [the full branch](https://github.com/ariard/bitcoin/commit/f8820e583a3d0d819955e2177ead95789a6317f1)).

```
case OP_CTV:
{
    if (flags & SCRIPT_VERIFY_DISCOURAGE_CHECK_TEMPLATE_VERIFY_HASH) {
        return set_error(serror, SCRIPT_ERR_DISCOURAGE_UPGRADABLE_CTV);
    }

    // if flags not enabled; treat as a NOP4
    if (!(flags & SCRIPT_VERIFY_OP_CTV)) {
        break;
    }

    if (stack.size() < 1) {
        return set_error(serror, SCRIPT_ERR_INVALID_STACK_OPERATION);
    }

    // If the argument was not 32 bytes, treat as OP_NOP4:
    switch (stack.back().size()) {
        case 32:
        {
            const Span<const unsigned char> hash{stack.back()};
            if (!checker.CheckDefaultCheckTemplateVerifyHash(hash)) {
                return set_error(serror, SCRIPT_ERR_TEMPLATE_MISMATCH);
            }
            break;
        }
        default:
            // future upgrade can add semantics for this opcode with different length args
            // so discourage use when applicable
            if (flags & SCRIPT_VERIFY_DISCOURAGE_UPGRADABLE_CHECK_TEMPLATE_VERIFY_HASH) {
                return set_error(serror, SCRIPT_ERR_DISCOURAGE_UPGRADABLE_TEMPLATE);
            }
    }
}
break;
```

I think other upgradable paths could be tested for CTV, e.g upgradable tapscript version, to something that is not *0xC0* as a leaf head byte.

[quote="moonsettler, post:54, topic:1509, full:true"]
Also bare CTV seems pretty useful and economic for certain cases. Has no “address” to send to by accident from wallets which could result in burned funds.
[/quote]

This point deserves better discussions, as I think it’s the opposite. Due to the *immutability* of any chain of transactions generated with CTV (i.e with a sig-based escape path), the whole chain of transaction should be verified by any recipient (e.g verify that intermediary tx in a congestion control tree is not nLocktime=<year_2109> i.e practically *freezing* your funds). It’s more likely that good practice of using CTV should come with *enhanced* “address” carrying enough info to any recipient to verify the semantic of the chain of transaction, which is in itself a use-case specific thing.

[quote="instagibbs, post:56, topic:1509"]
there’s not much to be done with a “push to stack” variant except equality checks.
[/quote]

So if you can do equality checks on a template and one can pre-compute the result of equality checks on an outpoint (txid:output_index) and pre-commit them in a *redeemScript*, it’s already an interesting primitive to do adversarial [tx-withholding](https://blog.bitmex.com/txwithhold-smart-contracts/). However, I don’t think this is still a *non-collaborative* utxo oracle, as the templated tx would have been to be built with a valid witness for the probed utxo. So somehow there is an opt-in of the owner of the probed utxo for the spend to be integated in the CTV template. But I’m not sure.

-------------------------

instagibbs | 2025-04-11 14:08:36 UTC | #58

FYI: after discussion a while back, I hacked together very small loc changes that:

1) Makes bare CTV a first class citizen
2) Converts CTV into a push-onto-stack taproot-only opcode

It's a very small set of changes, does not change the hash digest formula for BIP119 (nor invalidate the unit tests), cheaper to use in conjunction with CSFS for any floating signature scheme.

https://github.com/instagibbs/bitcoin/commit/9b5ae77d97222efc2328c1916f790c7f67cf10b3

edit: I also tried another alternative where bare CTV follows exactly he format today, but only softforks for that OP_NOP template, to reduce the scope of the softforking behavior where we only have to test the single bare CTV case vs additional bare scripts. It was messy and not worth it.

-------------------------

ariard | 2025-04-12 02:51:25 UTC | #59

[quote="instagibbs, post:58, topic:1509"]
Converts CTV into a push-onto-stack taproot-only opcode
[/quote]


I’m +1 for taproot-only CTV as an op_success for future composability. No opinion on an OP_PUSHTEMPLATE + OP_EQUALVERIFY or OP_CHECKTEMPLATEVERIFY approach.

By the way, I believe there might be some DoS surface that has been not considered in the BIP:
https://github.com/bitcoin/bitcoin/pull/31989#discussion_r2040517393

-------------------------

securitybrahh | 2025-05-01 04:58:38 UTC | #60

(sorry for being a noob and lazy but) can someone give me tldr on why we stacking ctv and csfs, and not cat and csfs.

CAT gives more room for spam? also ofc ctv is most historically not merged tested opcode and appearantly CAT was switched off by Satoshi himself (or so I have heard on the streets) 

hope CTV doesn't activate some non-desirables.

P.S. I hate ossification. 

![1000015680|277x500](upload://7xgJsmZdk71OdYLuQmmqW3cUuY0.jpeg)

https://en.wikipedia.org/wiki/G%C3%B6del's_incompleteness_theorems

-------------------------

moonsettler | 2025-05-05 22:42:36 UTC | #61

[quote="securitybrahh, post:60, topic:1509"]
CAT gives more room for spam?
[/quote]
No. That sounds impossible. Can it enable new token protocols and applications interacting with them on bitcoin? Yes.
[quote="securitybrahh, post:60, topic:1509"]
hope CTV doesn’t activate some non-desirables.
[/quote]
Nobody was able to come up with anything remotely plausible so far. And it's been many years.

-------------------------

securitybrahh | 2025-05-06 06:38:14 UTC | #62

[quote="securitybrahh, post:60, topic:1509"]
switched off
[/quote]

oh CAT was removed in [2010](https://github.com/bitcoin/bitcoin/commit/4bd188c4383d6e614e18f79dc337fbabe8464c82#diff-8458adcedc17d046942185cb709ff5c3R94) lol. 

because of this apparently:


https://en.bitcoin.it/wiki/Value_overflow_incident

-------------------------

securitybrahh | 2025-05-06 06:42:36 UTC | #63

https://gnusha.org/pi/bitcoindev/20220420023107.GA6061@erisian.com.au/

prob relevent to the thread, I should read the thread quickly first though.

-------------------------

securitybrahh | 2025-05-08 07:23:41 UTC | #64

Also the core-devs has put a "regtest" label on the PR, can I see thr opcodes running on something like mempool.space?

-------------------------

moonsettler | 2025-05-27 14:44:18 UTC | #65

regtest means you can only use it on your own computer on a private blockchain.

-------------------------

securitybrahh | 2025-05-28 10:08:00 UTC | #66

lol, atleast let me run on some test net that we all can view through mempool space UI.

-------------------------

Chris_Stewart_5 | 2025-06-05 00:09:35 UTC | #67

Has anyone actually published a branch that has both CTV+CSFS? From skimming through this topic I don't see one. It would be nice to have one on top of the major latest release of bitcoin core for integration testing the 2 opcodes together.

-------------------------

AntoineP | 2025-06-05 13:29:31 UTC | #68

Good point. Greg [has a PR](https://github.com/bitcoin-inquisition/bitcoin/pull/72) to introduce CSFS to inquisition. This is in effect a branch with both CTV+CSFS. It's worth noting nobody bothered reviewing it in the past 3 months.

-------------------------

Chris_Stewart_5 | 2025-06-05 21:47:46 UTC | #69

I think the @RobinLinus and @ajtowns interactions on the BitVM & CTV+CSFS thread illustrates the subtle foot guns that come with OP_CTV as an OP_NOP and OP_SUCCESS. I don't think I would have caught this subtle vulnerability that comes with supporting legacy Script. 

This interaction occurred after the posts on this thread arguing about OP_NOP support, so I feel like its worthwhile to highlight it here.

[quote="ajtowns, post:8, topic:1591"]
where `<H>` locks in where the output goes to, and input 2’s scriptSig. But because H is only committing to the second input’s scriptSig, then it’s easy to construct another utxo that can be used instead, eg one with the (non-standard) scriptPubKey `OP_2DROP OP_TRUE`:

```
utxo C:  500 sats, scriptPubKey: OP_2DROP OP_TRUE

input 1: utxo A, no witness/scriptSig
input 2: utxo C, scriptSig = "<S;NONE|ANYONECANPAY>" "<P> CHECKSIG"
output: whatever
```

That allows utxo A to be spent via the CTV path independently of whether utxo B has already been spent/burnt, which, as far as I can see, breaks the protocol you’re trying to enforce.

(I’m assuming in a real example utxo B’s spend condition is more complicated than `<P> CHECKSIG`, as otherwise the availability of a NONE|ANYONECANPAY signature means it can be spent immediately, so a griefer could fairly easily prevent the happy path from ever being taken)
[/quote]

It seems like this could be worked around since OP_CTV does commit to the scriptSig according to instagibbs

[quote="instagibbs, post:9, topic:1591, full:true"]
The scriptSig could include the `CHECKSIG` opcode directly, contra standardness rules. :grimacing:

In the original idea, a p2sh redeemscript is just pushes, so the spk could just be blank and it would pass, since cleanstack isn’t consensus anyways.
[/quote]

However this seems rather unpleasant. 

I think Jeremy Rubin is conceding the point that at least for p2sh OP_CTV is broken? 

[quote="JeremyRubin, post:11, topic:1591"]
This is an interesting point; without some sort of “stack sentinel” that guarantees a specific script type, using CTV as a gadget for any P2SH type seems broken, as you can replace it with a legacy script that does something else. This is “confusing” because you cannot replace it with a p2sh script that does something else. I can make an effort to better document this issue…
[/quote]

Which leaves us with only spending "raw"/"bare" outputs to be able to commit to meaningful data in the scriptSig with CTV.

Is the juice really worth the squeeze for committing to scriptSig data?

-------------------------

instagibbs | 2025-06-06 11:47:46 UTC | #70

I've come to see this capability of CTV (committing to other prevouts in very round-about way) as an anti-feature, an unexpected capability that is so sharp-edged that if it's something we want we should explicitly design for it instead to discourage such bizarre usage.

Alternatively we could remove the capability by making non-empty scriptSigs fail the script execution. This of course only makes sense in a post-segwit world only which I've already advocated above as a taproot-only push opcode.

-------------------------

josh | 2025-06-06 17:37:06 UTC | #71

> This of course only makes sense in a post-segwit world only which I’ve already advocated above as a taproot-only push opcode.

@instagibbs If a taproot-only push opcode is the way to go, would it make sense to introduce a new witness version for bare tapscript?

That way, you can still make bare CTV commitments (leaving other types of bare scripts non-standard).

-------------------------

instagibbs | 2025-06-06 20:41:37 UTC | #72

[quote="josh, post:71, topic:1509"]
If a taproot-only push opcode is the way to go, would it make sense to introduce a new witness version for bare tapscript?
[/quote]

I did here https://delvingbitcoin.org/t/ctv-csfs-can-we-reach-consensus-on-a-first-step-towards-covenants/1509/58?u=instagibbs 

I find it nice to consider both separately, even if they both end up being adopted.

-------------------------

jamesob | 2025-06-07 16:06:31 UTC | #73

[quote="instagibbs, post:58, topic:1509"]
FYI: after discussion a while back, I hacked together very small loc changes that:

1. Makes bare CTV a first class citizen
2. Converts CTV into a push-onto-stack taproot-only opcode
[/quote]

This is a cool patch, thanks for doing it.

I still prefer the existing CTV impl. for a few reasons:

1. Bare legacy CTV is upgradeable (i.e. >32byte CTV hashes). Though you could make your patch upgradeable by returning true for any witV2 with a program size of over 32 bytes.
2. In wit v0 CTV, you can have scripts that are more complicated than just a single CTV invocation, but avoid the Taproot control block overhead of 33vB (e.g. in [simple-ctv-vault](https://github.com/jamesob/simple-ctv-vault/blob/7dd6c4ca25debb2140cdefb79b302c65d1b24937/main.py#L303-L316)). 33vB may well be worth fretting over in the future.
3. This is probably fringe, but one "nice" thing about bare CTV legacy scripts is that because they don't have an associated address, accidental address reuse by a human is impossible (i.e. erroneous send to an already-setup vault). ~~Of course your witV2 scheme doesn't define an address format, but it would be easy to.~~ Edit: witV2 by default [uses bech32m addresses](https://github.com/bitcoin/bitcoin/blob/e2174378aa8a339c7be8b4e91311513ed520a16d/src/key_io.cpp#L68-L78).

-------------------------

jamesob | 2025-06-09 17:14:44 UTC | #74

@instagibbs Maybe it goes without saying, but I think it'd be totally reasonable to layer on a tapscript-only `OP_CHECKTEMPLATE` to the existing proposal. If it saves ~~32 vbytes~~ 8WU for the Lightning case and likely others, I think that's great.

-------------------------

instagibbs | 2025-06-09 15:51:41 UTC | #75

[quote="jamesob, post:73, topic:1509"]
Bare legacy CTV is upgradeable
[/quote]

No other output lengths would be defined for the "P2CTV" case, there's no difference in this respect.

[quote="jamesob, post:73, topic:1509"]
33vB may well be worth fretting over in the future.
[/quote]

33WU, or <9vB. I don't find byte pinching over hypotheticals very persuasive but other's mileage may vary.

[quote="jamesob, post:73, topic:1509"]
This is probably fringe, but one “nice” thing about bare CTV legacy scripts is that because they don’t have an associated address
[/quote]

I know we differ on this but I expect very little bare CTV usage vs some form of script. We can't protect people in script hash land; I think that ship sailed a long time ago.

---
The proposal is materially different than before with CSFS added. Let's avoid sunk cost thinking.

-------------------------

jamesob | 2025-06-10 01:10:30 UTC | #76

Another more practical point in favor of making CTV available in witness v0 is that many (or most?) HSMs do not yet natively support Schnorr signatures - and for existing installations, may never. There is likely overlap between companies who currently use such HSMs (making witness v0 the default) and companies who would like to make use of CTV vaults.

-------------------------

instagibbs | 2025-06-10 15:38:09 UTC | #77

That's a reasonable point, under the restricted set of functionality that encompasses a hash-assertion and a disjoint signature over the transaction itself.

So you could have a "next tx" transition or override with some (non-aggregated) key or require both.

-------------------------

jamesob | 2025-06-18 02:54:18 UTC | #78

Brief overview of my current thoughts, for whatever they're worth: 

I'd be happy to modify BIP-119 to omit legacy script support for CTV if there's a 34-byte "witness v2" along the lines of what @instagibbs sketched out.

My hold out here had been that I thought the BitVM "sibling input requirement" trick relies upon legacy use, but it does not. Note that I still think CTV availability in witness v0 is worth preserving for reasons mentioned above.

If there's demand to introduce a push-style `OP_TEMPLATEHASH` that puts the CTV digest on the stack, that sounds like a totally reasonable and low-effort/risk enhancement to me.

---

Do the stakeholders opining here think there could be consensus around this? Namely:
- BIP-119 modified to remove legacy script support (but retaining wit v0 support),
- definition of a witness v2 per [instagibb's patch](https://github.com/instagibbs/bitcoin/commit/9b5ae77d97222efc2328c1916f790c7f67cf10b3), and
- inclusion of a tapscript-only `OP_TEMPLATEHASH`?

-------------------------

sjors | 2025-06-23 15:27:04 UTC | #80

[quote="instagibbs, post:7, topic:1509"]
I do think it would accelerate updating to PTLCs, f.e., even if symmetry per-se is never adopted. Re-bindable signatures just makes life so much less of a headache when stacking protocols…
[/quote]

Can you clarify the situation with regards to PTLC (without LN Symmetry)? My understanding is that PTLC is an enormous privacy upgrade, since it removes the trail of identical hash pre-images along a (split) route. (especially in a scenario where an adversary is gradually collecting log files through nefarious means)

I guess I'm stuck in 2021 when we thought that just Taproot would do the trick.

Is it merely easier if we add CSFS (plus CTV)? Or is it nearly impossible without that?

Since it tends to take a few years for a soft fork to activate and for a protocol like Lightning to take advantage of it, by that time, could we be in situation where already achieved PTLC the hard way? It would still be a welcome simplification, but not introducing something new. Or do you reckon we'll have PTLC sooner despite the extra step of activating a soft fork?

-------------------------

instagibbs | 2025-06-23 15:37:14 UTC | #81

[quote="sjors, post:80, topic:1509"]
Is it merely easier if we add CSFS (plus CTV)? Or is it nearly impossible without that?
[/quote]

Take a look [here](https://gist.github.com/instagibbs/1d02d0251640c250ceea1c66665ec163) to see what complications arise from PTLCs on today's protocols. It's by no means impossible, but with rebindable signatures it just gets significantly simpler.

I will not try to predict uptake on established protocols however. There's a long list of improvements that users actually care about, and PTLCs isn't one of them generally.

-------------------------

ajtowns | 2025-07-02 05:06:22 UTC | #82

[quote="sjors, post:80, topic:1509"]
I guess I’m stuck in 2021 when we thought that just Taproot would do the trick.
[/quote]

I think there's also a lack of tooling/standardisation for doing a PTLC reveal in combination with a musig 2-of-2 (which would be efficient on-chain), or even general tx signatures (ie `x CHECKSIGVERIFY y CHECKSIG`).

The efficient way of doing PTLCs would be to have a partially-presigned musig2 signature for the redeem tx, where completing the signature reveals the PTLC secret to the other party. But this would require adaptor signature support for musig2, and that's not part of the spec and was removed from the secp256k1 implementation see [pr#1479](https://github.com/bitcoin-core/secp256k1/pull/1479). Doing it less efficiently as a separate adaptor signature would work too, but even plain adaptor signatures for schnorr sigs also isn't available in secp256k1.

These also aren't included even in the more experimental secp256k1-zkp project, see [pr#299](https://github.com/BlockstreamResearch/secp256k1-zkp/pull/299), though secp256k1-zkp does have support for ECDSA-based adaptor signatures ([pr#117](https://github.com/BlockstreamResearch/secp256k1-zkp/pull/117)) and adaptor signatures in the old musig scheme that predates musig2 and [BIP-327](https://github.com/bitcoin/bips/blob/36618d1c3c6b1559d0ce69fd958191b8789f350a/bip-0327.mediawiki) (see [musig.md](https://github.com/BlockstreamResearch/secp256k1-zkp/blob/6152622613fdf1c5af6f31f74c427c4e9ee120ce/src/modules/musig/musig.md)).

If the tooling were ready, I could see PTLC support being added as a "let's get it in early so it's already widely supported when we actually want to enable it", but I don't think anyone considers it a high enough priority to put in the work to get the crypto stuff standardised and polished. Making the unhappy path more efficient is something that could be done later on a per-peer basis without too much hassle.

Having CAT+CSFS available would avoid the tooling issue, at a cost in on-chain efficiency (though only in the unhappy path, of course), using 102 witness bytes for a PTLC reveal versus 56/67 witness bytes for a HTLC reveal. In particular, the script `<R> CAT <G> DUP CSFS` can be satisfied by having `<s>` on the stack where `s*G = R + H(R,G,G)*G`, so the preimage of `R` can be calculated as `r = s - H(R,G,G)`, with straightforward ECC maths, and no need for secret keys. I think with only CSFS available you continue having similar tooling problems, because you need to use adaptor signatures to prevent your counterparty from choosing a different R value for the signature.

(These issues are independent of the update complexity and peer protocol updates @instagibbs describes above)

-------------------------

instagibbs | 2025-07-02 13:37:58 UTC | #83

[quote="ajtowns, post:82, topic:1509"]
In particular, the script `<R> CAT <G> DUP CSFS` can be satisfied by having `<s>` on the stack where `s*G = R + H(R,G,G)*G`, so the preimage of `R` can be calculated as `r = s - H(R,G,G)`, with straightforward ECC maths, and no need for secret keys
[/quote]

To be clear, does this include per-hop blinding? My PTLC context has completely paged out of memory.

-------------------------

ajtowns | 2025-07-02 16:59:32 UTC | #84

Per-hop blinding just means that you have inbound value `R1` and outbound value `R2=R1+kG`, where `k` is a per-hop constant known only to you and the person initiating the payment, so when you receive/calculate `r2` on your outbound link, you can calculate `r1=r2-k` to claim the inbound funds. You just need to relate r1/R1 (and r2/R2) on-chain, you don't need to relate r1/r2 or R1/R2 on-chain (which would remove the privacy benefit anyway).

-------------------------

