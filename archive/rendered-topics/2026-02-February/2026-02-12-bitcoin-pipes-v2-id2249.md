# Bitcoin PIPEs v2

nemothenoone | 2026-02-12 16:56:49 UTC | #1

# Bitcoin PIPEs v2: Covenants and ZKPs on the Bitcoin L1 via Witness Encryption.

> PIPE v2 \[2\] explores a cryptographic alternative to Bitcoin covenants that operates entirely within existing consensus rules. Instead of extending Script or relying on optimistic dispute mechanisms, PIPE v2 enforces spending conditions by *witness-gating access to a Bitcoin signing key*. A valid Schnorr signature can be produced if and only if a specified NP statement is true. This yields a non-interactive mechanism deployable without a soft fork, in which Bitcoin funds become spendable if and only if a specified condition is satisfied, sufficient for vaults, exits, fraud proofs, and zk-proof gated releases, under well-defined cryptographic assumptions.

Bitcoin’s transaction validation model is intentionally minimal. At the consensus layer, Bitcoin verifies only that a transaction correctly spends unspent outputs and that each input is authorized by a valid digital signature. This simplicity is a defining strength: it keeps consensus rules stable, auditability high, and the security model well understood. At the same time, it places hard limits on the kinds of spending policies that can be enforced directly on-chain.

Several approaches try to address this limitation. Script extensions such as `OP_CTV`, `OP_CAT`, and `OP_CSFS` increase expressiveness but require a soft fork which is a change to Bitcoin’s consensus rules. Optimistic protocols such as BitVM avoid consensus changes, but introduce interactivity, challenge periods, and liveness assumptions. These tradeoffs are acceptable for some applications, but not all.

PIPE versions \[1,2\] explore a different point in the design space: enforcing spending conditions **without new opcodes and without optimistic challenge mechanisms,** by shifting enforcement entirely *out of Bitcoin Script and into cryptography*. Rather than asking Bitcoin to verify whether a transaction satisfies a condition, PIPE v2 makes it *cryptographically impossible* to produce a valid signature unless that condition holds.

This post presents PIPE v2 through the abstraction of **witness signatures**: conditional signature schemes in which producing a valid signature is equivalent to proving membership in a hard NP relation. We explain how PIPE v2 realizes witness signatures using witness encryption, characterize the resulting authorization semantics, and clarify its scope.

## Core Idea: Enforcing Spending Rules Cryptographically

Let’s think what Bitcoin already provides.

* At the consensus layer, Bitcoin exposes a single authorization primitive: **signature validity**. A transaction input is accepted if it carries a valid Schnorr signature under the public key specified by the spent output. This is the only condition Bitcoin enforces for spending authorization.
* What Bitcoin deliberately does not encode is *why* a signature was produced. Bitcoin Script does not capture intent, context, or the computation that led to a signature.

This fully characterizes the available design space. Any spending rule that works **without changing Bitcoin’s consensus rules** must ultimately boil down to a single question:

> Can a valid signature under this public key be produced or not?

There is no other hook available to enforce authorization.

The key insight behind PIPEs is to treat this constraint not as a limitation, but as the design surface.

> If signature validity is the only thing Bitcoin enforces, then the strongest spending rule we can implement without a soft fork is one that controls **whether a signature can be produced at all**.

PIPE v2 do exactly this by moving enforcement *before* signing. Instead of asking Bitcoin Script to check a condition, PIPEs attach the condition to **access to the signing key itself**.

In a PIPE, the signing key corresponding to a Bitcoin address is not directly available. Instead, it is cryptographically locked behind a condition—such as the existence of a valid zero-knowledge proof, an exit proof, or a fraud proof. If the condition is satisfied, the key can be recovered and used to produce an ordinary Schnorr signature. If the condition is not satisfied, producing a valid signature is computationally infeasible.

From Bitcoin’s perspective, nothing changes. The transaction contains a standard signature under a standard public key and verifies as usual. From a protocol designer’s perspective, this extracts the maximum possible programmability from Bitcoin’s existing authorization model: arbitrary off-chain logic can gate spendability, without new opcodes, without interactivity, and without changing consensus.

### Formalization: PIPE as a Witness Signature

To instantiate this idea, we introduce **witness signatures** and the corresponding **PIPE construction (PIPE v2).**

