# Improving the Security of lArk OOR Channels with Equivocation Bonds

ademan | 2026-08-18 17:28:42 UTC | #1

# Improving the Security of lArk OOR Channels with Equivocation Bonds

This post describes a bond which improves the economic security of one-time, out-of-round assignments of Ark VTXOs.
These assignments are intended to enable instant opening of small-value just-in-time Lightning channels with inbound liquidity for the user.
The bond works by making equivocation in the VTXO assignment provable on-chain using `OP_CHECKSIGFROMSTACK`.
Each assignment is authorized by a signature over the BIP-341 sighash of the assignment transaction, so conflicting authorization signatures provide exactly the evidence needed for an equivocation proof.
Using `OP_CHECKSIGFROMSTACK` to prove equivocation in an off-chain protocol may also be useful beyond this application.

## Background

LSPs face significant challenges in efficient capital allocation for channel liquidity.
If the LSP allocates too much liquidity, then they risk wasting liquidity better allocated elsewhere.
If they allocate too little, they may pay additional on-chain fees if their new channel partner exhausts the initial liquidity.
Onboarding new users receiving small amounts adds another challenge.
New users receiving small amounts are prone to abandoning their channels, leaving the LSP to pay more on-chain fees to close the channel and recover their capital.
In high-fee environments, small-value channels may not be economical to recover at all.

lArk is a modified version of Ark focused on providing LSP-like services with greater on-chain efficiency.
Ark provides a very useful environment to LSPs where VTXO operations like splices can be performed off chain, and batching channel opens reduces the on-chain footprint.

For end users, receiving initial Lightning payments is a significant impediment to onboarding into the ecosystem.
For LSPs, onboarding requires pricing the risk of channel abandonment into JIT channel support, increasing fees and capital requirements.
Ark servers can instead preallocate small-value VTXOs in their transaction trees which they can assign out-of-round (OOR) to open channels just-in-time.
This enables instant channel opening without additional on-chain cost for the Ark server, and abandoned channels are automatically swept with the rest of the tree!

## Problem

However, OOR VTXO reassignment has unfavorable sovereignty and regulatory implications.
The sovereignty tradeoff may be acceptable to most users for small amounts, but unilateral control over funds creates regulatory risk for LSPs operating this way.
VTXO OOR transfers are secure under the assumption that the Ark server and the VTXO holder do not collude, but in the proposed scheme the initial VTXO holder and the Ark server are the same entity.
The Ark server could reassign these VTXOs arbitrarily many times, opening LN channels not backed by real Bitcoin.

## Solution (Using CSFS)

I propose that Ark servers post an on-chain "equivocation bond" which can be slashed by providing an "equivocation proof".
The Ark server preallocates VTXOs `V_0`, `V_1`, ..., `V_n` to use in this scheme.
Each VTXO `V_0`, `V_1`, ..., `V_n` must be assignable by producing a signature with the respective key `K_0`, `K_1`, ..., `K_n`.
Every VTXO spending path for `V_i` must require a signature with `K_i` which fully commits to the spending transaction.
Every key `K_i` in `K_0`, `K_1`, ..., `K_n` is chosen unilaterally by the Ark server, but must be globally unique and single-use to prevent spurious bond slashing.
To assign a preallocated VTXO to Alice, Alice and the Ark server agree on an assignment transaction funding a Lightning channel between them.
The Ark server signs the assignment transaction with key `K_i`, producing a signature over a message `M_a`.
If the Ark server signs a conflicting assignment transaction assigning the same VTXO to a JIT channel with Bob, this produces a signature with `K_i` over a second message `M_b`.

The equivocation bond penalizes this by permitting slashing of the bond using the equivocation proof.
The equivocation proof consists of a proof that the key `K_i` is covered by the bond, and two valid signatures from `K_i` over two different messages `M_a` and `M_b`.

