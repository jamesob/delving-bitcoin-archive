# Softfork before GTA VI?

cmp_ancp | 2026-09-24 18:15:36 UTC | #1

Hi all

Something that really bothered me is the sudden loss of momentum we had for pushing a new softfork for bitcoin. Some years ago the covenants topic was really hot, the entire community was pushing towards a consensus.

And then, the attention shifted to ordinals, inscriptions, policy, knotz vs core, finally ending in a hardfork. At the same time, the community really focused in a post quantum softfork, and I mean, that's great, but we are not even sure if quantum computing will ever be a menace.

I think the community already reached a point of considering covenants a given, multiple protocols have variations on "when covenants, we will do that", "when rebindable signatures, we will do that", most notably Ark and LN. But covenants are not a given, we don’t have it today, and we are already full planning on the next softfork.

I think we could, as a community, use this post hard fork moment of unity to reach some consensus and regain that momentum. I am not stating preference between TEMPLATEHASH or CTV, I am well with either, but we need to set that attention. GSR, CCV, all sort of interesting building blocks that could be a bit more on the lights.

I know there is some frequent talking about BIP54, but what are our specific planning on that? It will be released alone? Or will it come in a package with rebindable opcodes and so? Because, if we normalize half decade long for each softfork, what are the implications on the selection of focus?

I'm just sharing some thoughts, want to know what you think.

-------------------------

garlonicon | 2026-09-27 10:41:01 UTC | #2

> Softfork before GTA VI?

I think it is unlikely to see next soft-forks in the future, unless someone will push them forward. I remember times, when touching OP_CAT was a no-go zone, however, when the discussion started, and [BIP-347](https://github.com/bitcoin/bips/blob/02bebeb5ef53dc21a52d5706c6b4c17b6af6c1cc/bip-0347.mediawiki) was created, then suddenly it became more reachable, and some people suddenly shifted their position from "it will never happen" to "it might be activated, even if I still disagree with it".

> But covenants are not a given, we don’t have it today, and we are already full planning on the next softfork.

You know, it is always a choice between preserving status quo, and not activating new things, or launching new features, and facing all consequences of that. However, even if new soft-forks will never be deployed, then some people will still try to hack the system, and build new things anyway, by using tools, which are already there.

One example of that is what happens with sidechains: we don't have [BIP-300](https://github.com/bitcoin/bips/blob/02bebeb5ef53dc21a52d5706c6b4c17b6af6c1cc/bip-0300.mediawiki). It probably won't be activated, because some people are simply against sidechains, because of Miner Extractable Value, and other issues. However, the consequence of not having decentralized sidechains, is that we will have centralized ones. And people will try to hack the system anyway, for example by using OP_SIZE on DER signatures. Of course, there is no Merged Mining yet, but well, maybe other people will hack it later, by finding next hole, and abusing it to implement yet another feature. Or: if features like OP_CAT will be there, then we will have not only sidechains, but also much more things.

> I know there is some frequent talking about BIP54, but what are our specific planning on that?

Some mining pools are already producing BIP-54-compatible blocks. And there will be more. If nothing will break, then I guess it will be activated quite smoothly.

> It will be released alone?

Yes, I guess that's the current plan.

> Or will it come in a package with rebindable opcodes and so?

It is easier to reach consensus for a change, where mining pools are already supporting it, and it is quite small and non-controversial, than add more things into it, and risking not having BIP-54 at all.

> Because, if we normalize half decade long for each softfork, what are the implications on the selection of focus?

People will just try to read the code more carefully, and find hacks, which would give them more features anyway. It is similar to templates in C++, and metaprogramming: it was not deployed. It was discovered in a hackish way, because the language didn't support, what users wanted. And now, we have to live with that quirky syntax, just because of historical reasons. I guess the same will happen to sidechains: there are too many people against them, to see it activated in a normal way, so it will happen as a side effect of other features.

-------------------------

ZmnSCPxj | 2026-10-21 14:29:02 UTC | #3

Nah make it GTA VII

There's a strong reason for why Bitcoin has already ossified.

SegWit softfork showed that it is control of UTXOs that is necessary to keep control of the direction of Bitcoin.  It was called "Economic Majority Rule".

In reaction to that, those who wanted to control Bitcoin started creating large custodial wallets, especially those that are then offered as shares on ETFs on various stock markets, a largely untapped market just after SegWit.

With control of large numbers of UTXOs, they can keep control of Bitcoin, ***INCLUDING*** blocking covenants.

Covenants would allow expanding the capabilities and usability of existing technologies that are NOT inherently custodial, like Lightning Network.  Remember, the biggest blocker for most people is that non-custodial Lightning has a number of pain points in useability and cost, which custodial services amortize by creating a few large channels rather than having lots of tiny noncustodial wallets creating many tiny channels.  More ability to scale with covenants, the smaller the moat of custodial services.  Thus, it is to the interest of these large custodial wallets to ***BLOCK*** covenants.  And they can block covenants by a combination of diversion tactics (like the inscriptions war and post-quantum signatures non-issue) and simply discouraging ***EVERY*** attempt at pushing covenants, in favor of retaining custody of control over the future of Bitcoin.

Not ***our*** keys, not ***our*** Bitcoin protocol.

-------------------------

ariard | 2026-10-22 03:30:45 UTC | #4

Hello,

On the loss of momentum to land covenants, the design and development process of soft-forks are far to be linear, as during such process people might find *actual* vulnerabilites worthy to fix on the soonest timeline possible. E.g back when there was some work to move forward with APO in 2022, while reviewing a newer APO-enabled version of Lightning channels, it turns out we found the so-called attack dubbed "Replacement Cycling Attack" which was affecting almost anything that would be a good candidate for APO.

During the mitigation process of such attack of such process, the question was floating among the people involved that it might necessary to do a soft-fork to correct this vulnerability cleanly. The question arise publicly once the vulnerability was disclosed, though somehow the existence of such vulnerability was a source of discouragement to slow down or halt the progress on APO.

We can put the work, ressources and effort to advance on any soft-fork change, though there is never guarantee that during the process sadly substantial technical reasons won't appear to slow down or put it in the background. Severe security issues first over soft-fork progress in my view.

Now, to answer your questions on BIP54 I did share my thoughts recently here (https://groups.google.com/g/bitcoindev/c/TBBakeBK8_E). I think anyone in the world is welcome to participate to process, be it reviewing the BIP, one of the implementation in a full-node ( https://github.com/bitcoin/bitcoin/pull/35793 ) or working on quality assurance to level up the bar for it. If you don't have the analytical skills or know-how to bring more value to the review process than a edge LLM, that's fine I think there are also community or social engagement thing that can be done ( https://github.com/bitschmidty/bip54/issues/15 ).

*Ask not what the community of devs can do to advance the soft-fork you wish - -ask yourself what you can do join the community of devs to advance the soft-fork you wish (to paraphrase some famous quote).*

PS: I've never seen or heard any attempt of a custodial service to block the progress of covenants, I'm not saying that it has never happend, though I've never had a report first-hand of such thing, and I have good memory.

-------------------------

