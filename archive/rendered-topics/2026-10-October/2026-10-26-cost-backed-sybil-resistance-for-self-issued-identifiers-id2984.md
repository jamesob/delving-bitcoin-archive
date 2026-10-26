# Cost-backed Sybil resistance for self-issued identifiers

groundinglab | 2026-10-26 15:55:30 UTC | #1

Open systems issue identifiers without an associated resource cost. The marginal cost of an additional public key approaches zero, so bulk production incurs almost no incremental cost while the accepting party bears the costs of filtering, storage, and abuse handling.

Volume is limited only by the capacity of the parties that must process the identifiers. In the absence of a cost on the object itself, the cost of watching falls on the acceptor.

Conventional defenses place a gate in front of issuance. These raise the cost of bulk production by requiring something external to the identifier, yet they reintroduce a third party or shared policy into the issuance path. The gate itself can be flooded or gamed, and its cost is frequently borne by the legitimate user as friction rather than by the bulk producer as a durable resource commitment.

An alternative attaches the cost directly to the production of the identifier, without an issuer. The Satoshi Hike 2.3 specification defines a construction in which a producer retains a 256 MiB memory terrain and performs a sequence of memory-dependent legs bound to consecutive Bitcoin headers. The resulting object records the evidence of that process. Bitcoin supplies a public ordering reference; it does not enforce or pay for production.

An independent verifier measures the object against the specification and a portable public index. It checks structural and cryptographic bindings and recomputes a random sample of six legs. A successful measurement confirms consistency with the specified construction. It does not, by itself, establish that an attacker could not have produced the same evidence more cheaply. Whether the construction forces a meaningful marginal physical cost under adversarial conditions remains an open question.

The resource commitment is itself self-issued: memory residency expended and bound to an independent clock. The commitment travels with the identifier.

The specification defines the object and the measurement procedure. Accepting parties apply their own policy on top of the measured commitment. The specification does not prescribe an acceptance policy or a minimum cost in currency units.

**Specification**: https://github.com/groundinglab/pcoc

Critical review is welcome on whether time-memory tradeoffs, precomputation, or reuse across objects can substantially reduce marginal production cost; the contribution and failure modes of the Bitcoin header binding; the actual production-versus-verification asymmetry given sampled recomputation; and closest prior art.

The question is whether this construction can make additional self-issued objects costly enough to be useful under adversarial conditions, and what evidence would demonstrate that.

-------------------------

