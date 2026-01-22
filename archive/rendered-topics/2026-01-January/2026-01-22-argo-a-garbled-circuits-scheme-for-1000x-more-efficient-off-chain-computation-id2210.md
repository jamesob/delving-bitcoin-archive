# Argo: a garbled-circuits scheme for 1000x more efficient off-chain computation

RobinLinus | 2026-01-22 02:12:31 UTC | #1

### **Argo MAC: Garbling with Elliptic Curve MACs**

##### **Abstract**

Off-chain cryptography enables more expressive smart contracts for Bitcoin. Recent work, including BitVM, use SNARKs to prove arbitrary computation, and garbled circuits to verifiably move proof verification off-chain. We define a new garbling primitive, Argo MAC, that enables over 1000x more efficient garbled SNARK verifiers. Argo MAC efficiently translates from an encoding of the bit decomposition of a curve point to a homomorphic MAC of that point. These homomorphic MACs enable much more efficient garbling. In subsequent work, we will describe how to use Argo MAC to construct garbled SNARK verifiers for pairing-based SNARKs.


https://eprint.iacr.org/2026/049

-------------------------

