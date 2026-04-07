# BIP-110 update: v0.4.1 release and implementation submitted to Bitcoin Core

dathonohm | 2026-03-31 23:24:03 UTC | #1

# BIP-110 update: v0.4.1 release and implementation submitted to Bitcoin Core

Hi all,

Following up on the BIP-110 proposal from [October](https://groups.google.com/g/bitcoindev/c/nOZim6FbuF8), I am announcing the most recent stable release (v0.4.1) and the submission of an implementation of BIP-110 to Bitcoin Core.

## Background

BIP-110 was [initially](https://groups.google.com/g/bitcoindev/c/YO8ZwnG_ISs/m/NpQfYlGLAgAJ) [proposed](https://groups.google.com/g/bitcoindev/c/nOZim6FbuF8) in October 2025, and the first version of the [official activation client](https://github.com/dathonohm/bitcoin/releases/tag/v29.3.knots20260210%2Bbip110-v0.4.1) was released at the end of January as a [fork of Bitcoin Knots](https://github.com/bitcoinknots/bitcoin/pull/238). It was [accepted into the BIPs repo](https://github.com/bitcoin/bips/pull/2017) shortly thereafter.

For those new to BIP-110, it is a proposal to temporarily limit the size of data fields at the consensus level, in order to reject the standardization of data storage as a supported use case by consensus. It achieves this by temporarily limiting all officially supported methods of data storage (and methods that could be or become perceived as officially supported) to 256 bytes or less, while preserving all known monetary use cases.

This has several desirable effects, including but not limited to:

- Reducing demand for arbitrary data storage by frustrating attempts to monetize and gamify arbitrary data.

- Bringing consensus and policy back into closer alignment, by restoring still-popular policy limits from a few years ago in consensus.

- Protecting the decentralization of the node network by eliminating unnecessary costs and (in some cases, extreme) risks introduced by making arbitrary data storage an officially supported function of Bitcoin.

- Making Bitcoin better money by preventing Bitcoin payments from having to compete at an unfair disadvantage against gamified data spam.

## Activation client v0.4.1

The official BIP-110 activation client is currently `bitcoin-29.3.knots20260210+bip110-v0.4.1`, a fork of Bitcoin Knots 29.3. While a few bugs were found early on, they were all swiftly resolved and the code has now been stable for several weeks.

### Consensus rules (when active)

- scriptPubKeys limited to 34 bytes (83 bytes for OP_RETURN)

- OP_PUSHDATA\* payloads and witness stack elements limited to 256 bytes (exception: BIP16 redeemScript)

- Spending undefined witness/Tapleaf versions is invalid

- Taproot annex is invalid

- Taproot control blocks limited to 257 bytes (128 script leaves)

- OP_SUCCESS\* opcodes in Tapscripts are invalid

- OP_IF/OP_NOTIF in Tapscripts is invalid

- Empty witness required for P2A spends

Inputs spending UTXOs created before the activation height are exempt (this is also known as "UTXO grandfathering").

Rules expire automatically after ~1 year (52416 blocks).

### Deployment

- Signaling bit: 4

- Threshold: 55% (1109/2016)

- Start time: ~December 1, 2025

- Max activation height: 965664 (~September 1, 2026)

- Mandatory signaling enforced in the retarget period before mandatory lock-in

### Downloads (Guix reproducible builds)

**Binaries:** [GitHub Release](https://github.com/dathonohm/bitcoin/releases/tag/v29.3.knots20260210%2Bbip110-v0.4.1)

- Linux: x86_64, aarch64, arm, powerpc64, powerpc64le, riscv64

- macOS: arm64, x86_64

- Windows: x86_64

**Node platforms:**

- Umbrel: Available in the Umbrel app store ([PR #5044](https://github.com/getumbrel/umbrel-apps/pull/5044))

- MyNode: Available in MyNode ([PR #996](https://github.com/mynodebtc/mynode/pull/996))

- Start9 (StartOS v0.3.5.1): Sideload .s9pk package ([Release](https://github.com/dathonohm/knots-startos/releases/tag/v29.3.knots20260210%2Bbip110-v0.4.1))

- Start9 (StartOS v0.4): Sideload .s9pk package ([Release](https://github.com/dathonohm/knots-startos/releases/tag/v29.3.knots20260210%2Bbip110-v0.4.1-startos0.4.0-1))

- Parmanode: Available in Parmanode

- Ubuntu PPA (22.04, 24.04, 25.10, 26.04): [PPA](https://launchpad.net/~dathonohm/+archive/ubuntu/bitcoinknots+bip110)

## Bitcoin Core implementation

The activation client has proven immensely popular, and now comprises over [8% of listening nodes](https://bitnodes.io/nodes/?q=BIP110) after just 2 months. While BIP-110's mandatory signaling period is not until August, it allows for early activation if 55% of hashpower signals support, so activation could be as soon as a few weeks from now. Indeed, 4 blocks signaling for BIP-110 have already been mined during the month of March.

Due to BIP-110's rising popularity, it seems prudent to augment Core with the ability to enforce the new rules, so that once BIP-110 is activated, Core users will remain secure because their nodes will follow the valid chain. Accordingly, I have submitted an implementation of BIP-110 to Bitcoin Core, split across two PRs:

- **PR 1** - Versionbits extensions for BIP-110: [bitcoin/bitcoin#34929](https://github.com/bitcoin/bitcoin/pull/34929)

- **PR 2** - ReducedData Temporary Softfork (BIP-110/RDTS): [bitcoin/bitcoin#34930](https://github.com/bitcoin/bitcoin/pull/34930)

Both PRs were automatically closed by a bot shortly after submission. I am working to get them reopened. In the meantime, review and discussion can take place on mirror PRs:

- **PR 1** (mirror): [bip110-core/bitcoin#1](https://github.com/bip110-core/bitcoin/pull/1)

- **PR 2** (mirror): [bip110-core/bitcoin#2](https://github.com/bip110-core/bitcoin/pull/2)

The Core PRs are derived from the activation client's [Knots branch](https://github.com/bitcoinknots/bitcoin/pull/238). Some refactoring was required to accommodate recent upstream changes to Core's BIP9 implementation.

The implementation totals ~375 lines of non-test C++/H across both PRs -- roughly a third the size of CSV (BIP 68/112/113). Every commit builds and passes all tests independently.

## Review

I welcome technical review on the mirror PRs linked above, or on the upstream PRs once they are reopened.

My sincere hope is that BIP-110's activation will heal the divide in our community and allow us all to get back to building Bitcoin into history's greatest money.

Sincerely,

Dathon Ohm

-------------------------

ariard | 2026-04-01 20:33:45 UTC | #2

From the [mailing list](https://groups.google.com/g/bitcoindev/c/nOZim6FbuF8/m/kXtE07UEAwAJ).

> \> as for anyone who is following the work of the Electronic Frontier Foundation from times to times, there is a pending case in front of the US Supreme Court, Cox Communications, Inc vs. Sony Music Entertainment specifically on the liability of Internet service providers.

For the one who are following this kind of topic, in Cox vs Sony [the decision](https://www.supremecourt.gov/opinions/25pdf/24-171_bq7d.pdf) has been yield over the last weeks, where the USC has withheld a lack of liability for internet service providers in the context of copyright infringement (“*This Court has repeatedly made clear that mere knowledge that a service will be used to infringe is insufficient to establish the required intent to infringe*”). I’ll leave the details to the US constitutional law specialists, of which I’m not as a I never clerked for a justice, but the concurrent opinion is also interesting as it’s analyzing distributed peer-to-peer software and for a wider scope.

In my view, in the light of this novel jurisprudence, I think the legal worries of the BIP110 proponent have been far overblown. In its current state, [the current BIP110 text](https://raw.githubusercontent.com/bitcoin/bips/refs/heads/master/bip-0110.mediawiki) is still indirectly mentioning those legal worries half-word (i.e “*The problem becomes even worse when the data is objectionable to node operators*”). Might I suggest to edit the BIP in a sense to completely remove this mention, as with now there is more legal clarity on the matter (“objectionable” is a terminology directly coming from US administrative law doctrine on communication broadcast).

In my opinion, in some measure, we shouldn’t as protocol developers consider “exogenous” reasons in the design and development of consensus rules. Notably, among others reasons, to avoid introducing security bugs in the network due to ill-written legal texts.

I do not wish to sound too much dismissive of the BIP110 legal concerns (anyone is free to go to read the USC’s latest decision to make its own opinion, or the equivalent EU’s jurisprudence fwiw). Indeed zero legal risk can never be a thing, though here I can only share [the measured and considered take](https://www.eff.org/pages/legal-faq-tor-relay-operators) of the EFF on running Tor nodes.

Excerpt:

“*Can EFF promise that I won’t get in trouble for running a Tor relay*?”

”*No. All new technologies create legal uncertainties, and Tor is no exception. We cannot guarantee that you will never face any legal liability as a result of running a Tor relay*”.

In my opinion, in some measure, we shouldn’t as protocol developers consider “exogenous” reasons in the design and development of consensus rules. Notably, among others reasons, to avoid introducing security bugs in the network due to ill-written legal texts.

In my since belief, the public debate among the community on the adoption or not of BIP110 would be of a far higher intellectual quality if wasn’t repeatedly poisoned by ungrounded legal claims on factual elements that said BIP110 proponents wish to be take in account for the evolution of technical consensus rules. Such attitude, in my personal opinion, can only lead any reasonable observer to doubt if they have real technical arguments after all or if it’s not an attempt to influence the public debate with deceptive tactics.

For clarity, I do not support BIP110, less for the technical issues it aims to address, and more because I think the peer-to-peer network stability is a valuable end goal in itself. We are not building a stronger ecosystem by bothering and burdening the wallet developers to have to adapt their stuff, or even forcing them to have to go through the myriad of BIP110 restrictions just to be sure that their wallets softwares, even if there are wallets only for simple funds transfers, do not have to be changed.

Discussing the categories of use-cases, the chain ressources consumptions, if we should favor or deprioritize some with weight penalties, etc all sounds legit technical discussions to have in my belief. After all, there are some unanswered real technical problems, e.g  what we do if in the future the “vaults” off-chain transactions are over-bidding the blockspace at the very same time there is a massive closure of lightning chans.

The discussions can only be better if they are made on qualitative and quantitative arguments on adequate public forums where not only developers and maintainers are respected in the exercise of their (difficult) janitorial roles, but also where “exogenous” considerations like half-baked legal worries have been structurally ruled out.

Those considerations, if followed, in my humble view, can only favor a more serene and courteous public conversation.

-------------------------

neonrooks | 2026-04-02 05:39:31 UTC | #3

Before supporting changes to Bitcoin - especially at consensus level - I would like to get some clarification about the “desirable effects” of BIP-110.

[quote="dathonohm, post:1, topic:2360"]
Reducing demand for arbitrary data storage

[/quote]

What gain to users is achieved whether blocks are full or not? The current rules allow for only a maximum growth of the blockchain at a linear rate.

[quote="dathonohm, post:1, topic:2360"]
Bringing consensus and policy back into closer alignment

[/quote]

Who decides which policies to uphold? This should be the decision of each node by its own mempool policy, no?

[quote="dathonohm, post:1, topic:2360"]
Protecting the decentralization of the node network by eliminating unnecessary costs and (in some cases, extreme) risks introduced by making arbitrary data storage an officially supported function of Bitcoin

[/quote]

Can you define the unneccessary costs and risks of running a node in this scenario? We should be clearly informed of these issues.

[quote="dathonohm, post:1, topic:2360"]
Making Bitcoin better money by preventing Bitcoin payments from having to compete at an unfair disadvantage against gamified data spam

[/quote]

Can you demonstrate an example of payments being unfairly disadvantaged?

Thank you for your time.

-------------------------

ArmchairCryptologist | 2026-04-02 09:04:42 UTC | #4

[quote="dathonohm, post:1, topic:2360"]
The activation client has proven immensely popular, and now comprises over [8% of listening nodes](https://bitnodes.io/nodes/?q=BIP110) after just 2 months.

[/quote]

Even disregarding all the deficiencies with BIP110 which have been thoroughly elaborated upon elsewhere, I have no faith whatsoever in those numbers being representative of anything but a handful of people spinning up nodes to pad the numbers.

First of all, of the 1905 nodes registered on Bitnodes that advertise BIP110, more than 1300 are onion nodes. Creating an unlimited amount of onion addresses and pointing them at a node, then submitting that address to Bitnodes, is effectively free. As such, if anyone were to accept use Bitnodes stats for arguments of node distribution - and that’s a big if - the only meaningful number would be counting the nodes that have some semblance of cost, which would limit it to IPv4 nodes. Which would bring the number down to \~420 out of 7242, or 5.7%

Secondly, even those numbers do not at all match the distribution of subver strings that connect to my four public listening nodes. As of right now, I’m seeing:

* 1026 nodes running Core
* 141 nodes running Knots without BIP110
* 16 nodes running Knots with BIP110

So 16 out of 1167 nodes are advertising BIP110 in the subver, or 1.4%. Assuming that all nodes have an equal number of outgoing connections and an equal chance to connect to any of my nodes, this statistically gives a greater than 99% confidence that the real number is between 0.5% and 2.3%.

I put a [raw data dump on pastebin](https://pastebin.com/KdSGnqKR) for anyone interested. I filtered out all subvers without “Satoshi” in the string, which mostly limits it to Core and Knots nodes.

There is a reason soft forks are activated by mining power alone; node listen counts will always be suspect, and unlike node sockpuppeting, mining power cannot be falsified.

-------------------------

dathonohm | 2026-04-07 03:29:28 UTC | #5

Greetings, @ariard.

I am aware that many developers do not consider toxic data to be a serious threat to Bitcoin. However, as you may be aware, there are many developers, lawyers, and other Bitcoiners who disagree. For example, [the researchers who actually studied the topic](https://fc18.ifca.ai/preproceedings/6.pdf) concluded that "We thus believe that future blockchain designs must proactively cope with objectionable content. Peers can, e.g., filter incoming transactions or revert content-holding transactions, but this must be scalable and transparent."

Bitcoiners have always engaged in adversarial thinking in order to anticipate novel attacks before they happen. This was the basis for rejecting a hardfork to a larger block size in 2017, for example. Many "Big Blockers" considered large blocks to be harmless, and criticized the "Small Blockers" for overestimating the concern. But since Bitcoin is conservative and does not unnecessarily impose risks on itself, it is simply common sense to reject the standardization of arbitrary data storage, just as it was common sense to reject a hardfork to a larger block size. The risk, as it turns out, is the same in both situations: that fewer people will operate nodes.

It does not require much imagination to envision Bitcoin dangerously centralizing due to becoming a popular method of storing toxic data. Most of humanity consists of moral actors and is repulsed (if not frightened) by the idea of aiding in the distribution of such content. Officially embracing arbitrary data storage, as Core 30 has done, makes node operators complicit in this activity because if arbitrary data is an official use case of Bitcoin, then by operating a node, users are *explicitly opting into* a system where anyone can store any file up to 100 kilobytes, completely unencrypted and impossible to delete without compromising the node operator's entire motivation for operating a node, which is independent verification of the transaction history. (On the other hand, if we activate BIP-110 to reject arbitrary data storage, it can easily be argued that node operators are not complicit, because the content must, by consensus, be hidden from the user by disguising it as financial data.)

In light of this, it is incredibly reckless to move forward with embracing data storage as an official use case. As you are aware, very few Bitcoin users actually operate nodes, and this is an existential problem for Bitcoin. Why make their lives even harder by forcing them to participate in activity *completely unrelated to money*, that they most likely consider immoral and/or dangerous, in order to use Bitcoin without a trusted intermediary? The inevitable result of this will be centralization, because only the tiny minority of humanity with no moral scruples will even be *candidate* node operators.

It is abundantly clear, both from recent spam waves and from the threat of toxic data, that arbitrary data poses at least a serious risk to Bitcoin's decentralized node network and its proper functioning as money, and at most a fatal risk. We don't know the exact level of risk because Bitcoin never officially supported arbitrary data storage before the release of Core 30 (and it arguably still doesn't because the BIP-110 movement began immediately thereafter).

But even in the absence of a fatal threat from toxic data, BIP-110 would still be worth activating because it is such a simple, safe measure, which nevertheless achieves the important goal of making arbitrary data storage officially unsupported on the network.

>Might I suggest to edit the BIP in a sense to completely remove this mention, as with now there is more legal clarity on the matter (“objectionable” is a terminology directly coming from US administrative law doctrine on communication broadcast).

I am not sure why you think only citizens of the United States find certain kinds of content objectionable. Indeed, the researchers who authored the paper linked above are German, and used the term "objectionable content" many times. I don't think you will find a single country whose citizens don't broadly find abhorrent the type of content examined by the German researchers.

>In my opinion, in some measure, we shouldn’t as protocol developers consider “exogenous” reasons in the design and development of consensus rules. Notably, among others reasons, to avoid introducing security bugs in the network due to ill-written legal texts.

Can you clarify this vague reference to "security bugs in the network" that BIP-110 supposedly introduces? Would you consider a desire to avoid centralizing the network via discouragement of node operation to be an "exogenous" motivation?

As for Tor, operating a Tor relay is not at all similar to operating a Bitcoin node. The purpose of Tor is anonymous information sharing, while the purpose of Bitcoin is to be money. In the case of Tor, the relay operator cannot see any of the data that flows across her relay, whereas with Bitcoin, everyone can see all of the data, unencrypted, and no one can ever delete or modify it, and indeed all Bitcoin node operators are forced to continue distributing it on-demand, until the end of time.

It very much makes sense that Tor relay operators are both protected from intent (since they cannot see the data and it does not persist even if they could), and that they understand the risks and accept them in order to participate in the Tor network (since information sharing is Tor's purpose).

Conversely, Bitcoin node operators are neither protected from intent (since everyone can see the data), nor do they consent to the risks of operating an uncensorable, public file server (that must accept and distribute on-demand, any data posted to it, for free forever), since Bitcoin's purpose is to be money and has nothing to do with file storage.

Bitcoin is not Tor. Bitcoin is money.

>I do not support BIP110, less for the technical issues it aims to address, and more because I think the peer-to-peer network stability is a valuable end goal in itself

BIP-110, if anything, improves network stability, by discouraging non-payment spam.

>We are not building a stronger ecosystem by bothering and burdening the wallet developers to have to adapt their stuff, or even forcing them to have to go through the myriad of BIP110 restrictions just to be sure that their wallets softwares, even if there are wallets only for simple funds transfers, do not have to be changed.

While I can sympathise with wallet developers burdened with needing to update to support BIP-110, the node operators must take precedence. Again, Bitcoin must remain decentralized. Wallet devs also needed to upgrade in order to support Segwit, and supporting BIP-110 is *much* simpler than that.

Further, there is only one wallet developer, Nunchuk, that even allows its users to create wallets that might be affected by BIP-110, so when you say that BIP-110 is burdening wallet developers, you are really only referring to this one wallet project. And even Nunchuk's users will likely be able to withdraw their funds after activation because of UTXO grandfathering. Failing that, they will be able to withdraw their funds when BIP-110 expires. The number of known users who will have their funds frozen during BIP-110's one-year deployment so far is zero, and BIP-110 has been publicly discussed in technical circles for over 5 months now.

I think it is safe to say that one wallet developer having to do a little extra work leading up to BIP-110's activation is worth it in order for Bitcoin to remain sustainably decentralized and sustainably money. Indeed, if Bitcoin becomes widely used for file storage and stops being money, what use are wallets, anyway?

In any case, all of this discussion is somewhat moot. The point of my post is not to rehash the conceptual discussion which has already taken place uncountably many times over the past several months, but to update users and developers on the steady progress BIP-110 is making towards activation, and to solicit code review on the Core version of the activation client, since Core will eventually need to merge this code.

-------------------------

dathonohm | 2026-04-07 03:52:35 UTC | #6

Greetings @neonrooks.

>What gain to users is achieved whether blocks are full or not? The current rules allow for only a maximum growth of the blockchain at a linear rate.

Arbitrary data impacts full validation by rapid expansion of the UTXO set which, as demonstrated in the recent spam waves, does not necessarily grow linearly. Disk space is also a concern, though secondary.

>Who decides which policies to uphold? This should be the decision of each node by its own mempool policy, no?

Consensus is "decided" by 100% of nodes.

>Can you define the unneccessary costs and risks of running a node in this scenario? We should be clearly informed of these issues.

These are outlined in [the BIP document](https://github.com/bitcoin/bips/blob/805c9b54f6d38f644d1f9c3ce871e2ea3df1f7d8/bip-0110.mediawiki). Please read it if you would like to understand BIP-110 more deeply. I have also just reiterated them in my [reponse to ariard](https://delvingbitcoin.org/t/bip-110-update-v0-4-1-release-and-implementation-submitted-to-bitcoin-core/2360/5?u=dathonohm). Please let me know if anything is unclear.

>Can you demonstrate an example of payments being unfairly disadvantaged?

Yes, during the recent spam waves, transaction fees often went so high that most monetary use became impractical, due to non-payment spam pricing out payments. This situation is nonsensical for a payment network.

>Thank you for your time.

My pleasure. Feel free to reach out to me [on X](https://x.com/dathon_ohm) if you need any further clarification.

-------------------------

dathonohm | 2026-04-07 04:09:06 UTC | #7

Greetings @ArmchairCryptologist.

>Even disregarding all the deficiencies with BIP110 which have been thoroughly elaborated upon elsewhere

No one has yet brought up a reasonable technical objection to BIP-110.

>I have no faith whatsoever in those numbers being representative of anything but a handful of people spinning up nodes to pad the numbers.

While it is true that support is difficult to gauge *only* from node counts, it would be foolish to claim that node counts don't matter at all, especially given BIP-110's overwhelming popularity among node operators, who have taken to social media in droves to post claims, and often screenshots, of their operation of the activation client. The fact that 4 BIP110-signaling blocks were mined in March alone is also a strong indicator that BIP-110 is rapidly gaining support.

>So 16 out of 1167 nodes are advertising BIP110 in the subver, or 1.4%. Assuming that all nodes have an equal number of outgoing connections and an equal chance to connect to any of my nodes, this statistically gives a greater than 99% confidence that the real number is between 0.5% and 2.3%.

All of the services I know of that count different node software have much higher estimates than this. Forgive me if I am skeptical of your statistics here, especially since you are openly opposed to BIP-110.

>There is a reason soft forks are activated by mining power alone; node listen counts will always be suspect, and unlike node sockpuppeting, mining power cannot be falsified.

Miners are not in charge of Bitcoin, as was demonstrated during the 2017 block size war. The nodes are in charge, and node operators are speaking in favor of BIP-110. I expect BIP-110 to activate early via the threshold, following Segwit's 2017 example.

-------------------------

ArmchairCryptologist | 2026-04-07 07:08:26 UTC | #8

[quote="dathonohm, post:7, topic:2360"]
No one has yet brought up a reasonable technical objection to BIP-110.

[/quote]

There is a [whole thread](https://groups.google.com/g/bitcoindev/c/nOZim6FbuF8/m/fk30wcgJAwAJ) of technical objections on the mailing list, and even more on your [pull request](https://github.com/bitcoin/bips/pull/2017), which you are well aware of. But to sum it up for new readers, the BIP does not accomplish its stated objective since the restrictions are easily bypassed by people who are looking to insert data, so it ends up doing nothing but cause disruption and uncertainty for legitimate usage - i.e., people who actually program their money.

[quote="dathonohm, post:7, topic:2360"]
it would be foolish to claim that node counts don’t matter at all

[/quote]

Which no one has claimed. But fact remains that nodes can be easily spun up in large numbers at little cost, especially in the case of onion nodes, so all claims of community support that are backed by node counts must be taken with a massive dump truck of salt. Again, there are very good reasons for why Bitcoin operates on Proof-of-Work as opposed to Proof-of-Nodes or Proof-of-Tweets.

[quote="dathonohm, post:7, topic:2360"]
All of the services I know of that count different node software have much higher estimates than this. Forgive me if I am skeptical of your statistics here, especially since you are openly opposed to BIP-110.

[/quote]

Accusing people of forging data to dispute a claim is not a reasonable way to garner buy-in for your proposal.

[quote="dathonohm, post:7, topic:2360"]
Miners are not in charge of Bitcoin, as was demonstrated during the 2017 block size war. The nodes are in charge, and node operators are speaking in favor of BIP-110. I expect BIP-110 to activate early via the threshold, following Segwit’s 2017 example.

[/quote]

Widespread community consensus is required for sure, but if a small minority of nodes insist on following rules that are not enforced by a majority of miners, the best they can hope for is that they’ll end up on a minority chain, as opposed to a dead one.

-------------------------

neonrooks | 2026-04-07 08:05:09 UTC | #9

[quote="dathonohm, post:6, topic:2360"]
Arbitrary data impacts full validation by rapid expansion of the UTXO set which, as demonstrated in the recent spam waves, does not necessarily grow linearly. Disk space is also a concern, though secondary.

[/quote]

I asked about blocks; the UTXO set is another matter. 

![utxo1|600x500](upload://2uO2rf8c0Nb1RKVRInjHmz7VuVj.png)

Above is a 3 year chart of the UTXO count, which is currently at 165,086,123 (verified by my own node at block height 944021), down from a high of over 180 million. It appears that the “rapid expansion” has already ended. If we take a conservative estimate of each user controlling just 2 UTXOs, we have at best \~80 million users in the Bitcoin network. Are we to propose changes to Bitcoin once we reach 100 million users? Or 200 million users? Is having many UTXOs harming Bitcoin?

[quote="dathonohm, post:6, topic:2360"]
Yes, during the recent spam waves, transaction fees often went so high that most monetary use became impractical, due to non-payment spam pricing out payments. This situation is nonsensical for a payment network.

[/quote]

The Bitcoin network has faced this many times in the past and will again in the future. Part of a permissionless system is allowing users to waste their money on whatever transactions they choose. Pricing out payments does not make sense - this implies you have the right to confirmed transactions at any time for any fee rate. It is nonsensical to ignore a dynamic fee market for transactions.

Transaction fees are currently low, which can only mean the “spam waves” have ended long before BIP-110. I don’t see a reason for change based on the lack of evidence.

-------------------------

