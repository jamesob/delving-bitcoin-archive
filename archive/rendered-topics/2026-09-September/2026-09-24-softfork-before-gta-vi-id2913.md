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

