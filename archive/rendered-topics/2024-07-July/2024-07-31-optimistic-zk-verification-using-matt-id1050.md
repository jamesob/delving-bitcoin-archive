# Optimistic ZK verification using MATT

halseth | 2024-07-31 12:54:53 UTC | #1

Earlier I posted about [Elftrace](https://github.com/halseth/elftrace), a tool for creating Bitcoin Scripts to verify RISC-V execution: https://delvingbitcoin.org/t/verification-of-risc-v-execution-using-op-ccv/313 

Since then it has had some updates, and I wanted to share that the toolchain has had some additions making it easier to compile Rust without having to write custom assembly for every new program. It now also support doing input/output to the program using standard IO.

With this in place we are able to compile the **Winterfell STARK library** as a dependency, making it possible to _verify-the-verification_ of  ZK proofs in Bitcoin Script (this still requires CAT and a covenant).

A more detailed writeup is here, with example code: https://github.com/halseth/elftrace/blob/831f537bc1509bd45c350e103f7fc73aa818f7dd/docs/zk_verifier.md

-------------------------

AdamISZ | 2024-12-15 16:06:16 UTC | #2

From a very high level view and a quick read, this seems like an extremely promising direction. Can you expand a bit on the necessity for OP_CAT and covenants? At the end of the writeup it notes

>" and use Bitcoin Script to verify its execution. In order to achieve this we need a covenant like `OP_CHECKCONTRACTVERIFY` to carry state across transactions."

Why is that needed? Is it needed in a model using fraud proofs? (I'm vaguely remembering the basic idea of bitvm and earlier optimistic rollups etc). Apologies for the slightly uneducated question :)

-------------------------

salvatoshi | 2024-12-16 13:14:25 UTC | #3

In MATT, `OP_CAT` is only used for the purpose of creating vector commitments, as that enables checking Merkle proofs; other opcodes like [`OP_PAIRCOMMIT`/`VECTORCOMMIT`](https://github.com/bitcoin/bips/pull/1699) would do the job, too.

About the need for a covenant: it is used in order to execute a protocol as an arbitrary state machine across multiple UTXOs, where each 'transition' can do some computation on the *state* committed in it (if any), and dynamically compute the state for the 'next step' (which corresponds to a different node in the state machine of the protocol). Fraud proofs for any computation are a use case. Notably, no presigned transaction is needed for such protocols, as all the possible futures are directly preprogrammed in Script thanks to the covenant.

I sketched the fraud proofs in the [initial post about MATT](https://gnusha.org/pi/bitcoindev/CAMhCMoH9uZPeAE_2tWH6rf0RndqV+ypjbNzazpFwFnLUpPsZ7g@mail.gmail.com/) (starting from *Commitments to computation and fraud challenges*), with a toy example [here](https://gnusha.org/pi/bitcoindev/CAMhCMoEONv3jriBU3qwm0pt75iF_sgra1H2Z4rOF8u+e7hs_cw@mail.gmail.com/). There's also a [python implementation](https://github.com/Merkleize/pymatt/blob/55c9239288ae9b4900a251bfe5d8fc0f99308491/matt/hub/fraud.py) of the bisection protocol, which contains a formal description in the comments.

I collect all the resources related to MATT at https://merkle.fun, if you're interested to go more in-depth on any of these topics, or check the existing code. Please don't hesitate to reach out if you have any question.

-------------------------

