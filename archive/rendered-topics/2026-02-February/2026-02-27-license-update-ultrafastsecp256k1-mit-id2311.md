# License Update: UltrafastSecp256k1 → MIT

shrec | 2026-02-27 15:50:14 UTC | #1

After valuable feedback from the community, I’ve decided to relicense **UltrafastSecp256k1** under the MIT license.

The project was previously released under AGPL. While that license protects open-source contributions, it also creates adoption friction in the Bitcoin ecosystem, where MIT is the dominant standard (Bitcoin Core, libsecp256k1, Lightning implementations, etc.).

My goal is not to restrict usage, but to:

• enable broad experimentation
• allow integration into Core, Lightning, nodes, and research projects
• reduce legal uncertainty
• align with ecosystem norms

UltrafastSecp256k1 is built as:

* zero-dependency

* portable across CPU, GPU, MCU, SBC

* unified API across platforms

* performance-oriented but correctness-first

The mission remains the same:
to provide a high-performance secp256k1 engine usable across all platforms without friction.

All future releases will be MIT licensed.

I appreciate the candid feedback that led to this decision.
https://github.com/shrec/UltrafastSecp256k1

-------------------------

AntoineP | 2026-02-27 16:02:50 UTC | #2

Hi, in the past 3 days you've created 4 threads about your library. I don't think it's necessary to create a new topic for every update.

-------------------------

