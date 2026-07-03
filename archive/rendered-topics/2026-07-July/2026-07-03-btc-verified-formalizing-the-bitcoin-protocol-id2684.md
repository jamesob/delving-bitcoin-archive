# Btc-verified: formalizing the Bitcoin protocol

ProofOfKeags | 2026-07-03 16:55:21 UTC | #1

Esteemed colleagues,

Today I'd like to share something I've been working on for the last month or so. I have been attempting to define the Bitcoin protocol in Lean4 and have gotten to a point where I can point to an interesting result in an attempt to demonstrate the type of value that something like this can provide. If you wish to jump immediately to the repository, you can find it here:

https://github.com/ProofOfKeags/btc-verified

# Background

For the uninitiated, Formal Verification is a practice in software development that gives greater assurances than typical testing procedures or forum arguments on how various schemes, protocols, and implementations behave. When we argue about how Bitcoin is designed or how it ought to be designed, engineers will make statements of fact about how the system works, but depending on how much effort is put into the presentation of that argument, it can be difficult to tell if a statement is true, or if it is, why it is true. By formally specifying these statements of fact on how the Bitcoin protocol works we can be precise about what we are saying, what properties we have, what properties we want, and whether the system actually delivers those properties or not.

Normal programming languages do this to an extent, and indeed one of the best ways to settle a dispute is to look at how the code is implemented. However, the nature of C++ can make it difficult to get certainty over certain claims about the behavior of the protocol due to its type system being comparatively weak when judged against the tools typically tasked with this responsibility.

If you want to be able to make specific arbitrary logical statements about Bitcoin or any other software you typically need to specify it in a language with a Dependent Type Theory. The class of tools used for this is called an Interactive Theorem Prover (ITP). There are many, but the one I chose for this project is Lean4 due to its recent momentum and its corpus of mathematical facts defined in the open source "mathlib" which ends up being a useful piece of infrastructure to rely on when you start wanting to ask deeper questions about the protocol and don't want to have to rederive all mathematical knowledge from scratch.

# Goals

Like most open source work this is to scratch a personal itch of mine, but I do think that it has the ability to help settle disputes of fact (as opposed to disputes of value) when we have design discussions suggesting changes to how the system works.