At a high level, a witness signature is just a normal Bitcoin signature but with a twist:

> A valid signature can be produced if and only if a specific external condition is satisfied.

A PIPE consists of two main phases: **setup** and **signing**.

During setup, a standard Bitcoin key pair is generated, but the private key is never revealed. Instead, it is cryptographically locked behind a condition such as the validity of some off-chain computation. This locking is done using **witness encryption**, which allows the private key to be recovered only by someone who can produce valid evidence (a *witness*) that the condition holds.

During signing, a party that has such a witness tries to recover the private key. If the witness is valid, key recovery succeeds, and the party can produce an ordinary Schnorr signature. If the witness is invalid, recovering the key and thus producing a valid signature is computationally infeasible.

### PIPE v2 Construction (at a glance)

* **Setup** *(performed in a distributed manner)*

  * Generate a standard Bitcoin key pair $(sk, pk)$.
  * Fix a spending condition, expressed as a statement (e.g., a circuit $C$ with public input $x$).
  * Encrypt the signing key $sk$ under this statement using **witness encryption**.
  * Publish the public key $pk$ and ciphertext.

  (No party learns $sk$  during setup, assuming at least one setup party behaves honestly i.e., no full collusion.)

* **Sign**

  * Input: a message to be signed and a witness $w$.
  * If $w$ is a valid witness for the statement, decrypt the ciphertext and obtain $sk$
    * Generate a Schnorr signature with $sk$.
  * If $w$ is invalid, recovering $sk$ and producing a valid signature is computationally infeasible.

* **Verification (Bitcoin)**

  * Bitcoin verifies the resulting Schnorr signature under $pk$ as usual.
  * No statement, witness, or auxiliary data appears on-chain.

> **Research Status:** At \[\[alloc\] init\], we are leading ongoing research on the cryptographic core of PIPE v2, with a particular focus on witness encryption. In addition to the abstract PIPE construction, we investigate how witness encryption can be instantiated in a way that is both analyzable and scalable. To this end, we introduce a withness encryption scheme based of Adaptive Arithmetic Determinant Programs (AADP) \[3\].
> Alongside this, we systematically analyze prior and contemporary witness-encryption constructions.

## Covenant Semantics: What PIPEs Enforce (and What They Don’t)

PIPEs v2 do not attempt to reproduce the full range of covenant semantics proposed for Bitcoin Script. Instead, they realize a specific and intentionally limited class of spending rules.

### Binary NP Covenants

PIPE v2 enforces what we call **binary covenants**. At a high level, a PIPE enforces exactly one decision about a UTXO:

> Is spending authorized or not?

Concretely, a PIPE controls whether the signing key for a UTXO is released. No additional structure is imposed on the spending transaction itself. PIPEs do not constrain the transaction format, outputs, or fee structure once authorization is granted. After the key is released, the signer holds an ordinary Bitcoin private key and can construct any valid transaction.

This model naturally captures a wide range of useful conditions whose outcome is binary. Examples include:

* **“A valid zk-proof/ SNARK proof is provided.”**

  For example, a proof that a condition holds (e.g., eligibility, membership, or correctness of a hidden computation).

* **“A fraud proof exists.”**

  Used to unlock funds only if incorrect behavior can be shown.

* **“An exit condition is satisfied.”**

  Such as proving inclusion in a commitment, knowledge of a secret, or eligibility to withdraw from an off-chain system.

* **“A state transition is valid.”**

  For instance, verifying that a rollup, bridge, or metaprotocol update follows the agreed rules.

In all cases, the covenant enforced by PIPEs answers a single question: *is the condition satisfied or not?* If yes, the funds become spendable; if no, they remain locked.

> This tradeoff is intentional. **PIPE v1** \[1\] already achieves non-interactivity and consensus compatibility, and serves as a proof of principle for cryptographic enforcement of spending conditions on Bitcoin. However, it relies on highly expressive cryptographic machinery (e.g., Functional Encryption, Indistinguishable Obfuscation) that is difficult to instantiate efficiently with current techniques. **PIPE v2** \[2\]deliberately narrows the scope to one-time, binary authorization conditions. This simplification reduces cryptographic complexity while preserving the core guarantee: spending is possible if and only if a specified condition is satisfied.

