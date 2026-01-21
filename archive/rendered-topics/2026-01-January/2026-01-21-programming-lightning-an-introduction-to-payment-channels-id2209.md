# Programming Lightning: An Introduction to Payment Channels

beige-coffee | 2026-01-21 20:43:25 UTC | #1

# Programming Lightning: An Introduction to Payment Channels

Hi All, I’m excited to share Programming Lightning - an educational resource I’ve been working on as part of a Spiral (and, recently, HRF) grant.

Programming Lightning, inspired by [Programming Bitcoin](https://github.com/jimmysong/programmingbitcoin), teaches developers and technically-inclined folks how Lightning works by coding important pieces of the protocol from scratch.

Today, I’m sharing the first module of the larger Programming Lightning course, **Intro to Payment Channels**, which guides students through building a simple off-chain wallet and Lightning payment channel from scratch. By the end of the course, students’ implementations will pass some of the major BOLT 3 test vectors, meaning they will have successfully implemented functionality such as key derivation, obscured commitment numbers, and HTLC second-stage transactions (success and timeout).

While most exercises can be completed by forking the GitHub repo and running the tests locally, the course is optimized to be completed on Replit. This is because each Repl comes with Bitcoin Core running in the background, allowing students to easily generate Lightning transactions as they complete the exercises and simulate operations such as broadcasting transactions, opening channels, sending payments, and decoding raw transactions.

Replit: https://replit.com/@austin-f/Programming-Lightning-Intro-to-Payment-Channels?v=1

GitHub: https://github.com/Beige-Coffee/programming-lightning-payment-channels

## Goals

Programming Lightning has two main goals:

1. Provide hands-on, in-depth protocol-level learning to help developers understand how Lightning works - preparing them to contribute to Lightning implementations, protocol design, or application development.

2. Provide an accessible yet highly-technical resource for anyone wanting to deeply understand Lightning.

## Up Next

With my Spiral & HRF grant continuing through 2026, I plan to build new chapters focusing on Lightning payments (authentication, onion routing, gossip), invoices (BOLT 11, BOLT 12), and newer protocol advancements like taproot channels, splicing, and more.

Feedback and corrections are welcome! Please feel free to open an issue, submit a pull request, or email me directly at [hello@programminglightning.com](mailto:hello@programminglightning.com). I’d love to hear your thoughts on how to improve the content!

-------------------------

