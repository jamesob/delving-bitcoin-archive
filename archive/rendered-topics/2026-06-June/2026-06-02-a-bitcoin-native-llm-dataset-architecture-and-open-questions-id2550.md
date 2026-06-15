# A Bitcoin-native LLM: dataset, architecture and open questions

Tsua00021 | 2026-06-02 14:11:09 UTC | #1

## Motivation

Bitcoin has one of the richest open-source knowledge bases in existence. Every BIP, every mailing list thread, every Delving post, every annotated transaction on-chain --- all of it is public, structured, and freely available. Yet no LLM exists today that genuinely *thinks* in Bitcoin: one that can parse a scriptPubKey, reason about UTXO ancestry, identify a spending pattern, or recommend an appropriate script construction for a given custody requirement.

The closest attempt is Spirit of Satoshi (Satoshi-7B, fine-tuned Zephyr), but its focus is education and culture --- Austrian economics, orange-pilling --- not protocol reasoning. On the academic side, a January 2025 paper (arXiv 2501.18158) applied general-purpose LLMs to Bitcoin transaction graphs for cybercrime detection, with promising results, but no model was trained specifically on the protocol.

The gap is real. This post is an attempt to sketch what a Bitcoin-native LLM would look like, what dataset it would require, and what the community could do about it --- without anyone needing to commit to building the whole thing alone.

---

## What "thinking in Bitcoin" actually means

A Bitcoin-native LLM is not a price predictor or a chatbot that knows who Satoshi is. The target capabilities are:

**Script reasoning.** Given a raw scriptPubKey or witness script, the model should identify the spending pattern (P2WPKH, P2WSH multisig, HTLC, Taproot key path vs script path, covenant construction...), explain the conditions under which it can be spent, flag unusual or risky constructions, and suggest alternatives. This requires the model to have internalized the Bitcoin Script execution model, all relevant opcodes, and the BIP stack (BIP 141, 143, 340, 341, 342, 65, 68, 112...).

**UTXO graph traversal.** Given a set of addresses or txids, the model should be able to reason about transaction ancestry, identify common-input-ownership clusters, detect peeling chains, coinjoin patterns, or change outputs --- and articulate *why* it drew those conclusions. Not as a black-box classifier, but as an interpretable reasoning chain.

**Script recommendation.** Given a natural-language custody or payment requirement ("2-of-3 multisig with a 6-month recovery path for one key", "HTLC for a Lightning channel", "vault with a 24h delay and a cold emergency key"), the model should produce a correct, minimal script construction and explain the tradeoffs.

**Protocol Q&A.** Deep questions about consensus rules, sighash types, descriptor syntax, miniscript semantics, PSBT fields --- answered accurately, with references to the relevant BIPs.

---

## What the dataset would look like

The raw material is almost entirely already public:

* **BIPs** (full text, rationale sections, test vectors) --- \~200 documents

* **Bitcoin mailing list and Delving threads** --- years of high-signal technical discussion

* **Bitcoin Core source annotations and functional tests** --- ground truth for consensus behavior

* **Annotated real-world scripts** --- parsed scriptPubKey/witness from mainnet, labeled by type (P2WPKH, P2WSH 2-of-3, Taproot, HTLC, etc.). A sample of a few hundred thousand suffices.

* **Miniscript / descriptor corpus** --- the policy-to-script compilation logic is well-documented and could be turned into thousands of (policy, script, explanation) triples automatically.

* **Instruction pairs** --- the missing piece. Someone needs to produce (question, answer) pairs that exercise the capabilities above. This is the labor-intensive part, but it is also partially automatable: a capable general-purpose LLM can generate draft pairs from the raw corpus, which humans then verify.

A rough estimate: \~50k high-quality instruction pairs covering the four capability areas above would be sufficient for a LoRA fine-tune of a 7B-class model that meaningfully outperforms general-purpose models on Bitcoin-specific tasks.

---

## Architecture sketch

The cleanest approach separates two concerns:

1. **A fine-tuned base model** (Llama 3 8B or Mistral 7B + LoRA) trained on the static corpus above. This handles script reasoning, protocol Q&A, and recommendation tasks --- everything that doesn't require live data.

2. **Tool-calling layer** for live queries: a Bitcoin Core RPC bridge, an Electrs interface for scripthash lookups, and optionally a mempool.space API wrapper. The fine-tuned model decides when and how to use these tools; it never holds keys or signs.

The two concerns are independent. The static fine-tune is useful even without the tool layer (for offline script analysis, PSBT review, etc.). The tool layer is useful even with a general-purpose LLM today.

---

## What I'm *not* proposing

This is not a proposal to build a chain analysis surveillance tool. The UTXO graph reasoning capability described above is the same reasoning a wallet developer or a protocol researcher applies when auditing a transaction history --- the goal is interpretability and developer tooling, not deanonymization infrastructure.

This is also not a proposal to give an LLM any signing authority or key material. The model is a reasoning layer, not an execution layer.

---

## Open questions for the community

