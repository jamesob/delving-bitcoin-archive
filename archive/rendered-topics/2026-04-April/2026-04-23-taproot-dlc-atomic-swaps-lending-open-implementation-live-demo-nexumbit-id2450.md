# Taproot DLC Atomic Swaps & Lending: Open Implementation + Live Demo (NexumBit)

TheLonelyBit | 2026-04-23 22:56:48 UTC | #1

Hey everyone,

I’ve built and open-sourced a practical, non-custodial implementation of cross-chain atomic swaps and basic lending using Taproot + Discreet Log Contracts (DLCs). Sharing it here for feedback and discussion.

What it does

* Trustless atomic swaps and conditional lending between Bitcoin (Taproot) and other Taproot-compatible chains (Fractal Bitcoin, potentially Litecoin, etc.).
* Fully non-custodial: users keep control of keys at all times.
* Uses adaptor signatures + oracle attestations for settlement.
* Built on Tadge Dryja’s DLC spec with Taproot for better privacy and efficiency.


Live early demo: [https://bridge.thelonelybit.org](https://bridge.thelonelybit.org)
Full open-source repo (builders, signers, DLC contract logic, PSBT workflows, lending primitives):
[https://github.com/The-Lonely-Bit/Taproot-based_Discreet_Log_Contracts](https://github.com/The-Lonely-Bit/Taproot-based_Discreet_Log_Contracts)It’s still in controlled beta (not fully public mainnet launch yet), but the core contracts are functional and the code is public for review.

Purpose

The goal is to provide a Bitcoin-native, trustless way for users to move value and access liquidity across Taproot-compatible chains without custodians, wrapped tokens, or centralized bridges, or new metaprotocols. This is especially aimed at giving individuals in regions with capital controls and financial surveillance a censorship-resistant option using pure cryptographic guarantees.

I’m looking for technical feedback around :

* Contract design and Taproot script usage
* PSBT flows and signing logic
* DLC builder primitives and reusability
* Any improvements to efficiency or edge cases

This is meant to be a practical contribution to DLC tooling in the Bitcoin ecosystem, Any eyes on the GitHub and thoughts on the implementation would be greatly appreciated.

Thanks !

-------------------------

