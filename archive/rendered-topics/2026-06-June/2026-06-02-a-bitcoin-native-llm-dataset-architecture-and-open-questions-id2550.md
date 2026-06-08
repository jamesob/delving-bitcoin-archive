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

