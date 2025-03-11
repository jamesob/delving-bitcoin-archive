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

