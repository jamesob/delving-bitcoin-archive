# Garbled circuits and BitVM3

RobinLinus | 2025-06-15 14:04:55 UTC | #1

## Garbled Circuits and BitVM3

The BitVM Alliance is pivoting to garbled circuits, as they make SNARK verification on Bitcoin over 1000x more efficient than BitVM2, enabling drastically cheaper BTC bridges. A vast design space is now being explored in parallel by multiple teams.

One such approach is [BitVM3](https://bitvm.org/bitvm3.pdf), which strikes a practical balance between implementation complexity, communication overhead, and proving cost.

The design is inspired by Jeremy Rubin’s [Delbrag](https://rubin.io/bitcoin/2025/04/04/delbrag) scheme. The key improvement is that BitVM3 circuits can be verified in plaintext and then reblinded — addressing Delbrag’s main drawback: the need for either expensive circuit correctness proofs or 200kB disprove scripts.

-------------------------

