# Some problems with current bitcoin core `MODERATION.md`

ariard | 2025-04-19 01:13:47 UTC | #1

Following some [moderation incident](https://github.com/bitcoin/bitcoin/pull/31989#issuecomment-2702330203) on the proposal to add CTV support on bitcoin core, I’ve [opened a PR](https://github.com/bitcoin-core/meta/pull/15) to remove the moderation guidelines (the `MODERATION.md` floating in the
`bitcoin-core-meta` repo). I don't think bitcoin core should have mod guidelines, not concerning consensus rules.

Philosophically, in consideration of bitcoin history with the block size war, there should be no formal governance, or even the gist of some "*Impatience led various participants to advocate*
*models that would privilege high-profile actors and grant them control over the direction of*
*the protocol*". See the [The Tao of Bitcoin Development](https://medium.com/@bergealex4/the-tao-of-bitcoin-development-ff093c6155cd) essay from 2017. Yes you will find the “project management" in the `.md`, so it's unclear what's the goal.

Legally, there is something called the [Berne Convention of 1886](https://en.wikipedia.org/wiki/Berne_Convention), it's an IP treatise with 170+ countries are contracting parties. Almost all the historical or active development or organization stakeholders contributing to bitcoin dev are located in those countries. In its [article 7bis](https://www.law.cornell.edu/treaties/berne/7bis.html), it's protecting what's called joint authorship. Yes, joint authorship can be applied to open source projects. See [A Theory of Jointauthorship for Free and Open Software Projects](https://ctlj.colorado.edu/wp-content/uploads/2018/09/3-Chestek-6.20.18-FINAL.pdf) from some 2017 FSF workshop.

The `.md` has been acked by [~13 contributors](https://github.com/bitcoin-core/meta/pull/13#issuecomment-2748178454) only, with somehow no respect for the hard
contributed work of the ~1200 contributors that can be fetched from bitcoind git log, and as such there eventual right to consent or not to the `MODERATION.md`.

Legally more, there is a UK judicial ruling of 2023, to which few formers or current maintainer(s) have been defendants ([Tulip Trading Limited v Van Der Laan and ors [2023] EWCA Civ 83](https://www.judiciary.uk/wp-content/uploads/2023/02/Tulip-v-Van-Der-Laan-judgment-030223.pdf)). Within, they made the following defense "*They contend that they have nothing like the power or control*" on the bitcoin network and go to argue "*they are part of a very large, and shifting, group of contributors without an organisation or structure*" (quote from the judicial ruling). The ruling explicitly considers Github, "*with the relevant electronic password for the particular code database on GitHub*”.

Now currently, there is a `.md`, with some specific contributors ("*the moderators*" and "*the maintainers*”) that can enforce [behavioral guidelines](https://github.com/bitcoin-core/meta/pull/14#issuecomment-2753889160) on other contributors, decide what is "*on-topic*" / "*off-topic*", decide what is related to "*project management*" and what is related to "*technical issues*"...I don’t know if it's “*power*" or “*control*", though it doesn't sound exactly like "[janitorial roles](https://bitcoincore.org/en/about/)”…

Somehow, there is a logical problem somewhere.

I care about civility and courtesy in an internet community of contributors spread over the whole world, and treating with fairness and respect any contributor whatever the social background, however I don't think at all the current `MODERATION.md` flies very well.

Opening this as a "meta", as I don't see why it's not a meta subject as long as it’s discussed with politeness and kindness. And it's of interest to everyone in bitcoin.

-------------------------