Consider [bip340](https://en.bitcoin.it/wiki/BIP_0340) (schnorr). In that BIP it makes the claims that Schnorr signatures are "[provably secure](https://github.com/bitcoin/bips/blob/861e235e93b40e84e86652ef6e80c2f2dbfc1e17/bip-0340.mediawiki?plain=1#L34)", "[non-malleable](https://github.com/bitcoin/bips/blob/861e235e93b40e84e86652ef6e80c2f2dbfc1e17/bip-0340.mediawiki?plain=1#L35)", and "[linear](https://github.com/bitcoin/bips/blob/861e235e93b40e84e86652ef6e80c2f2dbfc1e17/bip-0340.mediawiki?plain=1#L36)". In that BIP it has good citations about the evidence for the claim of "provably secure" and prose definitions of what those claims are trying to say, but what's cool is that these are all statements that have precise mathematical formulations that can be written down, verified, and later reused in other arguments that rest on those assumptions. At it's best, this project would be able to help document the set of interlocking behaviors in the protocol that allow us to sleep easy about how it works, know where the holes are (yes there are some), and waste less energy arguing about what \*is\*, and save all of our arguments for what \*ought\*.

# Initial Result

Like software, this process of formal verification requires building the protocol up from scratch, and so most of the value still lies in the future, but an interesting first result I stumbled on is related to the Merkle Tree and Root construction inside of Bitcoin blocks.

One of the important properties we rely on in Bitcoin is that a Merkle Root \*uniquely\* identifies the Merkle Tree from which it is derived. This property is usually stated as "non-malleability" and to some may be more easily understood as "injectivity" of the inverse relation. However you wish to name it, we rely on the idea that for an arbitrary Merkle Root, I should only be able to show you a single Merkle Tree that has that root that would be accepted by the Bitcoin network.

Bitcoin Core does this check via the (poorly named) "mutation check" in [ComputeMerkleRoot](https://github.com/bitcoin/bitcoin/blob/d84fc352cbc1363df5cd6024a22e73fc63283e4f/src/consensus/merkle.cpp#L46-L63). This computation that Bitcoin Core has a primary design goal of being efficient to compute, as indeed these types of computations can be expensive during IBD and being smart about how to compute them has a real impact on users running the software in normal operations. However, it is not obvious from the C++ code why this check is there or why it's important. The [prose description](https://github.com/bitcoin/bitcoin/blob/d84fc352cbc1363df5cd6024a22e73fc63283e4f/src/consensus/merkle.cpp#L9-L43) at the top of the file does a decent job of trying to close that gap but still leaves a lot to be desired in terms of understanding how the check fixes the flaw mentioned.

btc-verified instead specifies a notion of [Canonicality](https://github.com/ProofOfKeags/btc-verified/blob/d86d78a5236004a9f3f285935f4104448cc10e91/BtcVerified/Crypto/Merkle.lean#L117-L143), a check that determines whether a tree is defined in its "most reduced form", and a [simple algorithm](https://github.com/ProofOfKeags/btc-verified/blob/d86d78a5236004a9f3f285935f4104448cc10e91/BtcVerified/Crypto/Merkle.lean#L86-L91) for computing the merkle root. Then it specifies the more [memory efficient algorithm ](https://github.com/ProofOfKeags/btc-verified/blob/d86d78a5236004a9f3f285935f4104448cc10e91/BtcVerified/Impl/BitcoinCore/Merkle.lean#L224-L237)used by Core and [proves the equivalence of those algorithms](https://github.com/ProofOfKeags/btc-verified/blob/d86d78a5236004a9f3f285935f4104448cc10e91/BtcVerified/Impl/BitcoinCore/Merkle.lean#L450-L461) for all possible merkle trees. With this equivalence in hand, we can then show that the structure of this computation combined with a [canonicality assumption](https://github.com/ProofOfKeags/btc-verified/blob/1542a806192fe778ea58e8882cc25d55fa8182f4/BtcVerified/Crypto/Merkle.lean#L480) [built from Core's "mutation check"](https://github.com/ProofOfKeags/btc-verified/blob/d86d78a5236004a9f3f285935f4104448cc10e91/BtcVerified/Impl/BitcoinCore/Merkle.lean#L463-L479) implies that a [merkle root identifies a merkle tree or we have a sha256 collision](https://github.com/ProofOfKeags/btc-verified/blob/d86d78a5236004a9f3f285935f4104448cc10e91/BtcVerified/Crypto/Merkle.lean#L347-L357).

This type of result allows us to think about Bitcoin's properties compositionally in a way that stays faithful to the implementation itself.

# Limitations

Before any of you get out your pitchforks, let me discuss a bit about what this project does not claim, because claiming certainty where it doesn't exist is what inevitably gets us in trouble. For obvious reasons this implementation, at any time, can diverge from how Bitcoin Core works and as such should not be used as a final arbiter of how the Bitcoin protocol works. However, it does have the ability to tell us what properties are provably true under the model of implementation specified in btc-verified. Additionally, like in the example above, we can specify Bitcoin Core's actual semantics (along with alternative implementations such as [btcd](https://github.com/btcsuite/btcd)) and prove equivalence to a less computationally efficient but correct-by-construction representation of the same ideas that is more suitable for this type of reasoning.

**This repository is structurally incapable of giving guarantees about [u]Bitcoin Core's[/u] behavior.**

There is a camp of people that says that the Bitcoin Core codebase IS the spec. I have always hated that this is true, but in 2026 it is an accurate statement and safe bet for anyone writing production systems. That said, Bitcoin Core's code is generally not useful as a specification as the requirements are very often obscured by optimization decisions. As such there is still utility in this type of a project where we can map out the architecture of the protocol's soundness more cleanly.

# Disclaimers

1. This is a brand new project and should be treated as a curious "science project" with all of the positive and negative connotations associated with that term.

2. I am NOT a seasoned professional at formal verification. I have had a pet-interest in this level of rigor in reasoning about the correctness of systems, but the tools and techniques are new to me. Those who have worked with me before know that I have been circling these ideas for the better part of a decade, but I am still just a software engineer with math envy and a reading habit.

3. 100% of the code in this repository is authored by AI. However, I have personally laid eyes on every line of code in this repository and have interactively guided decisions at every step along the way.

4. The proof bodies themselves, while they pass the lean kernel, leave a lot to be desired in terms of their legibility. Whether I put work into making the proofs themselves easier to digest will depend on whether there is appetite for being able to read them as opposed to just knowing that the statements are provable and have been verified by the Lean4 kernel.

5. This project doesn't offer much in terms of normative claims about how the Bitcoin protocol SHOULD work and what tradeoffs are desirable, however, I think it has the potential to offer a lot in terms of settling disputes about descriptive claims on how either the current protocol works, how a hypothetical change to the protocol would affect invariants we've come to rely on, or which parts of various implementations are load bearing or not.

# One More Thing

I think the most exciting aspect of this type of work is that since Lean4 is capable of generating working C code, at a sufficient level of maturity, it can make things like [libbitcoinkernel](https://github.com/bitcoin/bitcoin/issues/27587) a formally verified piece of common infrastructure that also discharge skepticism about cross-implementation compatibility.

# CTA

If you have thoughts or opinions on the repo or the line of inquiry I'd love to hear your feedback.

Keags

-------------------------