A few things I genuinely don't know and would value input on:

* Is there prior work on a labeled Bitcoin script dataset I'm not aware of? The Elliptic dataset covers address labels for ML, but not script semantics.

* What would a minimal but meaningful benchmark look like? Spirit of Satoshi built one for culture/economics; a protocol-reasoning benchmark would need to cover script validity, spending condition analysis, and descriptor parsing at minimum.

* Is a LoRA fine-tune on a 7B model actually sufficient, or does Bitcoin Script reasoning require a larger base? The syntax is regular enough that it feels tractable, but the interaction between opcodes, sighash types, and segwit versions has a lot of edge cases.

* Who in this community has touched this problem space? Happy to coordinate on dataset construction --- the bottleneck is annotation, not raw data.

The data is there. The tooling for fine-tuning is accessible. The missing piece is a focused, collaborative effort to assemble the right corpus and define what "correct" looks like for a Bitcoin-reasoning model. This post is an invitation to have that conversation.

-------------------------

sudocarlos | 2026-06-03 00:05:20 UTC | #2

[quote="Tsua00021, post:1, topic:2550"]
What would a minimal but meaningful benchmark look like?

[/quote]

[quote="Tsua00021, post:1, topic:2550"]
Someone needs to produce (question, answer) pairs that exercise the capabilities above. This is the labor-intensive part, but it is also partially automatable: a capable general-purpose LLM can generate draft pairs from the raw corpus, which humans then verify.

[/quote]

Dont overlook stack exchange. Bitcoinops.org uses it as a source for their newsletters. All of their sources and their site, especially transcripts from the optech recap, would be great for training a model

-------------------------

Tsua00021 | 2026-06-03 01:39:30 UTC | #3

Definitely! All technicals aggregators like https://bitcoin.stackexchange.com/, https://bitcoinops.org, or bitcointalk if they want to participate are potential high level sources for training.

-------------------------

l0rinc | 2026-06-07 21:02:13 UTC | #4

I think this came up a few times before, would be really cool to see someone champion it.

Besides the usual sources, I would also include Bitcoin Core PR review discussions, CoreDev transcripts, OpTech transcripts, Bitcoin Stack Exchange, technical books, Learn Me A Bitcoin, and similar.

There often is no single final answer, just different experts arguing for different tradeoffs, so keeping attribution would matter a lot.

-------------------------

Tsua00021 | 2026-06-08 10:03:14 UTC | #5

Do you have links or discussions about this, previously made by Bitcoin devs? 

Yes stackexchange was mentioned. Learn me a Bitcoin is definitely a good website to keep in mind for us to learn but most of common LLMs nowadays know what is into it. Maybe extracting as small papers and give them as RAG ressources could be interesting when you need advice. 

What do you mean by keeping attribution? Attribution of sources or LLM action or do you have something else in mind?

-------------------------

l0rinc | 2026-06-08 11:41:38 UTC | #6

> What do you mean by keeping attribution?

I mostly meant attribution of arguments, not just sources. There often is not a central "truth" here, but different experts arguing for different tradeoffs. For example, in the AssumeUTXO discussion, one side argues that the hash is in the source code and therefore part of the same trust users already place in the software, while the other side argues that it is a separate trust-me-bro shortcut that temporarily accepts unvalidated state: https://github.com/bitcoin/bitcoin/pull/35054#issuecomment-4501781606

So ideally a Bitcoin-native LLM would not just try to produce *the* answer, but preserve which expert argued what, in what context, and what assumptions or compromises each side is making.

-------------------------

AdamISZ | 2026-06-10 10:51:15 UTC | #7

Another useful source is the archives that exist for the old bitcoin-* IRC logs. A lot of discussion was there back in the day.

There's gnusha.org and there's a few others, but with a caveat:

 Here's Poelstra's archive of *-wizards:

https://download.wpsoftware.net/bitcoin/wizards/

this is AJ Towns' for #bitcoin-core-dev going back to 2015: 

https://www.erisian.com.au/bitcoin-core-dev/

