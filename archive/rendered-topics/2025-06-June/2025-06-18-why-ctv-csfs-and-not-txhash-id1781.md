# Why CTV+CSFS and not TXHASH

arshbot | 2025-06-18 18:11:22 UTC | #1

I'll keep this brief.

As a community, it's important to align clearly on the purpose of the current covenant proposal. Informally, there appears to be consensus around implementing CTV+CSFS.

However, is there potential alignment around TXHASH? Its functionality is broader. Unlike CTV+CSFS, TXHASH enables commitment to specific inputs and outputs, providing more granular control over transaction structure. This granularity helps avoid potential pitfalls with CTV+CSFS, such as vault setups that necessitate all-or-nothing fund movements (due to amounts being fixed and pre-templated).

If there is value in narrowly scoped transaction commitments (like those provided by CTV and CSFS), it logically follows there could also be value in broader transaction commitments.

Indeed, broader commitments have clear use cases. Projects benefiting from this could include:

* LN-Symmetry
* Ark
* PTLCs
* DLCs

There are definite, compelling applications for TXHASH. For instance, Magnolia could significantly leverage TXHASH to offer transaction guarantees that currently require expensive legal infrastructure, such as Trust charters.

Nonetheless, before pursuing TXHASH, it seems prudent to first validate that there is sufficient demand for basic wholesale transaction commitments (CTV+CSFS). Once that's established, we can explore the need for more granular transaction commitment features.

An important consideration remains: Is there perhaps greater demand for these more granular transaction commitments (like those enabled by TXHASH) than for basic template validation alone? If that's the case, CTV+CSFS itself should experience robust adoption and clear demand.

-------------------------

ProofOfKeags | 2025-06-24 19:12:57 UTC | #2

TXHASH does not obviate the need for CSFS. It is a full-replacement for CTV, though.

-------------------------

