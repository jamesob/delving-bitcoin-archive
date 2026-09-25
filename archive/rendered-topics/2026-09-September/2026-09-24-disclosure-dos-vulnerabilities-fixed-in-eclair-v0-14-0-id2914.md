# Disclosure: DoS vulnerabilities fixed in Eclair v0.14.0

morehouse | 2026-09-24 22:47:22 UTC | #1

Two denial-of-service vulnerabilities were fixed in [Eclair v0.14.0](https://github.com/ACINQ/eclair/blob/972dfe995de0a24e8b496c6cf7144e7db377d51d/docs/release-notes/eclair-v0.14.0.md).  Users should upgrade to v0.14.0 or later to protect their nodes.

## [LNF-2026-0001](https://lnfuzz.org/advisories/eclair-feature-parsing-dos/): Feature bit parsing DoS

Eclair v0.13.1 and earlier parsed feature vectors one bit at a time, allocating several heap objects for every bit.
A single maximum-length `init` message caused about 300 MB of heap churn and occupied a parsing thread for up to 300 ms.
By flooding the victim with such messages, an attacker could disconnect all the victim's peers within a minute and run it out of memory within five.

[smite](https://github.com/lnfuzz/smite) found this vulnerability with its most primitive scenario, which sends raw bytes as a single message and then checks that the target still answers a ping promptly.
One input, which happened to be an `init` with a large feature vector, made Eclair take too long to respond.

More details in the [lnfuzz advisory](https://lnfuzz.org/advisories/eclair-feature-parsing-dos/).

## [LNF-2026-0002](https://lnfuzz.org/advisories/eclair-zlib-decompression-dos/): zlib decompression DoS

Eclair v0.13.1 and earlier still accepted zlib-encoded channel queries, four years after BOLT 7 retired the encoding.
Decompression had no output limit, so a 64 KB `query_short_channel_ids` message inflated to 64 MB and decoded into about 17 million heap objects.
A flood of these messages took the victim offline within seconds and ran it out of memory within minutes.

smite did not find this one directly.
After LNF-2026-0001, I ran LLM-assisted variant analysis over the Eclair codebase, looking for other places where a peer could impose far more work on the node than it spends itself.
The analysis flagged the zlib codec, and experiments confirmed the DoS.

More details in the [lnfuzz advisory](https://lnfuzz.org/advisories/eclair-zlib-decompression-dos/).

-------------------------