The caveat is, I don't think any of these really go back much beyond 2014-15. However, this one is Kanzure's archive tarball (for #bitcoin-dev, #bitcoin-core-dev (which is later anyway) and #bitcoin-otc (for some reason) specifically):

https://diyhpl.us/~bryan/irc/bitcoin/bitcoin-logs.2010-2018.tar.bz2

... which *does* go back to 2010. It might be the only source for that (?).

It would indeed be nice to have something you could query which knew *every* one of the historical conversations on technical topics. Not easy, but clearly LLM tech can partially help.

-------------------------

0xB10C | 2026-06-10 20:12:08 UTC | #8

There's also a GitHub issues and PR dump for a couple repositories that might be interesting to use:

- https://github.com/bitcoin-data/github-metadata-backup-bitcoin-bitcoin
- https://github.com/bitcoin-data/github-metadata-backup-bitcoin-core-secp256k1
- https://github.com/bitcoin-data/github-metadata-backup-bitcoin-bips

You can use https://github.com/0xb10c/github-metadata-mirror to generate input files as markdown, similar to e.g. https://mirror.b10c.me/bitcoin-bitcoin/35506/index.md.

-------------------------

Tsua00021 | 2026-06-15 17:44:15 UTC | #9

Hi @0xB10C 
Your post sounds pretty self-promoting. 
Do you want to add something precisely on the discussion?

-------------------------

0xB10C | 2026-06-10 22:18:39 UTC | #10

[quote="Tsua00021, post:9, topic:2550, full:true"]
Hi @0xB10C Your post sounds pretty self-promoting. Do you want to add something precisely on the discussion?
[/quote]

I'm not sure I understand your problem with my comment.

[quote="Tsua00021, post:1, topic:2550"]
* **Bitcoin mailing list and Delving threads** — years of high-signal technical discussion
* **Bitcoin Core source annotations and functional tests** — ground truth for consensus behavior
[/quote]


your are looking for datasets on technical Bitcoin discussion, I link to available GitHub issue comment and PR review datasets containing these kinds of discussions. While I happen to provide these to the community for free (which seems to be well received e.g. https://delvingbitcoin.org/t/gitlab-backups-for-bitcoin-core-repository/624/10?u=0xb10c) and paying for the infrastructure to run them out of my pocket, I don't see the problem with suggesting to use these GitHub comments and reviews as a dataset for LLM training. But feel free to ignore and don't use them!

-------------------------

Tsua00021 | 2026-06-11 13:13:35 UTC | #11

Catching up on the last few replies, there's more here than I initially registered, so let me tie the threads together.

@l0rinc, your point about attribution of *arguments* rather than just sources is, I think, the most important design constraint raised so far, and it directly reshapes the benchmark question from my original post. For contested topics (AssumeUTXO trust model, mempool policy, covenant proposals), a reference-answer QA benchmark is the wrong evaluation shape entirely. A meaningful benchmark would need two tiers: an objective tier (script validity, spending conditions, descriptor parsing -> mechanically verifiable) and a contested tier, where the question is whether the model reproduces the *structure* of a disagreement — who argued what, under which assumptions — rather than collapsing it into a single confident answer. That second tier is harder to score, but it's also what separates a protocol-reasoning model from a confidently hallucinating one. That two-tier structure seems like the right frame to develop.

@AdamISZ, thanks. The pre-2015 IRC logs are exactly the kind of source nothing else covers, especially Kanzure's 2010-2018 tarball. The signal-to-noise ratio is lower than mailing list posts, but #bitcoin-wizards in particular contains design reasoning that predates its formalization in BIPs, which makes it valuable for "why" questions where the BIP only records the "what". Probably a corpus to filter aggressively rather than ingest wholesale.

@0xB10C, my earlier reply was unnecessarily curt. Your pointer does answer my open question on existing datasets, and combined with l0rinc's earlier mention of PR review discussions, the bitcoin/bitcoin dump may be the highest-signal source on the list: it's the only corpus where technical claims are systematically adversarially reviewed and resolved against actual consensus code. One question to make it actionable: does github-metadata-backup preserve the anchoring of review comments to their diff hunks, or are they exported as flat threads? That determines whether the dump can yield (code, critique, resolution) triples directly, or needs re-alignment against the git history first.

Updated source list so far: BIPs, ML/Delving archives, Core source + functional tests, GitHub issue/PR dumps (bitcoin, secp256k1, bips), Stack Exchange, OpTech transcripts, CoreDev transcripts, IRC archives (2010+), miniscript/descriptor generation. The raw-data question is essentially solved; the open problems are annotation and benchmark design.

-------------------------

Tsua00021 | 2026-06-11 14:27:09 UTC | #12

Following up on my own question to @0xB10C: I inspected the dump directly, and the answer is better than I hoped.

The backup preserves full anchoring. Each review comment in `pulls/{n}.json` carries `diff_hunk` (the exact code fragment under review), `path`, `line`, `commit_id`/`original_commit_id` (so the anchoring survives rebases and force-pushes), `in_reply_to_id` for reply threading, and `pull_request_review_id` for grouping by review session. PR-level `events` (commits, force-pushes, review submissions) are in the same file.

I sampled PRs across the full history — 2014 (#5159), 2015 (#6312), 2016 (#8149, SegWit), 2018 (#15006), 2020 (#19988, ~540 review comments), 2023 (#28122, BIP352) — and every single review comment carries its diff hunk. The format is uniform across twelve years.

Concretely, this means (code, critique, resolution) triples can be extracted directly from the JSON: `diff_hunk` gives the code, `body` the critique, and the resolution reconstructs from the reply thread plus subsequent commits in `events`. No re-alignment against git history needed. As a bonus, #35506 — the AssumeUTXO PR from @l0rinc's attribution example — is in there with the same structure, so the contested-topics angle is covered by the same corpus.

This moves the bitcoin/bitcoin dump from "interesting source" to primary corpus candidate in my view. Thanks again for the pointer.

-------------------------

