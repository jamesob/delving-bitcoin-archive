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

