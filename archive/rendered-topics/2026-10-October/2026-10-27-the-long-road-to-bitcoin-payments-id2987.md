# The Long Road to Bitcoin Payments

ConorOkus | 2026-10-27 15:18:14 UTC | #1

**Why bitcoin payments haven’t taken off, and a call for moonshot ideas that could change that**

“If \[bitcoin\] ends up just being digital gold, I think it’s failed,” Jack Dorsey [said in January 2025](https://www.youtube.com/watch?v=KyFkv5DGeUA&t=1250s). “If it ends up being a currency that people can use daily, then it’s succeeded.” Not everyone agrees. Many in bitcoin argue it must first succeed as a store of value, and that demand for payments will follow. So we start with a clear question: does the current path lead to irrelevance, or is it the expected grind of hardening bitcoin for the day it’s needed for payments? This report asks what is holding bitcoin payments back, and what moonshot ideas could speed up adoption.

Bitcoin payments work better than ever. Turning that progress into everyday use remains the challenge, and the time to move is now, as stablecoins are ready to gain rapid adoption.

**Where things stand**.Over the past seven years, bitcoin spending technology has improved dramatically. Lightning infrastructure runs in production, and self-custodial wallets now hide most of their complexity. Seven years ago, building a bitcoin payments app took more than a year and several developers. Today, anyone can build a working first version from a single prompt to an AI coding tool. Custodial bitcoin is now no harder to integrate than custodial stablecoins, though self-custodial bitcoin is still harder to run at scale. More than a million US merchants can accept bitcoin through Square. Yet few people pay with bitcoin in everyday life, a point most in the industry accept. The gap is mostly demand, not technology.

**Why now.** The wider financial system is being updated, raising an open question: will the rails chosen during this shift be hard to change later, or will payments follow what people want to pay and be paid in, with software making it easy to add any payment rail? As large payment providers update their rails, they are choosing stablecoins, not bitcoin. Stripe, Shopify and Coinbase are building stablecoins in, and major consumer products appear likely to follow. Stripe, Visa, American Express and Google have also joined a consortium launching a new stablecoin, OUSD, and the first protocol implementations for AI agent payments settle almost entirely in stablecoins. Developers build on the rails their tools support, and wallets ship with the networks their partners already use. AI agents are the clearest case. They need small, programmable payments, which Lightning suits, yet the first agent payment tools are being built around stablecoins and cards. If early choices do stick, bitcoin’s absence now could be hard to reverse. If they don’t, the question becomes demand alone.

**What’s blocking it?**

* **Demand**: neither side is pulling for it. Holders would rather save than spend, and bitcoin’s culture celebrates holding. Merchants prefer to receive and hold dollars, so few accept bitcoin, and new payment platforms leave it out.
* **Volatility and competition:** price swings make bitcoin hard to price goods in, and dollar stablecoins meet the need for stable payments.
* **Policy:** US tax treatment of every purchase and pending regulation.

**What could change this?** We’re looking for moonshot ideas: changes big enough to shift bitcoin payments, not just improve them. A few possible paths to start the discussion:

* **Shared rails:** let people pay in dollars or bitcoin over the same Lightning network, with a compelling reason to choose it, so bitcoin is one tap away wherever people already pay: in agent payments, merchant checkout and fintech apps.
* **Payments for people who need them most:** self-custodial bitcoin payments that better solve the needs of people facing capital controls or unstable currencies.
* **New reasons to spend:** spending incentives for existing holders.
* **New ways to spend:** product experiences that create entirely new kinds of payments, such as paying creators or AI services per use. The killer app for micropayments probably won’t feel like a traditional payment at all.

We don’t know which, if any, will work. That’s why we want to hear yours.

**What we’re asking.** Our central question: what moonshot ideas would create a believable, viable plan to accelerate bitcoin payments adoption? To make it explicit: what would it take for a major platform such as Meta, Shopify, or OpenAI to accept bitcoin? Builders: propose ideas and test them. Payment providers: share data on how people pay. Funders: back the most promising experiments and help foster demand. This is a living document. Read the [full report on GitHub](https://github.com/spiralxyz/bitcoin-payments), share your ideas or data in the replies below, or join the [GitHub discussion](https://github.com/spiralxyz/bitcoin-payments/discussions/1). We’ll keep updating the report as we learn.

-------------------------

Anzus_GemWallet | 2026-10-28 03:04:46 UTC | #2

From the wallet support side (I work in BD and support at Gem Wallet, not as a developer), one blocker shows up before demand even comes into it: many users don’t know whether they’re about to pay on-chain or over Lightning, and the differences in fees and timing catch them off guard at the moment of sending. When the first payment feels confusing, people tend to go back to just holding.

So next to the moonshots, I’d add a small, unglamorous one: make the first payment boring. That means a clear label for the payment type, fees shown before confirming, and recovery instructions that cover Lightning funds too. None of that creates demand, but it stops early users from giving up.

-------------------------

