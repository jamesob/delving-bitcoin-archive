# Provable Cryptography for Bitcoin: An Introduction (Workbook)

jonasnick | 2025-09-09 13:09:02 UTC | #1

I’m excited to share a comprehensive cryptography workbook that we developed for an interactive workshop on Bitcoin cryptography. This workbook contains definitions, propositions, lemmas, theorems, and exercises, along with a complete “solutionbook” with solutions to the  exercises.

**Download:** https://github.com/cryptography-camp/workbook/releases

This workbook was designed for a multi-day interactive workshop that has already taken place. During the workshop, participants provided significant feedback, resulting in substantial polish and battle-testing of the material from actual use.

## Contents

The workbook covers selected topics in Bitcoin cryptography:

- Introduction (algorithms, negligible functions, games & asymptotic security, hash functions)
- Reductions
- Commitments
- Accumulators
- One-Time Signatures
- Discrete Logarithm
- Random Oracle Model
- Programmable ROM and Forking Lemma
- Signatures
- Signatures with Key Tweaking

The workbook has two primary goals:
1. **To provide sufficient background to understand state-of-the-art papers on cryptographic signatures**, including their security proofs, with a focus on discrete-logarithm-based (multi-party) signatures (like the DahLIAS interactive aggregate signature scheme).
2. **To develop the skills needed to formalize security notions for cryptographic primitives**. This skill is crucial for selecting appropriate primitives when proposing and reviewing cryptographic protocols, and for defining precisely what a protocol aims to achieve.

The workbook provides detailed treatment of how security is formalized and systematically introduces key proof techniques. The goal is to build intuition step-by-step through the exercises, which provide hands-on experience with the theoretical concepts.
One of the highlights is examining how security proofs break after making seemingly minor changes to either the security definition or the cryptographic scheme.
The workbook contains optional exercises that go deeper into the material for those who want to explore topics more thoroughly.

One of the outcomes of the workshop was that participants developed a [preliminary proof for the Schnorr sign-to-contract scheme](https://github.com/cryptography-camp/workbook/tree/s2c), which, to the best of my knowledge, hasn't been proven secure before.

## Prerequisites

The prerequisites are mainly modular arithmetic and basic probability theory, though a basic overview of cryptography is certainly helpful. For preparation, I highly recommend Nadav Kohen's excellent prework material that was created for the workshop: https://github.com/cryptography-camp/prework

## Feedback Welcome

We welcome feedback, corrections, and suggestions for improvement. Feel free to open issues or pull requests on the GitHub repository.

I hope this resource proves useful for anyone looking to deepen their understanding of the cryptographic foundations underlying Bitcoin and related protocols.

-------------------------

marathon-gary | 2026-01-07 15:08:08 UTC | #2

Just noting here another resource I encountered recently along the same topic but not specifically bitcoin focused: https://joyofcryptography.com/

-------------------------