During assignment, the Ark server provides a signature `sig(B, K_i)` which is used as proof that `K_i` is covered by the bond.
During slashing, `sig(B, K_i)` is validated against `B` which is committed to in the bond output's slashing spend path.
The assignment itself requires the Ark server to sign an assignment transaction with `K_i` and provide the signed transaction to the client.
This means any two signatures from `K_i` over two different assignment transactions, together with a bond coverage proof, constitute a sufficient equivocation proof.

If the Ark server signs a transaction assigning `V_i` to Alice, it produces `sig_a = sig(K_i, M_a)`.
`M_a` must commit to the entire assignment transaction such that no conflicting assignment can be made valid without producing another signature with `K_i` over a distinct message.
If the Ark server then signs a conflicting transaction assigning `V_i` to Bob, it produces `sig_b = sig(K_i, M_b)`.
These two signatures and their respective messages prove that `K_i` has equivocated, and `sig_delegate = sig(B, K_i)` proves that key `K_i` is covered by the bond.
The bond can then be slashed by providing `sig_delegate`, `K_i`, `sig_a`, `sig_b`, `M_a`, and `M_b` when all of these are true: `valid_sig(B, sig_delegate, K_i)`, `valid_sig(K_i, sig_a, M_a)`, `valid_sig(K_i, sig_b, M_b)`, and `M_a != M_b`.

The client must verify the assignment transaction and signature, and that every spending path requires a signature with `K_i` which commits to the entire spending transaction.
The client must also verify that the bond output is confirmed, unspent, and sufficiently funded, that it cannot be reclaimed by the Ark server during its active lifetime, and that it can be slashed with the described equivocation proof.
The client will not be able to verify with certainty the number of outstanding assignments, since the Ark server may produce secret assignments.
Nor can it verify with certainty how many VTXOs are covered by the bond.
The client should require a certain ratio between the advertised total VTXO value covered and the amount forfeited if the bond is slashed.

## Soft Fork Assumptions

This scheme relies on the activation of an opcode for checking signatures over arbitrary messages and a simple covenant which restricts the outputs on a spending transaction.
`OP_CHECKSIGFROMSTACK` satisfies the signature checking requirement, which is needed for both the equivocation proof and authorized bond slashing bounties.
Either `OP_CHECKTEMPLATEVERIFY` or `OP_TEMPLATEHASH` satisfies the simple covenant requirement needed for all but the most naive bond-slashing schemes.
The virtual transaction tree also requires an irrevocable commitment to the tree transactions, preferably from `OP_CHECKTEMPLATEVERIFY` or `OP_TEMPLATEHASH`.
Because at creation time the Ark server is the only known party, the preallocated transaction tree cannot be spendable by an n-of-n of the involved parties.
An external committee could still be used to emulate the transaction commitment, but this requires assuming that at least one of the committee members deletes their key.

## Additional Considerations

### VTXO Outputs, Assignment Keys, and Signatures
`OP_CHECKSIGFROMSTACK` is required both for the bond coverage proof and for validating `sig_a` and `sig_b`.
`B` and `K_i` must therefore both be keys compatible with `OP_CHECKSIGFROMSTACK`.
`sig_a` and `sig_b` must be signatures compatible with `OP_CHECKSIGFROMSTACK`.
At the time of writing, this requires `sig(B, K_i)`, `sig_a`, and `sig_b` to be BIP-340 Schnorr signatures.
Keys `B` and `K_i` must both be 32-byte x-only public keys.
`K_i` will satisfy the requirement if preallocated VTXOs are Taproot outputs with no script paths and with `K_i` as the output key.
To ensure the signatures commit to the assignment transaction's inputs and outputs, `M_a` in `sig_a = sig(K_i, M_a)` should be a BIP-341 sighash using `SIGHASH_DEFAULT`.

### Assignment Transaction Fees
Since any two valid signatures from the same key `K_i` over two distinct messages enable slashing, assignment transactions should use CPFP for fees or another method that does not require signing a modified transaction.

### Tree Structure

