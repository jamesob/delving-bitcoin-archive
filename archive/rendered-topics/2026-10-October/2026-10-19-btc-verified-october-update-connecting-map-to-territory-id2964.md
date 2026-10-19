# Btc-verified October Update: Connecting Map to Territory

ProofOfKeags | 2026-10-19 16:32:15 UTC | #1

Originally published [on my site](https://proofofkeags.com/research/2026-10-19-connecting-map-to-territory.html). Replicating here because I think the Delving audience has an interest in this work.

--- 

From the project's inception until now, [btc-verified](https://github.com/ProofOfKeags/btc-verified) has been focused on
building up, layer by layer, a very minimal model of consensus. [Last month's
post](https://proofofkeags.com/research/2026-09-08-the-myth-of-the-valid-transaction.html) talked about how we can discover hidden subtleties in how something works
when we try to formalize it. However, even as we formalize things we have to
resist the temptation to merely assume that the precise map we have created
actually reflects the reality on the ground. This month, however, I decided to
take a stab at anchoring this model to the Bitcoin network as it exists in the
real world.

It's tricky to do this type of thing because it forces us to ask the question
how do we know the map matches the territory? How close can we get?

For most software that you want to verify it is an option on the table to simply
write the software in a verifiable language, compile that down to code that
actually runs on silicon and be on your merry way. Alternatively, if the code
you're trying to verify is pure C code, there are ways to [verify the C code
directly](https://www.frama-c.com/fc-plugins/wp.html) since C's operational semantics are small enough to reason somewhat
efficiently about.

The problem here is that [Bitcoin Core](https://github.com/bitcoin/bitcoin), which on a practical level *defines* the
Bitcoin protocol, is neither written in a verifiable language, nor is it written
in pure C. Rather, it is written in modern C++ so verifying it directly, while
possible on theoretical grounds is very far out of reach with my current
resources.

So what can we do instead?

The main thing we are trying to establish is that our model does not differ from
Bitcoin Core in any observable way with respect to its protocol semantics.
Hypothetically, if we were able to come up with an implementation of the
protocol in [Lean](https://lean-lang.org/) that had no differences in behavior from Bitcoin Core, it would
mean that the model, in a sense, IS Bitcoin Core. If the model predicts reality
with 100% accuracy, we can say that the model is "correct".

This seems like a simple enough thesis, but it is far from well defined enough
to build something around. What constitutes a behavior? How can we observe the
behavior? How can we tell if it's different or not? What things are
implementation details and what things are essential semantics?

People have been trying for years to extract out Bitcoin consensus from the rest
of Bitcoin Core and the most recent label for this effort is the
[libbitcoinkernel project](https://thecharlatan.ch/Kernel/). This carves out a chunk of Core inside of which is the
consensus verification code and outside of which we can safely write all kinds
of custom implementation experiments without compromising the safety of
consensus. The original purpose of this project is actually to reduce the usage
of non-Core implementations of consensus. However, it also gives us a natural
entry point for our purposes here.

To do this carve out, libbitcoinkernel must define an interface that preserves
the essential invariants of consensus and for it to be usable, its interface
must give us the ability to know whether something is consensus valid or not.

What I have decided to do is to take advantage of the existence of this
interface and ensure that btc-verified can serve as the implementation behind
it. We are not doing this to encourage people to run btc-verified's consensus
instead of Bitcoin Core's. We are doing this to set up a benchmark of
btc-verified's faithfulness of matching Core's behavior on the faultline that
has already been drawn by the Bitcoin Core team.

Once we have established a trivial implementation of libbitcoinkernel's
interface in the btc-verified project, then we can take advantage of another
technique that has an established practice in the space: differential
fuzzing.

[bitcoinfuzz](https://github.com/bitcoinfuzz/bitcoinfuzz) is a project whose aim is differential fuzzing of different bitcoin
implementations. The idea here is that we spew entropy at the kernel interface
and ensure that for any given payload that Core's kernel and btc-verified's
kernel report the same result, including any state changes that are made by the
operation executed.

bitcoinfuzz already has an established track record and has [found numerous bugs
in different implementations](https://github.com/bitcoinfuzz/bitcoinfuzz#bugsinconsistencesmismatches-found-by-bitcoinfuzz) using this strategy. Indeed, short of proofs, the
best we can do to establish equality is to do high-entropy sampling of the
behavior of each implementation and ensure they never diverge.

Of course, you never know if the next sample you take is the one that reveals a
difference.  This is the exact reason that formal verification even has a place,
because you cannot establish universal laws through sampling alone, but it does
have a practical benefit of reducing the probability of divergent behavior the
more that the sample space saturates the control flow graph.

Following this thread, I put together a
[standalone differential fuzzing harness](https://github.com/ProofOfKeags/btc-verified/blob/5cd2d824784dc2fac33aec641a0e10fcf869d0b2/Fuzz/README.md)
for Bitcoin Core and btc-verified, specifically against the kernel interface, and
we have [nine of the kernel functions](https://github.com/ProofOfKeags/btc-verified/blob/5cd2d824784dc2fac33aec641a0e10fcf869d0b2/Kernel/README.md#public-c-interface) backed by a Lean implementation. These
functions cover transaction creation, copying, serialization, checking, and the
associated validation-state objects. The harness is built directly on [libFuzzer](https://llvm.org/docs/LibFuzzer.html).
It gives the same transaction bytes to Core and btc-verified, then compares their
observable behavior.

As of right now btc-verified does not implement the entire kernel interface even
trivially, but it does implement a subset of it with apparently correct
semantics for the following behaviors:

- **Parsing:** Given identical bytes, do both implementations accept them as a
transaction object? This is distinct from accepting the transaction as valid.
- Serialization: For accepted inputs, do both report the same serialization
status and produce exactly the same bytes?
- **Transaction-local checking:** Do they agree on the verdict for rules concerning
input/output presence, duplicate or null input references, output-value and
stripped-size limits, and the special coinbase conditions? These checks don’t
consult the UTXO set or execute scripts.
- **Reported validation state:** Do they return the same validation mode and result
category, beyond merely agreeing on acceptance or rejection?
- State reuse: After checking a known-invalid transaction, does checking the
next transaction replace that previous result consistently—including clearing
the invalid state when the next transaction passes?

As of this writing these behaviors are identical between btc-verified and
Bitcoin Core when observed over this battery:

- **100,000 fuzz executions**: this measures how many attempts for
disagreement were made. Each execution feeds one byte sequence into the
differential harness, which runs both implementations and compares their
observations. This measures testing volume. It doesn’t count unique
transactions: inputs can repeat, and malformed inputs may be rejected during
parsing without reaching transaction checking. Mutations are also related to
earlier inputs, rather than independent random samples.
- **11 regression inputs:** this measures how many pre-determined cases are
explicitly replayed. These ensure particular cases are checked every time,
regardless of what mutation happens to discover. “Regression” here doesn’t mean
each input previously exposed a bug. The ones here include real legacy and
SegWit transactions, malformed encodings, a transaction that parses but fails
checking, and three transactions immediately below, at, and above the
million-byte stripped-size boundary. Those large cases matter because the
mutation campaign’s 4,096-byte limit prevents it from discovering them.
- **264 retained corpus inputs:** this was the size of the working corpus at the
end of the [October 18 CI run](https://github.com/ProofOfKeags/btc-verified/actions/runs/39158271308/job/117765944780).
LibFuzzer keeps inputs that contribute useful coverage feedback and uses them as
starting points for further mutations. The final corpus count describes that
working collection, including retained starting seeds.

This differential test additionally runs as a [CI check](https://github.com/ProofOfKeags/btc-verified/blob/5cd2d824784dc2fac33aec641a0e10fcf869d0b2/.github/workflows/ci.yml#L77-L106) before new code is merged
allowing observed disagreement with Core to block changes to the code.

There is a ton of work still to do to strengthen the link between btc-verified
and Bitcoin Core but assuming we can establish a credible link between them it
allows us to borrow some of the theorems and findings we can establish natively
in btc-verified and use them to make statements about Bitcoin Core's behavior.

Over the coming month the plan is to push forwards on filling out the rest of
the kernel interface so we can officially connect this map to this territory,
at least tenuously for now, before pushing forwards on analyzing the map itself.

-------------------------

