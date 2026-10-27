# Where should wallets explain the differences between on-chain Bitcoin and Lightning?

Anzus_GemWallet | 2026-09-30 02:48:06 UTC | #1

I work in BD and support at Gem Wallet, rather than as a developer. One recurring usability issue I notice is that beginners often see on-chain Bitcoin and Lightning simply as “BTC,” without understanding that the payment and recovery experience may differ.

This creates several moments where a wallet has to decide how much information to show.

**Receiving**

Should users choose between on-chain and Lightning before seeing payment details, or should the wallet guide them based on what they are trying to do? How much explanation is necessary without overwhelming someone making their first payment?

**Sending**

When a user pastes an address or invoice, the wallet can often identify the payment type automatically. Should the confirmation screen still label it clearly as on-chain or Lightning and explain the expected fee, speed and possible failure conditions?

**Backup and recovery**

“Your seed phrase backs up your wallet” sounds simple, but what it restores can depend on the wallet’s Lightning design. How should a wallet explain this accurately before a device is lost, without presenting users with implementation details they may not understand?

**Payment failures**

Should the difference between the two systems remain mostly hidden until something goes wrong, or does hiding it create an inaccurate expectation that all Bitcoin payments behave the same way?

There seems to be a tradeoff between showing too much information and giving users an overly simplified mental model. I’m interested in how wallet developers approach this boundary.

Are there interface patterns, terminology or user-testing results that have worked particularly well? At what point—onboarding, receiving, confirmation or backup—should these differences be explained?

-------------------------

lusabeko | 2026-10-26 19:29:15 UTC | #2

Treat payment type as a property of the payment, not a choice users must learn upfront. Detect the invoice or address, then label the confirmation screen “Lightning” or “on-chain” with the relevant fee and settlement expectations. Put the recovery caveat beside backup creation: a seed may restore on-chain funds, while Lightning recovery can depend on the wallet’s channel-backup design and service. Repeat it in recovery instructions, not just onboarding.

-------------------------

Anzus_GemWallet | 2026-10-27 02:11:14 UTC | #3

Thanks, this is really helpful. Labelling the payment type on the confirmation screen, instead of expecting people to know it beforehand, matches what we see in support: most confusion happens at the moment of sending, not during setup. The point about repeating the Lightning recovery caveat in the recovery instructions is a good one too, since that’s exactly when people need it. I’ll pass both along to our team.

-------------------------

