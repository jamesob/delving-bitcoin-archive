# Block-wide Signature Aggregation via SNARKs

conduition | 2026-09-04 18:17:56 UTC | #1

The core idea is simple: PQ sigs are big but cheap to verify. The huge size slows down block propagation and puts bigger storage/bandwidth costs on archival nodes forever. What if we could compress these signatures with a PQ-SNARK? (we do not need zero-knowledge properties here). I've heard this technique called "BitZip".

I'm not sure where this idea originated, but I first heard about it from @EthanHeilman's thread here:

https://delvingbitcoin.org/t/post-quantum-signatures-and-scaling-bitcoin-with-starks/1584

Research around this subject has renewed in the cryptocurrency industry owing to the recent developments of efficient binary circuit provers.

- Hash-based sigs with binary hashes like SHA256 can now be verified in a SNARK with highly efficient provers. See [Binius](https://www.binius.xyz/) and [Flock](https://blog.succinct.xyz/introducing-flock/) as examples.
- [The Ethereum foundation is pursuing this architecture](https://x.com/drakefjustin/status/2087905684180418733).
- AutoResearch communities have found even faster provers than those given by the initial Flock paper: https://www.yukon.org/flock

[Discussions are ongoing about providing a new witness discount for PQ signatures](https://delvingbitcoin.org/t/segwit-commitment-to-post-quantum-witness-data/2702).
This would massively increase block size and cause chain growth to balloon from a few hundred GB per year now, into the ballpark of terabytes per year.
Unless storage cost/GB rates drop significantly, or we consider more extreme options like sharding or regular checkpoints or something like that, then Bitcoin archival nodes would not be practical for long in such a world.

I am personally a big fan of SNARKs as an option to avoid this outcome, for many of the reasons @EthanHeilman pointed out originally. I would like to reopen discussion around its feasibility.

Particularly i would like to suggest a few key decision points, and articulate their most significant trade-offs.

## Choice 0: Who proves?

SNARKs are exceptionally asymmetric in their performance profile between prover and verifier.
The prover does a lot of work up-front, so that the verifier needs to do very little.
@EthanHeilman's post above assumed the miner does the proving to aggregate signatures, but this is not necessarily the case.
Proving can be outsourced.

One potential design is to introduce a new role in the Bitcoin P2P layer: **Aggregator nodes:** independent users with powerful general-computing hardware (CPUs and GPUs).
Each aggregator regularly builds and publishes SNARK proofs for a specific block template, based on transaction inclusion rules they define locally, perhaps with sane defaults. They earn no money from doing so, but they do gain some benefit: The aggregator can include their own transactions in the template.
If a miner uses the aggregator's proof, the aggregator's transactions must be mined (or at least, the same signatures must be verified at some point in the block, which is best done by including transactions that reference them).

Given that aggregator nodes are not economically incentivized though, we should not depend on this. Miners may also prefer to pick their own transactions to better fit business/legal constraints, or to optimize revenue.
For these reasons, we should assume that especially large mining pools will do proving on their own.

We should especially not assume average validator/relay nodes will do any proving.
SNARK proving work is hard and complicated, and so introducing a SNARK prover into the Bitcoin P2P relay codebase could easily introduce DoS vulnerabilities.

## Choice 1: VM or Circuit?

The Ethereum Foundation is pursuing a design that uses a hash-based SNARK to prove correct execution of a VM (leanVM). The actual signature verification algorithm is then implemented in a higher-level language which compiles to bytecode in the VM. The proof statement is *"This program executes correctly given this input."*

This allows the ETH folks some flexibility, because i believe they reuse the same VM elsewhere in their stack (e.g. consensus layer, execution layer). They can also more easily implement verifiers for recursive SNARKs (see below) because they can implement the recursive proof verifier in the VM.

However for Bitcoin, this might not be the right trade-off.
We plan to use this SNARK only for one bespoke purpose: aggregating signatures.
The additional VM layer used by ETH adds about a 10x proving performance overhead compared to a bare circuit, and would require a fully audited compiler toolchain built for the highly specific task of aggregating signatures.
This compilation layer may also introduce critical vulnerabilities (soundness bugs) if the compiler or program is designed maliciously. [Relevant paper](https://eprint.iacr.org/2026/1838).

I would therefore reason that the additional complexity of a VM is not worth it, and so we should elect instead to design a bespoke circuit specific to the purpose of aggregating signatures. This circuit can be proven secure more easily, especially if its design is transparent and reproducible.

## Choice 2: Flat or Recursive?

It is possible to prove validation of a SNARK inside a SNARK. This can even be recursed further, so that a SNARK proves a SNARK that proves a SNARK, etc.

In the context of non-interactive signature aggregation, recursive SNARKs give some interesting flexibility for relayers and miners.
The main benefit in the Bitcoin signature aggregation context would be that miners could use recursive proofs to incrementally (cheaply) update a proof for an aggregated set of transaction signatures, as new transactions enter their block templates.
Without recursive proofs, these same miners may need to redo all the proving work every time a signature is added to the aggregation batch, which could be considerable (multiple seconds of compute). This property (incremental updates) is one of the main reasons why the Ethereum folks are pursuing recursive proofs in a VM.

However, recursive proofs also introduce more proving overhead (per signature), more bug surface, and are much harder to do without a VM. Therefore I would suggest we do not use recursive proofs, and opt instead for a single flat circuit design which proves a simple non-recursive statement.

## Choice 3: Which Signature Scheme?

The choice of signature scheme heavily affects the design and performance of the SNARK system we use.
It is also critical for security that the signature scheme itself be both sound and zero-knowledge.

Hash-based signatures are the obvious choice here for tight security with minimal assumptions, but it may also be possible to use lattice schemes. More research is necessary here, but for now I'll assume hash-based signatures.
The general ideas outside this section apply equally well to both families.

In a SNARK, the prover's time/space complexity is linearly proportional to the size of the computation being proven, so reducing the compute cost of verifying signatures has a big impact on the efficiency of a signature-aggregation prover.

While I'd love to recommend [SHRINCS](https://github.com/SHRINCS/shrincs-bip/blob/3dd2c5d01adbc79b82365df4a9ca85826768dbb8/SHRINCS.md) for the role, the statefulness trade-off in SHRINCS is primarily a trick to buy us smaller signatures, but in this case signature size is much less important.
If we're going to aggregate signatures we probably want to go fully stateless to avoid the headaches of dealing with state management.
Likely this means a SPHINCS variant (e.g. SPHINCS+C).

Notably, using the official SLH-DSA spec is probably a bad idea here, because SLH-DSA requires a variable number of hash operations in the verifier, and the variance is controlled by the signer.
We can compute the upper bound on verification cost, but still the difference between average-case and worst-case could be as much as a 2x difference or more depending on parameters.
The SNARK circuit must account for the worst-case bound, even if this bound is never reached.
We can get therefore improve SNARK prover performance by using a variant of SPHINCS which offers a cheap constant-work verifier.

If we simply take SPHINCS+ (SLH-DSA) and replace WOTS-TW with WOTS+C, we can find parameter sets that require only a few hundred hash invocations to verify, and the verify workload doesn't change between signatures.

Take this parameter set for instance: `h=36 d=4 a=13 k=15 w=4`.
This parameter set has about 3x faster signing than SLH-DSA-128s, 8304-byte signatures, and with WOTS+C requires only \~700 SHA256 hash compressions to verify.
We can reduce the verification cost further if we can stomach other trade-offs, like larger signatures, more expensive signing/keygen, lower signature budgets, or more exotic algorithms (e.g. FORS+C, PORS+FP).

We can also improve prover performance by changing the hash function used for the hash-based signature, e.g. to Blake, which is faster to prove over binary field circuits, but this might be controversial, and non-SNARK verifiers would lose out on the benefits of SHA256 hardware acceleration.

## Choice 4: Proof Size?

The smallest hash-based SNARKs are typically 300-500kb in size.
No one seems to expect them to get any smaller, so I will assume this is the floor.

On the other hand, Giacomo Fenzi recently pointed out to me the fact that larger proofs admit faster provers.
If we are OK with accepting a large SNARK proof size, we may be able to increase prover performance (possibly verifier performance too?).

Notably, average blocks today by raw byte volume consist roughly half of witness data (mostly pubkeys and signatures), and half block data.
So if proofs are say 700kb, this would roughly match the block space usage of the signatures used today in Bitcoin.
I'm not sure what the "right" number is here, but we should keep it in mind as a parameter we can play with.

## Choice 5: Soft-fork Upgrade Hooks?

One of the biggest risks with introducing SNARKs into Bitcoin consensus is the possibility for soundness bugs.
If such a bug is found & exploited, provers could use it to prove false statements, which in this case means miners (or aggregators used by miners) can steal honest users' coins via a block-wide SNARK that falsely proves validation of invalid or non-existent signatures.

Furthermore, SNARKs are a very rapidly progressing technology. Binius was published only in 2024, and Flock only just a few months ago in mid 2026.
It seems likely that by the time we standardize and enshrine a SNARK system in Bitcoin consensus, the research community might well have made a lot of progress improving performance, compactness, security, or other properties of SNARK systems.

It'd be *really nice* if we had a way to disable the SNARK system in an emergency without confiscating users' funds or causing a hard-fork, while also leaving the system open to future upgrades if better SNARKs are developed in the future.

One way we can do both is to enshrine a **blockheight-based upgrade hook rule** in consensus when SNARKs are first deployed as a valid spending mechanism.
E.g: *"If the block height is greater than $N$, the SNARK-enabled output-type / leaf-script-version is anyone-can-spend."*

We could set this height to some large interval after deployment time, e.g. 10 years.

### How does this work?

Let's denote the first deployment of SNARK signature aggregation as **"SNARKv1".**
Let's denote the output type that supports spending via SNARK aggregation as **"P2QR"** (named after the terminology in [this discussion](https://delvingbitcoin.org/t/pqc-output-type-discussion/2749)).

If at some point in the first 10 years after deployment time, a soundness bug is discovered in the SNARKv1 system, there may be a significant fraction of active coins using P2QR and depending on SNARKv1 validation for spending.
If we disable SNARKv1 outright, all P2QR coins without a secondary spending condition would be confiscated.

But because of the upgrade hook deadlined for 10 years after SNARKv1 deployment, we don't need to do that.
We can instead activate a soft fork to disable SNARKs during the interval between SNARKv1 and the T+10yr hook height.

At T+10yrs (and no later), we deploy a new & improved - hopefully more secure - SNARKv2, which proves the same statement as SNARKv1.
As long as the underlying signature scheme remains secure, users do not need to migrate their coins.
This is a soft-fork because from the perspective of SNARKv1 nodes, the consensus rules have only tightened.

Notice SNARKv2 could use a completely different proving system to SNARKv1.
There's no reason it has to even be the same architecture.
Perhaps by the time SNARKv2 is being developed, we might have fancier and more efficient SNARK systems available.
If not, we can just deploy a simple fork that kicks the can down the road, and re-enables SNARKv1 for another 10 year period or something like that.

To ensure users can still spend coins at all times even if SNARKv1 is disabled, we can allow signers to opt out of aggregation, as in [CISA](https://github.com/bitcoin/bips/pull/2212). Opted-out signatures would not be aggregated, and would instead be included directly in transaction witness data at standard discount rates.
The signatures would be huge and expensive, but at least users would still access to their money if they urgently need it.
If a user has low time preference, she may opt to simply wait for SNARKv2 deployment - there is no urgency to migrate away from P2QR.

A big problem occurs if SNARKv2 is never deployed (e.g. it gets held up in consensus debate): P2QR addresses will become anyone-can-spend.
If P2QR is in heavy use, this seems unlikely to me - There would be too much at stake for inaction to dominate.

Another notable risk would be if a soundness bug is found, but the soft fork to disable SNARKv1 doesn't happen fast enough. This leads me to...

## Choice 6: Tripwires?

Closely related to the previous design choice.

Given the implementation risk of SNARK soundness bugs, we may want to deploy SNARKs along with some kind of canary or tripwire system ([see related thread](https://groups.google.com/g/bitcoindev/c/aWYtPLVPZ3U/m/htpzI5r3AgAJ)) that could be used to disable SNARKv1 temporarily (until SNARKv2 is deployed to fix any bugs).

It seems feasible that if the SNARKv1 verifier is ever broken so badly to the point where a false proof can be forged that proves verification of invalid signatures, then an honest miner may be able to activate a soft fork on short-notice by forging such a proof over a provably unusable (NUMS) public key.

For SPHINCS this is easy: A hash of fixed public data using the same hash as in SPHINCS should map to an unusable public key. 
A valid signature from that key would constitute a second preimage attack on the hash function.

Verifiers could have a simple rule that says *"If I verify a proof that proves a valid signature was issued by this specific NUMS public key, disable SNARKv1."*
Naturally, the disabling of SNARKv1 would not disable the future upgrade hook for SNARKv2, if applicable.

Many of the same design considerations apply here as in the case of CRQC canaries, and we could benefit from a common modular/reusable soft-fork design.

## Choice 7: Committed or loose?

Today, everything that makes a Bitcoin block valid (namely, the signatures) is committed into the block hash, and is thus covered by the mining proof-of-work.

If we were to upgrade Bitcoin with a new output type or some other means of spending that uses block-wide SNARK aggregation, then we tend to naturally assume the SNARKs will also be covered by the PoW in the same way.
However, a potentially tempting design decision could be to decouple the SNARKs from the block hash.

To see why this would be useful, let's first assume SNARKs *are* committed into block hashes, and thus are covered by PoW.
Then to start mining, the miner must first find or produce a SNARK - Otherwise they are mining an invalid block.

If proving takes, say, 10 seconds to aggregate signatures together for a given block template, then for those first 10 seconds the miner's ASICs are sitting idle and useless - This is lost revenue for the miner.
By itself this would be fine if proving is quick compared to the 10m block interval.
It might give a slight advantage to miners with good proving hardware, but more or less it's an equal effect across the network.

However, miners can (and would) choose to salvage some revenue by mining the easiest possible block template to prove valid with a SNARK: **An empty block template.**
So if we go this route, we should expect to see empty blocks more frequently, as miners have even stronger reasons to mine them during the initial mid-proving phase.

If we decouple the SNARK from the block hash, then miners can mine in parallel with proving.

As pointed out by @sipa though, this gives us a tough situation.

1. If blocks are valid with _either_ a valid set of signatures, or a SNARK proving the same, then resource constraints like witness discounts must account for the worst-case, which will be the full set of signatures.
2. If blocks are valid only with signatures, then SNARKs are not required and we end up with huge block witnesses which must be relayed and stored _somewhere._
3. If blocks are valid only with a SNARK, the miner still has to finish producing the SNARK before they can broadcast their block.

For a while now, I believed decoupling SNARKs from PoW and allowing two valid block formats (option 1) could be a great way to mitigate the empty-blocks issue.
But now I think I'm realizing the complexity and DoS risks of allowing both formats would be too great. (@sipa convinced me in DM)

For option 2, SNARKs aren't required in consensus at all; We'd have huge blocks and discussion about SNARKs would be deferred to the P2P layer; we'd need to explore sharding or checkpoints as alternatives.

That leaves option 3, which might help a little.
More research is needed, especially discussion with miners.

Or else don't do any of these options: Just include SNARKs in the block hash, and bear the possibility of a few more empty blocks.
We can mitigate the risk as best we can by speeding up SNARK proving performance.

-----------

## Conclusion

So that's a rough sketch of all the various design considerations I know of around SNARKs in Bitcoin. This is a mish-mash of both cryptographic and engineering considerations, and there's a lot of them. 

Given the volume of work that'd be needed to make block-wide SNARK aggregation technically possible and palatable to the community, I don't think we should expect this to happen soon. I would prefer if we still aim to have a more simplistic and minimal soft-fork package ready sometime in the next 2 years so that users can start migrating to PQC-supporting wallets ASAP, even if those wallets' on-chain throughput would not scale enough to be used as the common standard after Q-day.

regards,
conduition

-------------------------

evd0kim | 2026-09-05 20:01:58 UTC | #2

Why BitVM is not considered here? Technically it should be possible to wrap anything into groth16, hence leverage existing tech. LeanVM is the brand new tech which apparently ascends to abandoned project binius.

-------------------------

conduition | 2026-09-06 15:25:35 UTC | #3

I'm not sure how BitVM is relevant here. 

Groth16 is quantum-insecure and requires trusted setup, so it doesn't seem like a good option.

LeanVM is more interesting, and if it ends up working well perhaps we could integrate that. But as i said, you have a 10x performance overhead and the main benefit from doing so is portability and DX, which is not the major concern here.

-------------------------

