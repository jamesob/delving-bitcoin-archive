# Bitcoin Forking Guide

ajtowns | 2025-02-14 14:40:05 UTC | #1

I appreciated the perspective the [bcap](https://github.com/bitcoin-cap/bcap) project offered regarding consensus changes within Bitcoin, but one thing I found a bit dissatisfying was that it lacked specific recommendations on how you should do things, whether as someone proposing consensus changes, or as one of the "stakeholders" who might want to exercise some influence over proposed changes.

But I guess 2025 is the year of "you can just do things", so why not be the change I want to see in the world, and write my own guide to making consensus changes in Bitcoin? That also lets me tweak bcap's list of stakeholders into my preferred set of roles. So here it is:

https://ajtowns.github.io/bfg/

If you prefer to TLDR, the key idea is in the first paragraph:

> This guide advocates the approach of ***consensus before consensus***;
namely that it is important to establish social consensus in favour
of a change prior to changing the consensus code that will make the
change reality.

It's not intended to be the be-all and end-all of how to change Bitcoin -- after all, that's not a goal that's compatible with a decentralised system in the first place.

As such, there's obviously no teeth, per se, to any of its recommendations. At best the only real potential enforcement of any of its recommendations are either your conscience saying "you know, that recommendation is probably a good idea, maybe you should actually do it that way", or other people saying "you seem like you're skipping major steps here; that seems shady, so I don't think I can support you". But conversely, if you're skipping steps because you have good reasons, that's fine: it's a guide, not a cop.

Beyond that, it's also only a fairly high-level guide. In practice, following the steps outlined will require drilling down into a lot of fiddly details at every stage, and doing a lot of work to get all those details right. This isn't a shortcut for how to have your way with a two-trillion dollar asset.

And finally, it's only a guide to the cooperative path, where you make a change that makes everyone's life better, and more or less everyone ends up agreeing that the change makes everyone's lives better. It doesn't provide a path to activation for changes that want to make a more utilitarian calculation where more some people might lose, but more people benefit (apart from saying that until that changes they don't achieve consensus, and so get stuck forever in the early phases).

Anyway, I wanted to put forward some concrete, constructive thoughts on what I think is a sensible approach to changing bitcoin looks like, and that's it.

-------------------------

ariard | 2025-02-16 21:49:14 UTC | #2

From a brief read, I think the guide is summarizing well how social consensus can be build by the champions of any consensus change before reaching technical consensus.

> Comparing your idea to similar ideas from the past

> Documenting the tradeoffs of your idea versus other approaches

I believe this is where we have been mostly been stuck as a community at the very large, especially for covenants and contracting primitives in the last years. There is a multitude of ideas on how to do covenant the Right Way, and of course every time the primitive design is influenced by the use-case(s) thought off.

That phenomena is easy to point out to the 3 or 4 payments pools designed that have been proposed where according to the off-chain construction semantics wished, the primitive expressivity has changed. This kind of phenomena generally generate a lot of frustration among the bona fides devs and stakeholders involved and it has even led some to claim that the consensus process is "**memoryless**" as the same arguments are re-hashed again and again
from year to another. This is not completely wrong as a viewpoint.

In my view, we are suffering less than a lack of consensus change methodology and more of a lack of a high-quality technical **archive**, from which researcher and protocol dev in your terminology can converge on same set of experiences, ideas and frameworks argued on. Said archive maintained and established with the highest degree of objectivity and with the best of human ability to do things with a no partisan and disinterested mindset.

There is a page of the bitcoin wiki which is listing people's opinion on a bunch of soft-forks, though it's very **people-centric** rather than being **ideas-centric**. I.e pointing out what are the trade-offs among all the soft-forks proposals or what use-cases are enabled by the proposals, and then having people's taking positions w.r.t the ideas.

There has been attempt with the [Bitcoin Contracting Primitives WG](https://github.com/ariard/bitcoin-contracting-primitives-wg) in the past, where public IRC meetings were organized for people to speak about their different levels of understanding of a proposal and there was also a [similar attempt](https://github.com/bitcoinops/bitcoinops.github.io/pull/806) by the Optech folk to extensively document covenants.

(-- I do not plan to resurrect the Bitcoin Contracting Primitives WG, no time and no interest, though if optech or others with sufficient credibility are thinking to do the hard work of maintaining a public online archive to document proposed consensus change I believe it would be welcome as a nice initiative).

-------------------------