During the lifetime of the preallocated VTXOs, it must not be possible to spend the VTXO's ancestors in the transaction tree in any other way besides unrolling the tree.
From the on-chain UTXO through the tree, except for the leaf VTXOs, outputs cannot be spendable by the Ark server until after the UTXO times out.
The simplest way to do this is to put these preallocated VTXOs into a UTXO separate from normal Ark where levels in the tree are spendable only by the Ark server after the timeout, or by exactly the tree unrolling transaction (via CTV or TH).
For the sake of simplicity, the Ark tree and preallocated tree could also share a single on-chain UTXO, but this complicates normal Ark usage slightly, and makes the tree one level deeper in emergency exits.

### Bond Lifetime
It is desirable for the bond to expire periodically and be replaced for continued operation.
The bond can be periodically cycled by having a timelocked "recovery" path that enables the Ark server to reclaim the bond after its expiry.
Before the bond's expiry, the only available spending path must be the economically unattractive bond slashing; clients must reject bonds that permit early spending.
New bonds MUST use new bond keys `B` to isolate the new bond from old delegations and historical key compromise.
The bond's lifetime should cover not only the lifetime of the VTXOs but accommodate delays in equivocation detection and slashing transaction confirmation.

### Bond Value

Because a bond can only be slashed once, sizing the bond appropriately is important to produce the desired disincentive.
The Ark server always has the capability to produce its own equivocation proof and can attempt to recover the bond bounty; therefore, the forfeited amount is the important parameter.
Because of this, the bond cannot provide a perfect economic deterrent, but it is still a useful disincentive.
The more the Ark server equivocates, the more half-proofs exist and the greater the probability of slashing.
The Ark server's ability to reassign the same VTXO arbitrarily many times makes proactive detection important (see below).

To equivocate profitably, the Ark server must equivocate the VTXO with real victims, or obtain some benefit with colluding third parties willing to accept the risk of an unbacked channel.
When the Ark server equivocates a VTXO, even with a colluding third party, they produce a half-proof.
Any honest half-proof for the same VTXO is enough for a colluder to defect and slash.
A colluder might also use a second identity and obtain a second half-proof themselves, giving them a complete equivocation proof and the ability to slash the Ark server's bond.
This risk should discourage the Ark server from equivocating, even with parties willing to accept the equivocation.
Instead, it's more beneficial for the Ark server and potential colluders to use an entirely unbacked channel if it's acceptable to the third party.

### Detection

If users publish their VTXO assignment signatures and messages on some public channel, they can proactively detect conflicting VTXO assignments.
A user could still collude with the Ark server to conceal a conflicting VTXO assignment, but attempting to claim the VTXO on-chain would reveal that user's half of the equivocation proof.
Another user holding their own half of an equivocation proof for a conflicting assignment from the same key `K_i` could then slash the bond.

A full transparency log scheme is worth investigating, and would likely improve on the detection mechanism proposed in my scheme, but for brevity I'm proposing something considerably simpler.
Half-proofs are published on multiple third-party Nostr relays, with a filterable key corresponding to the VTXO's `K_i`.
Relay selection is important but out of scope.
Half-proofs might optionally be encrypted using client-verifiable key material deterministically bound to `K_i` and shared between the Ark server and VTXO recipients.
This would offer some privacy from relay observers without preventing equivocated users from discovering each other's half-proofs.
Clients must receive both `sig(B, K_i)` and their VTXO assignment signature from the Ark server, and check their relays for valid conflicting half-proofs.
They then publish their own half-proof, and proceed with channel setup, but they do not consider any funds from the VTXO to be received until after a period elapses with no conflicting half-proof published.
This extra wait reduces the likelihood of a race, in case an Ark server equivocates with two users who publish their half-proofs simultaneously.
The wait must be long enough for a matching half-proof to reach the client.
This should be on the order of hundreds of milliseconds, but a delay of a few seconds provides a considerable buffer without harming user experience.
Since slashing of the bond removes the Ark server's incentive against equivocation, clients must also watch for spending of the bond for the lifetime of their VTXO assignment and exit if it is spent.

