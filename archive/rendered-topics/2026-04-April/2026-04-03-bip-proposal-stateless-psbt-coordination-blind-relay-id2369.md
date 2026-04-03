# BIP Proposal: Stateless PSBT Coordination (Blind Relay)

scarlin90 | 2026-04-03 11:39:13 UTC | #1

Hi everyone,

I’m seeking conceptual and technical feedback on a draft BIP for Stateless PSBT Coordination, designed to solve the communication gap in multisig setups without relying on trusted, stateful servers.

Currently, coordinating a multisig transaction across hardware and software wallets requires either manual file passing (USB/SD) or relying on a centralized coordinator server that logs IP addresses, xpubs, and transaction metadata.

The Proposed Standard: I’ve drafted a BIP [(bip-stateless-psbt-coordination)](https://github.com/scarlin90/bip-stateless-psbt-coordination) that proposes a “Blind Relay” model. It leverages an ephemeral, end-to-end encrypted WebSocket architecture.

Zero-Knowledge: The relay server never sees the PSBT, the xpubs, or the signatures.

Stateless: The room and all associated data cease to exist the moment the room is closed or after 24hours.

Split-Key OpSec: The room URL and decryption key (#key fragment) are transported out-of-band by the users.

Implementation (Signing Room): To prove the viability of this BIP, I’ve built an open-source reference implementation called [Signing Room](https://signingroom.io/). I just released [v1.8.0](https://github.com/scarlin90/signingroom/releases/tag/v1.8.0), which enforces human-layer OpSec by actively separating the transport of the room link and the decryption key in the UI.

Feedback Request: I would love the community’s eyes on:

The BIP draft itself: https://github.com/scarlin90/bip-stateless-psbt-coordination

Thoughts on the UX/OpSec tradeoff of the Split-Key UI approach in v1.8.0.

Looking forward to your thoughts and any features you think might be useful in this reference implementation.

All the best,

Sean Carlin

-------------------------

