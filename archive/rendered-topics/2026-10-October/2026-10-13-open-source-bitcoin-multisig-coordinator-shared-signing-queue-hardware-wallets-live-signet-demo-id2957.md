# Open-source Bitcoin multisig coordinator: shared signing queue, hardware wallets, live signet demo

tom2o17 | 2026-10-13 13:25:48 UTC | #1

I've open-sourced **btc-safe**, a self-hosted, watch-only Bitcoin multisig coordinator. One owner proposes a transaction. Every co-signer sees it in a shared queue and signs from their own device. It broadcasts once M of N have signed.

• **Watch-only:** the server stores xpubs and descriptors only. It never holds keys and never signs.

• **Independent checks:** the browser re-derives change and receive addresses from the xpubs and cross-checks every PSBT against the server.

• **Signers:** Ledger (WebHID), Trezor, Sparrow and Caravan import, and PSBT files.

• **Accounts:** TOTP 2FA and per-wallet roles (admin, proposer, viewer). Reorg-aware confirmations.

• **Stack:** Rust (axum, bdk_wallet, Postgres) and Next.js. MIT licensed.

**Try it (signet, no real funds):** https://web-production-56752.up.railway.app. Sign up with any email, create a 2-of-3, and fund it from a signet faucet.

**Code:** https://github.com/tom2o17/btc-safe


![telegram-cloud-photo-size-1-5059908105818279286-y|689x431](upload://ugRslGOe8LYZHpxUXhInLvwAzQD.jpeg)

-------------------------