## Slashing Specifics

The equivocation proof described above is sufficient to slash the bond in many different ways to disincentivize Ark server misbehavior, but the specific slashing mechanics are secondary to the equivocation proof and open to discussion.
Importantly, if the slashing procedure pays a bounty, the Ark server can always produce a proof which enables it to attempt to claim the bounty.
In my opinion, paying a small bounty to incentivize reporting is helpful, but the bounty must not be too generous.

### Naive Slashing with Bounty

The most obvious penalty scheme is for the equivocation proof to enable spending the bond output, allowing the prover to claim it as a bounty.
Without any other authentication, miners can always steal the output themselves.
Therefore, the naive bounty is undesirable.

### Authorized Bounty Claims

The naive bounty can be improved upon by only permitting certain keys to spend the bond output and claim it as a bounty.
A straightforward protocol can be built to authorize a particular key `A` to claim a bounty that involves a particular assignment message `M_a`, the message that the Ark server signed with `K_i` to assign the VTXO to Alice.
To do this the Ark server establishes a delegation key `D` which is very similar to `B` but is only used to delegate.
Both `B` and `D` are committed to in the bond output, which the client should verify.
The Ark server signs an ephemeral key `X` with `D`, delegating to `X`.
The Ark server then uses `X` *only* to sign `M_a` and `A`, thereby authorizing `A` to claim the bounty using an equivocation proof involving `M_a`.
`X` should never be reused to sign for more than one pair of message and key, or else the wrong user may be able to claim the bounty.
The client cannot verify `X` uniqueness, but the Ark server can already forge additional authorizations itself, so equivocation here does not expand its ability to illegitimately claim a bounty.
Note that the bounty is meant to incentivize slashing; it cannot be expected to always reimburse the correct user, and so that is not a design goal.
The bond output is made spendable by key `A` using the same equivocation proof as above, plus something like `sig(D, X)`, `sig(X, M_a)`, and `sig(X, A)`, which prove key `A`'s authority to spend it. 
The claimant can then provide a normal `OP_CHECKSIG` transaction signature for `A` authorizing a transaction spending the bounty.
Alice must regard VTXO assignment as incomplete until she receives `sig(D, X)`, `sig(X, M_a)`, and `sig(X, A)`. 
Like in the equivocation proof, validating these signatures requires `OP_CHECKSIGFROMSTACK`.
`OP_CHECKSIGFROMSTACK` currently requires `D` and `X` to be 32-byte x-only keys, and `sig(D, X)`, `sig(X, M_a)` and `sig(X, A)` to be BIP-340 Schnorr signatures.
After authorization of the key `A`, an `OP_CHECKSIG` will validate a signature from `A` over the bounty claiming transaction.
`OP_CHECKSIG` in turn requires `A` to be a 32-byte x-only key, and the signature from `A` to be a BIP-340 signature.