## **Why Binary Authorization Is Often Enough**

In practice, many useful Bitcoin applications require exactly this level of control.

### Vaults

Vaults are a canonical example. In a vault, the core requirement is not to micromanage how funds are spent, but to ensure that **withdrawal is possible only if a specific condition is met**. For example:

* a beneficiary proves eligibility,
* a recovery condition is satisfied,
* an external certificate or proof is provided.

Once the condition is met, the vault’s purpose is fulfilled. Imposing further constraints on the withdrawal transaction itself is often unnecessary. PIPEs naturally match this pattern: the vault unlocks if and only if the condition holds.

![pipesv2|690x367](upload://azGHmolvnXrgUWXe0MfwB2EUlRy.png)

### Exit and Escape Conditions

Many Bitcoin constructions include *escape hatches* or *exit paths*. These are not long-lived policies, but one-time conditions that allow funds to be recovered safely.

Examples include:

* exits from off-chain protocols,
* recovery paths from stalled systems,
* fraud or misbehavior proofs.

In these cases, the relevant question is binary: *does a valid exit proof exist or not?* Once it exists, the funds should be released. PIPEs enforce exactly this logic, without requiring interactive disputes or challenge periods.

### Proof-Gated Releases

ZK proof or SNARK proof systems typically attest to a single fact:

* a state transition is valid,
* a computation was executed correctly,
* a committed value satisfies a predicate.

Such proofs already produce a boolean outcome: valid or invalid. PIPEs align directly with this model. They turn proof validity into spending authorization, without requiring Bitcoin Script to verify the proof itself.

### Optimistic Protocol Finalization

Even in optimistic designs, many outcomes ultimately reduce to a binary decision. After the dispute window closes, either a fraud proof was presented or it was not. Once that decision is made, no further enforcement is needed.

PIPEs allow such finalization conditions to be enforced non-interactively: the existence of a valid proof directly unlocks funds, without on-chain disputes or timing assumptions.

## Opcodes Examples

* `OP_VAULT.`According to the BIP (https://github.com/bitcoin/bips/pull/1421), `OP_VAULT` is valid only inside a Taproot leaf, and it enforces the following:

  * Trigger path. There must be an output (trigger output) whose index and amount are taken from the stack. That output must be locked with the same Taproot tree, except that the currently executing leaf is replaced by a new script taken from the stack. The opcode recomputes the modified Taproot commitment and checks that the actual transaction output matches it (both scriptPubKey and amount).
  * Optional recovery path. There may be a recovery output whose index and amount are also taken from the stack. That output must be locked with the original Taproot (unchanged tree) and have the specified amount (typically smaller).
  * From a low-level point of view, this means that `OP_VAULT`:
    * takes trigger/recovery output indices and amounts from the stack,
    * directly reads the corresponding outputs of the spending transaction,
    * recomputes the expected Taproot commitments,
    * and fails script evaluation if they do not match.

  This way `OP_VAULT` enforces a predicate over the spending transaction structure and it is not possible to emulate `OP_VAULT` directly with a single PIPE-based transaction. But! What we can do, though, is to emulate the `OP_VAULT`-alike core behavior by re-designing the vault to consist of many keys holding being released per-used

* `OP_CSFS` ( https://github.com/bitcoin/bips/blob/eae7d9fc57a5f3a2b0c4853c127cb4d27377dbf3/bip-0348.md ). Since CSFS is a signature verification over stack inputs i.e. a pure arithmetic function. This way this BIP is a perfect candidate for deprecation because of being possible to be completely replaces by PIPEs-induced functionality.

## BitVM Enhancements

In this section we show how the PIPE primitive may be used to enhance the optimistic-verification flow of the BitVM protocol. We show that the cryptographic and on-chain components of classic BitVM such as garbled circuits (GC), label commitments, and zk-proofs of consistency map naturally to a single PIPE instance.

A BitVM computation is defined by a Boolean circuit for a predicate

$$
f : \{0,1\}^n \rightarrow \{0,1\}.
$$

Both BitVM and BitVM-PIPE involve the same set of actors, and the emulation works because their responsibilities align structurally.

* **Operator**. The operator claims knowledge of a witness $w$ such that $f(w) = 1$. The operator
  provides all necessary information to make it possible to generate a compact fraud proof in case if $f(w) = 0.$
* **Signers Committee (SC)**. Verifies the operator’s setup, publishes the information provided by
  the operator, and signs the on-chain transactions $AssertTx$ and $DisproveTx$.
  * The $AssertTx$ enforces the operator to start the protocol’s dispute period by publishing some additional information necessary for fraud proof construction.
  * The $DisproveTx$ allows anyone to publish a fraud proof in case if the operator cheats and $f(w) = 0$.
* **Challengers**. Anyone able to detect operator’s misbehavior during the dispute window and publish the $DisproveTx$ transaction containing a valid fraud proof.

In the BitVM case a fraud proof is equivalent to knowledge of the garbled circuit false-output label. To emulate this using PIPE, we reinterpret BitVM’s disprove condition as the PIPE underlying hard relation: $R= f(w) = 0 ∧ h= Hash(w)$.

Here how the GC-based BitVM flow and its PIPE emulation compares:

|**BitVM using GC**|**BitVM using PIPE**|
| --- | --- |
| Operator constructs a garbled circuit for *f* and sends to the SC: (1) garbled tables, (2) commitments to input labels and the false-output label, (3) a zk-proof of their consistency. | Operator knows a witness $w$ with $f(w) = 1$. He computes $h = Hash(w)$ and runs $PIPE.Setup(1λ, h) = ((h, ct, π), pk)$ for a hard relation $R$. Operator sends the setup output to the SC. 
| SC verifies the zk-proof ensuring that the garbling table and label commitments are consistent. SC then publishes the garbling table and the transac- tion templates $AssertTx$ and $DisproveTx$ off-chain, and signs both transactions. | SC checks whether $PIPE.OutVerify((h, ct, π), pk, h) = 1$. If so, SC publishes $AssertTx$, $DisproveTx$, and $((h, ct, π), pk)$ off-chain. | 
| Operator publishes $AssertTx$ including openings of the committed input and false-output labels. | Operator publishes $AssertTx$ including a witness *w*consistent with *h*. |
| If *f*(*w*) = 0, a challenger can compute the false- output label from the public GC data and publish $DisproveTx$. | If *f*(*w*) = 0, a challenger can compute a valid PIPE signature $σ = PIPE.WSign((h, ct, π), w, m)$. The signing algorithm succeeds because (*h*, *w*) ∈ *R*. The challenger publishes $DisproveTx$ containing σ, and the script verifies the Schnorr signature under *pk*. |
| If no valid $DisproveTx$ is posted during the dispute window, the optimistic path finalizes and the computation result is accepted. | If no valid $DisproveTx$ is posted during the dispute window, the optimistic path finalizes and the computation result is accepted. |

The emulation preserves the trust structure of GC-based BitVM protocols. We model the Signers Committee (SC) as a single logical party composed of *n* signers, and a permissionless set of challengers. Informally, the SC controls which spending conditions are put on-chain, while challengers enforce these conditions against a potentially dishonest operator.

The SC is implemented as an *n*-of-*n* multisig on Bitcoin: all n signers must cooperate to authorize $AssertTx$ and $DisproveTx$. This has two consequences:

* *Liveness* depends on all n signers being online and willing to sign; a single unresponsive or malicious signer can halt progress.
* *Soundness* depends on the SC as a whole behaving honestly when the setup is produced. If at least one signer is honest and follows the protocol, an invalid setup (e.g., wrong GC, wrong PIPE instance, or wrong script template) will not be signed and therefore will never appear on-chain. or wrong script template) will not be signed and therefore will never appear on-chain.

Correctness with respect to the claimed computation holds as long as at least one challenger is honest. Challengers are permissionless and untrusted; their only role is to detect operator dishonesty during the dispute window. In BitVM, this means reconstructing the false-output label from the public GC; in BitVM-PIPE, it means opening the PIPE instance when $f(w) = 0$. If the operator cheats and the on-chain script is correct, a single honest challenger suffices to trigger the $DisproveTx$ path.

Liveness in the optimistic case requires that all *n* SC members remain online and sign the required transactions; no liveness assumption is placed on challengers. Finality is reached once $AssertTx$ confirms and no valid dispute is posted before the timeout.

Under these assumptions, BitVM-PIPE inherits the same safety–liveness profile as GC-based BitVM: the SC is a fully trusted *n*-of-*n* committee that fixes the on-chain spending conditions, while a single honest challenger suffices to enforce these conditions against a dishonest operator.

One structural difference between GC-based BitVM and BitVM-PIPE concerns the timing at which the operator’s witness is fixed. In BitVM, the garbling, label-commitments, and the zk-consistency proof can be generated without knowing the actual input *w*; the setup phase is witness-agnostic. In BitVM-PIPE, by contrast, the PIPE instance includes the value $h = Hash(w)$, so the operator must know *w* at setup time. This distinction does not affect the optimistic-verification or dispute logic.

Beyond the structural correspondence, the on-chain footprint of BitVM-PIPE is smaller than that of GC-based BitVM.

The opening transaction $AssertTx$ becomes smaller. In GC-based BitVM, the script must verify per-bit commitments to the input labels, which induces two hash computations for every bit of the operator’s input $w$ together with the associated control opcodes. In BitVM-PIPE, the assertion transaction checks only the single hash value $h = Hash(w)$. If w is an object that fits into a single Bitcoin stack item (for example, a Groth16 proof), then the $AssertTx$ script becomes a very small constant-size program.

The dispute transaction $DisproveTx$ is also compact: it consists of a single signature check.

Finally, the cost of both the setup phase and its verification depends only on the underlying PIPE primitive, and therefore solely on the chosen WE construction. As a result, the entire BitVM-PIPE construction inherits its asymptotic and concrete efficiency directly from the underlying PIPE instance.

## Performance

A python script (approximately) modeling the Garuda-alike verifier expressed in AADP gives the following results (with $v$ denoting variable count and $g$ denoting gate count):

Substituting the ciphertext size formula $v(2g+ 1)^2 |\mathbb{F}|$, we get 338 TB size of the ciphertext.
Due to the cubic scaling of the ciphertext size, even marginal improvements in the circuit size translate into significant improvements in the ciphertext size.
Size of the ciphertext might possibly be decreased further - for example, by picking a better hash with a certification-friendly S-box or choosing a curve with smaller 2-adicity of $F^d$. Overall, we conclude that this approach to witness encryption, while very costly, is realistically implementable.

Computationally, the dominant cost arises from determinant computation. If properly parallelized (e.g., following the approach in https://arxiv.org/abs/1308.1536 ), the determinant can be computed within approximately one Bitcoin L1 block interval (\~10 minutes) using up to 50 CPU machines, each equipped with 256 CPU cores and approximately 7 TB of storage. Memory requirements are comparatively modest and do not constitute a primary bottleneck.

At current cloud services pricing, renting such infrastructure amounts to approximately $100-200 per covenant execution, plus a negligible on-chain cost for signature verification. This makes the approach economically viable for near–real-time covenant emulation. Moreover, these execution costs can likely be further reduced through batching techniques under development.

## Conclusion

Bitcoin PIPEs v2 present a cryptographic means of enforcing certain types of covenants on Bitcoin L1 without changing consensus rules and without additional trust mechanisms.

Cryptographic covenant enforcement occurs by releasing a signing key only when a witness to an NP statement is provided. This way a valid Schnorr signature can be produced only if the prescribed condition holds.

PIPEs v2 focused on witness encryption based on AADPs as a concrete and arithmetic-native construction. AADPs enable NP predicates particularly SNARK-verifiable statements to be expressed as explicit algebraic objects, allowing spending conditions to be enforced through determinant evaluation without any on-chain changes.

While AADP-based witness encryption is not presented as a final or optimized instantiation, it provides a fully specified and analyzable framework that makes the trade-offs of cryptographically enforced spending conditions explicit. By grounding PIPEs in a single, arithmetic-native witness encryption approach, PIPEs v2 clarifies how expressive off-chain logic can be enforced in practice using existing cryptographic tools.

Overall, PIPE v2 demonstrates that expressive, verifiable spending conditions can be achieved on Bitcoin purely through cryptography. Advancing this direction, by enhancing the WE scheme with alternative proof systems, iterating with performance optimizations, and applying PIPEs to vaults, bridges, and rollup architectures, offers a promising path toward richer functionality on Bitcoin.

## References

\[1\] *Mikhail Komarov, Bitcoin PIPEs,* https://www.allocinit.xyz/uploads/placeholder-bitcoin.pdf

\[2\] *Michel Abdalla, Brent Carmer, Muhammed El Gebali, Handan Kilinc-Alper, Mikhail Komarov, Yaroslav Rebenko, Lev Soukhanov, Erkan Tairi, Elena Tatuzova and Patrick Towa. Bitcoin PIPEs v2.* https://eprint.iacr.org/2026/186

\[3\] *Lev Soukhanov, Yaroslav Rebenko, Muhammad El Gebali, and Mikhail Komarov. Implementable*
*witness encryption from arithmetic affine determinant programs.* https://eprint.iacr.org/2026/175

-------------------------

AdamISZ | 2026-02-13 15:30:32 UTC | #2

Fascinating, thank you for publishing this.

I find myself very interested in one particular line of reasoning in this post:

[quote="nemothenoone, post:1, topic:2249"]
This fully characterizes the available design space. Any spending rule that works **without changing Bitcoin’s consensus rules** must ultimately boil down to a single question:

> Can a valid signature under this public key be produced or not?

There is no other hook available to enforce authorization.

The key insight behind PIPEs is to treat this constraint not as a limitation, but as the design surface.

[/quote]

On a first read through, I found this convincing: let’s consider how one can gate spending via any privileged knowledge status: we only have (a) signatures and (b) hashlocks, available in Script. Even the ECC equivalent of hashlocks is not available: pubkeys can only be matched against signatures, not private keys, in Script.

So the obvious problem with hashlocks is part of the sort of “Bitcoin 101” that we all looked at on the wiki back in the day - “locking” a utxo to the preimage of a sha2 hash is non-functional because once you publish the unlocking script anyone can read it and replace your destination address. Which seems to reinforce your point: a *signature* is the only way to authorize *a particular payment*.

But two counterpoints spring up:

* The “canonical” solution to the hashlock enforcement problem is, of course, seen in HTLC and similar: you *can* attach specific identities, *additional* to the hashlock: still a signature, but an additional “gate” that needn’t be related to the witness encryption. Also interesting in this context may be that, through a combination of taproot structures and plain old Script, you can make some kind of large OR condition (one of these pubkeys and the hashlock)
* Most interesting is that you are using witness encryption *not* to enforce authorization of specific transfers but to enable full control of the utxo. You call this “binary covenant” but I feel like the bitcoin research community has taken up the word “covenant” as specifically restricting the manner of spending - and that’s even setting aside @ajtowns imo very valid point that “covenant” is a terrible name for this. But meh, forget about names. Your system decrypting to the private key, not to a specific authorization, means that the distinction between the value of a hashlock, and the value of a signature, doesn’t apply, right? Is WE for hashlock preimages a possible alternative in the design space? (I have zero clue whether it would be easier, harder, or just not functional; I’m just reacting to the statement that the design space is *only* signatures).

-------------------------

cmp_ancp | 2026-02-13 18:06:17 UTC | #3

Using a schematic with a counterparty operator, if a tx is required to have a 3-3 musig2 signature, being one key in possession of the operator, one in possession of the user and the third hidden behind WE, we can actually enforce a covenant-like tx format.

The user couldn’t form an abitrary tx, because it requires the operator signature. Even if the operator is malicious during the setup comittee, he couldn’t steal funds because of the user signature.

-------------------------

AdamISZ | 2026-02-14 12:15:51 UTC | #4

[quote="cmp_ancp, post:3, topic:2249"]
Using a schematic with a counterparty operator, if a tx is required to have a 3-3 musig2 signature, being one key in possession of the operator, one in possession of the user and the third hidden behind WE, we can actually enforce a covenant-like tx format.

The user couldn’t form an abitrary tx, because it requires the operator signature. Even if the operator is malicious during the setup comittee, he couldn’t steal funds because of the user signature.

[/quote]

Assuming this is answering my quibble about “is covenant a good term for this?” above, then: well but any multisig setup with a third party could do the same, couldn’t it? If the WE gates access to the private key itself, then in itself it doesn’t do any covenant-like behaviour. The restriction on the tx structure is purely from the operator in this setup, isn’t it?

-------------------------

