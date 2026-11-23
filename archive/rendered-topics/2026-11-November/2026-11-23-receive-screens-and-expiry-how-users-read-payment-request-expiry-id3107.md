# Receive screens and 'expiry': how users read payment-request expiry

Anzus_GemWallet | 2026-11-23 11:49:37 UTC | #1

A small implementation question from the support side, prompted by a recent Bitcointalk thread on receive-tab UX.

Some wallets attach an expiry to payment requests (amount, label, sometimes a time and expiry in the URI). From the user side, we regularly see that expiry read as the address itself expiring. That leads to two outcomes we'd rather avoid: users panicking that funds sent to an "expired" address are lost, or users thinking an address is dead and abandoning it while it's still being paid to.

The wording seems to matter more than the mechanism. Tooltips rarely help, because the users who misread it are the ones who copy the address and leave.

Questions for people who've built or maintained receive flows:

- Is there a reason to show an expiry for plain on-chain requests at all, beyond Lightning-style invoices?
- If a wallet does show one, has anyone found wording that makes it clear the request expired, not the address?
- For wallets that rotate addresses on every request, how do you signal that earlier addresses still work?

Disclosure: I work in support at Gem Wallet, not as a developer, so I'm describing what users report rather than proposing a design.

-------------------------