The bounty authorization can be substantially simplified using [`OP_PAIRCOMMIT`](https://github.com/bitcoin/bips/blob/857a7debc6625a3dadbaecee1ee7b2ed5e8ada75/bip-0442.md), where the Ark server only provides `D` and `sig(D, pc(M_a, A))` to the user.

This authenticated bounty claiming mechanism prevents miners or other observers from claiming the bounty merely by observing the equivocation proof.
It does not, however, prevent the Ark server from creating new signatures to claim the bounty and recover the value of its bond.
The next section, "Forcing an Economic Loss", proposes a scheme to make the Ark server expect a loss even if they recover the bounty.

### Forcing an Economic Loss

To prevent an Ark server from recovering the entire bond amount through the bounty, at least some of the bond should be forfeited by the Ark server and not claimable in the bounty.
Using CTV or TEMPLATEHASH to commit to the slashing transaction, the slashing transaction may forfeit some of the bond as fees by specifying its total output value to be less than the input amount from the bond.
Forfeiting to fees is a milder disincentive, since the Ark server might collude with miners to recover some of that value, but assuming the Ark server is not able to collude with all miners, the Ark server still expects a loss on average.

A stronger disincentive can be accomplished by truly burning some of the funds with an unspendable output with a non-zero value.
In my opinion, some proportion should serve as a bounty to an authorized claimant, and some proportion should be forfeit.
In any case, the exact type and proportion of funds to forfeit is open to discussion.

### Tentative Combined Scheme (Using CTV or TH)

My recommended scheme combines forfeiting part of the bond to fees with authorized bounties.

The bond commits to the slashing transaction using CTV or TEMPLATEHASH, spending *most* of the bond to fees, with an additional bounty output.
The bounty output is spendable only by someone who can provide an authorized key and an equivocation proof associated with that key.
Though successfully claiming the bounty is not guaranteed, users are still incentivized to slash the bond to claim the bounty, which should cause the Ark server an expected loss, subject to Ark server and miner collusion discussed above.

One could potentially have two bounty outputs, but enforcing that they are claimed by different parties is somewhat complicated and I haven't come up with a way to do it without major concessions to the basic soundness of the bond slashing.

## Related Work

### Keer et al. (2026) [Ark: Offchain Transaction Batching in Bitcoin](https://arxiv.org/abs/2605.20952v1)

In addition to formally defining Ark, Keer et al. propose a "Fast Finality" protocol in Section 4 which is directly comparable to the `OP_CHECKSIGFROMSTACK` scheme I propose here.
Due to the lack of certain opcodes, they replaced this approach with a BitVM scheme in [version 2](https://arxiv.org/pdf/2605.20952) of their paper.
Despite being superseded, version 1 of their paper contains the closest functional protocol to mine, and therefore is the object of comparison below.
[My first public draft](https://gist.github.com/Ademan/8366e808b80f0562f698bf633d7b0af5/revisions#diff-6eb5aa2af122281945d8d42cc285a04af319c927fec0b2e03ce47987c39985a2) predates their publication by 8 days; the similarity in approaches appears to reflect convergent thinking and common dissatisfaction with the trust requirements of out-of-round transfers.

Keer et al. v1 propose to punish an operator, the Ark server, for equivocation by slashing a bond.
To emulate `OP_CHECKTEMPLATEVERIFY` functionality, an existentially honest committee pre-signs the bond slashing transaction and deletes their keys.
So long as at least one signer is honest, the slashing transaction is fixed.
This is the same mechanism described under "Soft Fork Assumptions" for building the transaction tree, now applied to the bond slashing transaction.
They fix the nonce of signatures to a static value per-VTXO.
If the Ark server produces two signatures under the same key spending the same VTXO in different transactions, the private key may be extracted to slash the bond.
The fixed-nonce scheme permits slashing of the bond with only an ordinary signature, and avoids the need for key delegation.
Keer et al. v1 claim compatibility with existing Bitcoin, but as they note in [v2 of their paper](https://arxiv.org/pdf/2605.20952), their fixed-nonce enforcement script is not compatible with current Bitcoin script.
Remark 4.4 of Keer et al. v2 also notes the utility of `OP_CAT` for this purpose.
My scheme instead uses a larger equivocation proof consisting of two messages, two signatures, and a delegation signature which are checked using `OP_CHECKSIGFROMSTACK`, an opcode also not presently available in Bitcoin script.

Keer et al.'s key-recovery scheme has an important drawback in that exposure of the operator's private key enables forging of operator signatures.
Once the operator's secret key is known, the unique nonce for a VTXO can be recovered from any single signature for that VTXO.
These forgeries can potentially invalidate other "fast finality" transactions unrelated to the equivocation, making other outstanding fast finality transactions vulnerable to pre-emption by previous owners of their input VTXOs.
My scheme does not leak key material enabling signature forging, but it still undergoes a similar degradation in integrity after the bond is slashed: the Ark operator loses the economic incentive against further equivocation.

While the BitVM scheme in [Version 2](https://arxiv.org/pdf/2605.20952) can be implemented on Bitcoin today, fast finality is only available to existing members of the fast finality group.
Adding a member requires the user to have collateral already committed to the fast finality protocol, and requires setting up a new BitVM instance with the updated group.
Neither is compatible with onboarding a new JIT user.

### Ruffing et al. (2015), [Liar, Liar, Coins on Fire!](https://dl.acm.org/doi/10.1145/2810103.2813686)

Ruffing et al. present a cryptographic scheme using chameleon hashes to enable parties to extract a private key from equivocated assertions to spend coins locked as a bond.
Ruffing's proposal has the substantial advantage of not requiring any soft forks.
Ruffing et al. also anticipated approaches similar to mine.
My proposal instead relies on `OP_CHECKSIGFROMSTACK`, trading the use of chameleon hashes, which are fairly unusual in Bitcoin, for soft-fork requirements.
Ruffing et al. also give a more thorough treatment to the economics of these schemes.
Using Ruffing's work to adapt this proposal to the current Bitcoin network is potentially interesting, though Keer et al. note in v2 that Bitcoin cannot require an accountable assertion to be included on chain, which leaves a substantial enforcement gap.
My proposed scheme closes that enforcement gap by making the authorization signature part of the equivocation proof; there is no separate assertion to include on chain.

### Timeout Trees and Ark

Timeout Trees have obvious commonalities with both this scheme and Ark.
Timeout Trees, Ark, and my scheme all use a single UTXO to commit to a tree of many off-chain VTXOs.
In Ark and Timeout Trees, the VTXOs are jointly owned from creation by the end user and the creator of the tree.
However, trustless ownership of VTXOs is only available to the parties and allocations established when the tree is created.
Ark also allows OOR transfers to parties not belonging to the original Ark tree, but this comes with a marked increase in trust requirements.
The CSFS equivocation bonds proposed here attempt to mitigate the risk of accepting the out-of-round assignment of preallocated VTXOs.

### BitVM

BitVM is a broad family of protocols enabling trust-minimized computation enforced on-chain, often using slashable bonds to incentivize correct operator behavior.
This resembles the slashing functionality proposed here, but is substantially more general and not inherently bound to a particular transaction authorization.
Keer et al. v2 *apply* BitVM2 to enforcing non-equivocation of VTXO authorization signatures.
The limitations of this approach to JIT onboarding are discussed above.

## Open Questions

1. **Bond Sizing.**
  The ability for Ark servers to equivocate the same VTXO arbitrarily many times significantly complicates choosing an economically effective disincentive.
  In practice, under only modestly adversarial conditions, I suspect a small bond may suffice, but the worst case can be huge.
2. **Slashing Incentives.**
  Does the proposed bond bounty sufficiently incentivize slashing?
  Under all scenarios, such as high fees?
3. **Relay Selection.**
  Victims must share at least one relay in order to detect equivocation, and agreeing on shared relays is already a difficult problem in Nostr.
  Trusting the Ark server to define the relay selection is dangerous because they may propose relays which censor half-proofs.
  Choosing arbitrary relays may prevent victims from seeing each other's half-proofs.
4. **Transparency Log.**
  An alternative transparency log system might improve the reliability of detecting half-proofs and/or equivocation.
  Transparency log non-equivocation can also be enforced using `OP_CHECKSIGFROMSTACK`, though it's not enough to make a secure system.
5. **Chameleon Hash.**
  Can this scheme be implemented on the existing Bitcoin network without any soft forks, using a different chameleon hash that allows accountable assertions to be checked on-chain?
  I suspect not, but it would be a very useful discovery.

-------------------------

GeorgeTsagk | 2026-08-19 08:41:45 UTC | #2

Hi @ademan 

Doesn't this secure all OOR vtxos? they don't have to fund an OOR channel necessarily

-------------------------

