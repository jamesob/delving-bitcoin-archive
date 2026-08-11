# Libshrincs: A C implementation with a machine-checked security proof

jonasnick | 2026-08-11 20:31:42 UTC | #1

**Abstract**: @remix7531 and I are publishing [libshrincs](https://github.com/remix7531/libshrincs), a proof-of-concept C implementation of WOTS+C with machine-checked proofs of functional correctness and security.
WOTS+C is the one-time signature scheme used by [SHRINCS](https://delvingbitcoin.org/t/shrincs-324-byte-stateful-post-quantum-signatures-with-static-backups/2158), a post-quantum signature scheme proposed for Bitcoin.
We formalized the security proof in the [Rocq Prover](https://rocq-prover.org/) using [SSProve](https://github.com/SSProve/ssprove), and used the [Verified Software Toolchain](https://vst.cs.princeton.edu/) (VST) to prove that the C implementation behaves according to the Rocq specification used in the security proof.
The proofs were produced primarily by large language models under human guidance.
Because the proofs are machine-checked, human review can focus on whether the definitions capture the intended scheme, security property, and assumptions, and whether the theorem gives a useful bound.
I wrote a tutorial for conducting that review without prior knowledge of Rocq or SSProve.
libshrincs demonstrates that end-to-end formal verification can complement code review and testing of cryptographic C code intended for the Bitcoin protocol.

Library: https://github.com/remix7531/libshrincs
Tutorial for reviewing the security proof (no Rocq or SSProve background assumed): [PDF](https://github.com/remix7531/libshrincs/blob/tutorial/docs/SSPROVE_SECURITY_REVIEW.pdf)[^tutorial-authorship]

[^tutorial-authorship]: LLMs wrote the tutorial; I read it once from beginning to end, correcting and improving it as I went.

## A modern cryptographic library for Bitcoin

This work started with a question: What would a modern post-quantum cryptographic library for Bitcoin Core look like?

Bitcoin Core already depends on a dedicated C library for cryptography (libsecp256k1) and the developers who review Bitcoin's cryptographic code are used to C.
For reviewability and maintainability, libshrincs is therefore handwritten in C (and not generated from another language).

The main goal of libshrincs was to determine whether machine-checked proofs of functional correctness and security could be practical for a cryptographic library of this kind.
Until recently, producing both proofs would have been prohibitively difficult because of the specialized expertise and engineering effort required.
The large language models available in 2026 changed that.

## From C to unforgeability

Formal verification of an implementation often starts with functional correctness: proving that the implementation produces the outputs and state changes prescribed by a specification.
A functional-correctness proof establishes only this relationship between the implementation and the specification.
By itself, functional correctness does not establish that the specification has the desired security properties.
For libshrincs, VST proves that the C implementation satisfies a functional specification written in Rocq.
Using SSProve, we also formalize the pen-and-paper security argument and prove an unforgeability bound for the signature scheme defined by the same specification.
Rocq checks both proofs.

Two earlier projects also connect implementations to machine-checked security proofs: [Formosa ML-KEM](https://github.com/formosa-crypto/formosa-mlkem) for ML-KEM and [*Completing the Chain*](https://eprint.iacr.org/2026/134) for XMSS.
Both implementations are written in Jasmin rather than C.

Together, the VST and SSProve proofs establish the following chain:

```text
C implementation
       |
       | VST
       v
Rocq specification
       |
       | SSProve
       v
Unforgeability bound
```

## Reviewing the security proof

The VST proof contains about 6,200 lines of Rocq and the SSProve proof about 13,400, with another 2,700 lines shared between them.
Reviewers do not need to inspect the proof steps manually: `make prove` checks both proofs with Rocq.
On my laptop, running it with `-j 20` takes about four minutes and uses 14.5 GB of memory.
`make audit` also rejects unfinished proofs in the libshrincs sources and checks that every unproved dependency of the main results is explicitly allowlisted.

After these checks, manual review of the SSProve part can focus on a few hundred lines of Rocq:

- the six hash games and the axioms that bound adversarial advantage in them;
- the security notion, i.e. the unforgeability experiment;
- the theorem statement itself.

The [tutorial](https://github.com/remix7531/libshrincs/blob/tutorial/docs/SSPROVE_SECURITY_REVIEW.pdf) assumes no prior experience with Rocq or SSProve: it reproduces these definitions, teaches the notation needed to read them, and gives a step-by-step review procedure.
For a more basic introduction to security proofs for hash-based signatures, Section 5 of the [*Provable Cryptography for Bitcoin* workbook](https://github.com/cryptography-camp/workbook/releases) works through the security proof of Lamport one-time signatures.

We discovered firsthand how important it is to review the definitions carefully.
Late in the process, while writing the review tutorial, we found that four of the six hash assumptions were trivially broken.
Each could be broken by a two-line adversary with probability 1.
The security theorem effectively says that forging a signature is no easier than breaking at least one of the six assumptions.
The theorem therefore gave no indication of how hard the signature scheme is to break.
We fixed these flaws, but there is no guarantee that we found every mistake in the formalization.

## Current limits of the security theorem

The current theorem does not yet give a full post-quantum security bound.
Obtaining one would require modeling adversaries in the quantum random oracle model (QROM), accounting for the running time of the reductions, and deriving numerical bounds for the six hash assumptions in that model.

The theorem bounds forgery advantage in terms of the advantages against the six hash games, but does not give numerical bounds for those advantages.
Using published QROM bounds would require relating their experiments and resource measures to the formal games.

The attack experiment requires the adversary to choose the message digest before seeing the public key, rather than using standard one-time EUF-CMA.
This matches how WOTS+C is used inside SHRINCS, but it is a weaker security notion.

I know of no fundamental obstacle to addressing these limits, but doing so may require substantial additional work.
The [review tutorial](https://github.com/remix7531/libshrincs/blob/tutorial/docs/SSPROVE_SECURITY_REVIEW.pdf) explains these limitations in more detail.

## Producing the security proof with LLMs

The WOTS+C security proof grew out of a larger SHRINCS proof development.
Each LLM had access to the Rocq repository and its standard library, the SSProve repository, a pen-and-paper security proof of SHRINCS, and the [EasyCrypt security proof of SPHINCS+](https://github.com/MM45/FV-SPHINCSPLUS-EC).
A rough timeline follows:

- **20 May:** Work began with ChatGPT 5.5.
  It spent roughly two weeks in goal mode without making progress on the proof.
  It proposed new assumptions but could not prove them.
- **10 to 12 June:** I used Fable, which had been released on 9 June and was suspended on 12 June.
  It made substantial progress during those two days.
- **12 June:** I switched to Opus 4.8 (ultracode), which made slow progress on the SHRINCS proof.
- **16 June:** I handed the WOTS+C proof to @remix7531, who concurrently developed the C implementation, the VST proofs, and the shared Rocq model, and adapted the security proof to use that model.
- **22 July:** I switched to ChatGPT 5.6 (xhigh), which estimated that about 15% of the proof remained.
- **28 July:** ChatGPT 5.6 completed the first version of the SHRINCS proof.[^shrincs-review]

[^shrincs-review]: The full SHRINCS proof is still under review.

The rate of progress depended heavily on how quickly a model could check a change with Rocq.
In our setup, without the [MCP server](https://github.com/LLM4Rocq/rocq-mcp), a model checked each change by recompiling an entire source file.
One of the larger files took about 85 seconds to compile.
The MCP server instead let a model check individual theorem steps in seconds.

Another bottleneck was memory.
The security proof was developed on a machine with 64 GB of RAM, which often allowed only one agent to run Rocq at a time.
For similar work, I would use a machine with more RAM so that several agents could run Rocq in parallel and reduce the overall proving time.

Aside from initial proof development, maintaining the proofs is also important.
Changes to the shared specification may require updates to both proofs, while changes to the C implementation may require updates to the VST proof.
No production cryptographic library should rely on a particular proprietary model for proof maintenance.
I have not yet tested whether open-weight models can maintain these proofs.

## Conclusion

libshrincs demonstrates that LLMs can make machine-checked correctness and security proofs practical for a handwritten C cryptographic library.
I expect formal verification to become much more common in applied cryptography and cryptographic engineering.

Human review remains essential, but it can focus on definitions, assumptions, and theorem statements rather than thousands of proof steps.
Formal verification gives us another layer of assurance alongside code review and various forms of testing.

I want to thank @remix7531 for building most of libshrincs.
My own work on the proof relied primarily on LLMs, whereas @remix7531 brought actual understanding of formal verification, Rocq, and VST to the project.

-------------------------

