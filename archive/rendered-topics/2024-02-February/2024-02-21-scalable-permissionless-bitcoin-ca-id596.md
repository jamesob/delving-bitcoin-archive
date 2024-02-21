# Scalable & Permissionless Bitcoin CA

buffrr | 2024-02-21 19:41:55 UTC | #1

I'd like to discuss a project we're developing called Spaces, for creating a scalable & permissionless ~250KB Bitcoin Certificate Authority. We're working on building a zk light client for the protocol using RISC0 zkVM[0] and the protocol uses client-side validation.

Spaces introduces a concept of "subspaces" which are off-chain entities that have great autonomy. It is not exactly like zk-rollups as the data itself is off-chain - although it may not exactly fit into "validium" either as data availability itself is not a major issue. Users hold their own inclusion proofs. The commitments posted on-chain have a lifetime during which they're able to transact on-chain and off-chain. 


We're at an early stage and welcome your feedback. For more technical details or if you'd like to join us:

https://spacesprotocol.org


[0] https://dev.risczero.com/proof-system-in-detail.pdf

-------------------------

