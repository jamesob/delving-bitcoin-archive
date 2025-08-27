# The state of bitcoinfuzz

bruno | 2025-08-27 14:04:57 UTC | #1

# The state of Bitcoinfuzz

**bitcoinfuzz** is a project that does differential fuzzing of Bitcoin protocol implementations and libraries. I originally created it as an experiment, and the first version was quite rough with a poor design.
As the project started gaining attention, I refactored it to follow a modular approach similar to cryptofuzz. It means that you can choose which projects you want to fuzz, and you can build the projects (we call modules) individually.

The first targets that I wrote were descriptor and miniscript parsers, where we found many bugs. The first one we reported was in sipa’s miniscript implementation, which incorrectly considered `pk()()` as a valid policy because it identifies `)(` as the name. Currently, we have many other targets like: script evaluation, descriptor parse, miniscript parse, addrv2, psbt, address parse, and others. We plan to expand and work on additional targets in the near future.

So far, we discovered and reported over 35 bugs in projects such as btcd, rust-bitcoin, rust-miniscript, Embit, Bitcoin Core, Core Lightning, LND, etc. Examples include: many implementations using the incorrect type for the key’s type value for PSBTs, the CVE-2024-44073 of rust-miniscript, panic on btcd’s PSBT parser ( https://github.com/btcsuite/btcd/issues/2351 ), Embit incorrectly pointing a miniscript as valid ( https://github.com/bitcoinfuzz/bitcoinfuzz/issues/113 ), Core Lightning accepting invoices with non-standard witness address fallbacks that other implementations correctly reject ( https://github.com/ElementsProject/lightning/pull/8219 ).

## What are the current projects we support?

We currently integrate and support the following projects:

* Bitcoin Core - C++
* rust-bitcoin - Rust
* rust-miniscript - Rust
* embit - Python
* btcd - Golang
* LDK - Rust
* LND - Golang
* Core Lightning - C
* NLightning - C#
* NBitcoin - C#
* Eclair - Scala
* lightning-kmp - Kotlin

There is a PR (under review) that integrates libbitcoin. We welcome any PR that integrates more projects into bitcoinfuzz. However, we intend to review the support of some implementations since an immature implementation can hinder differential fuzzing with simple issues. For example, Embit’s miniscript/descriptor implementation is still incomplete, with many fragments not supported. We also experimented with Mako support, but decided to remove it because the project is not currently being maintained.

## Lightning Network on Bitcoinfuzz

Differential fuzzing of Lightning implementations is proving to be highly valuable.. To advance this effort Erick Cestari and Morehouse have joined the project. Erick began working on it as a Vinteum fellow, mentored by Bruno and Morehouse, and more recently received a full grant from Vinteum. So far, bitcoinfuzz includes targets for BOLT11 invoice decoding and BOLT12 offer decoding. There are also open pull requests adding lightning-kmp module for invoice deserialization, BOLT12 invoice request decoding, adding Eclair module for invoice deserialization, and more.

One advantage of applying differential fuzzing to Lightning implementations is that the Lightning Network has a well maintained specification. It means that when we find any discrepancy, we can reference the spec to determine the correct behavior. This is not the case for Bitcoin protocol implementations, where we mostly rely on Bitcoin Core as the reference. However, we have seen that differential fuzzing is also a good tool to improve the Lightning spec itself. From our experience, the specification is not always clear and often leaves room for improvement. As an example, one of the bitcoinfuzz trophies is a fix on the bolt12 specification, where a currency UTF-8 test vector lacked the required 3-byte length. As a result, many implementations were treating it as a case of malformed currency length instead of properly validating the UTF-8 encoding.

Also, bitcoinfuzz found bugs in virtually every implementation we support. E.g.

* Core Lightning was accepting invoices with non-standard witness address fallbacks that other implementations correctly reject;
* When deserializing an invoice with a large expiry value, LND produces a negative expiry value due to overflow;
* rust-lightning was not verifying whether the offer_currency field contains valid UTF-8 for invoices.
* Eclair was accepting bolt11 invoices with empty routing hints in the r field.

## Doing differential fuzzing of projects that do not have fuzz testing

Some projects either lack support for fuzzing altogether or do not run their fuzz targets continuously. In these cases, we sometimes find bugs not because of the
“differential” aspect, but simply because the project has not been fuzzed at all. As an example, we found and reported a panic on btcd when parsing a PSBT due to a bug on the `ReadTaprootBip32Derivation` functions. This type of bug could have been easily caught by a basic fuzz target and a few minutes of execution. Similar cases occurred with rust-miniscript and lightning-kmp.

## Nuances can also get in the way

When we find a discrepancy between two implementations, it doesn’t always mean there is a bug in one of them. Sometimes small differences in behavior lead to interesting cases. For example, Bitcoin Core accepted miniscripts and descriptors with integer values containing trailing zeros or a positive sign (+) - e.g. `older(+1)` or `older(000001)`, we believe that no one uses it in practice, however, since rust-miniscript rejects these cases, it would cause a discrepancy between them. That said, Bitcoin Core addressed it in https://github.com/bitcoin/bitcoin/pull/30577 by using ToIntegral instead of ParseInt64.

## OSS-Fuzz

We recently opened a PR to integrate bitcoinfuzz into OSS-Fuzz, Google’s continuous fuzzing infrastructure for open‑source software. We believe that it would benefit our project a lot since we currently have a limited fuzzing infrastructure.

## Corpora

We created a repository where we share some corpora for our fuzzing targets, similar to Bitcoin Core’s qa-assets. However, we might keep two folders per target. One folder with all the inputs (maximizing coverage) and another excluding inputs that we know would trigger crashes - e.g. when we’re waiting for an implementation to fix a reported bug. The goal of the second folder is to make it possible to run bitcoinfuzz in CI/CD pipelines.

## Future work

We have several directions we’re eager to explore. In the short term, our focus is on expanding bitcoinfuzz to be fully agnostic and compatible with multiple fuzzing engines beyond libFuzzer. We also plan to introduce new fuzzing targets—such as BOLT8, BOLT7, BOLT4 and ElligatorSwift/V2 Transport — and to strengthen our build system, with a particular emphasis on resolving linking issues.

-------------------------

