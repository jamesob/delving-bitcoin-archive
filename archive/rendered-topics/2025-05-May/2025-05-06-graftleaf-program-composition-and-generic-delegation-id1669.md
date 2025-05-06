# Graftleaf: Program Composition and Generic Delegation

josh | 2025-05-06 15:04:57 UTC | #1

_TLDR: I'm exploring an idea to enable generalized program composition and coin delegation, which seems to strike a nice balance between simplicity and flexibility._

I’ve been thinking recently about the optimal way to introduce delegation to Bitcoin. The idea of delegating one’s coins is not a new one. Some [speculate](https://bitcoinops.org/en/topics/op_codeseparator/) that `OP_CODESEPARATOR` was an early attempt by Satoshi to do delegation. More recently, proposals like [Graftroot](https://gnusha.org/pi/bitcoindev/CAAS2fgSnfd++94+40vnSRxQfi9fk8N6+2-DbjVpssHxFvYveFQ@mail.gmail.com/) and [Entroot](https://gist.github.com/sipa/ca1502f8465d0d5032d9dd2465f32603) have been put forward, which would enable delegation (and re-delegation), but only from key path spends and only to a locking script. In contrast, [CSFS](https://github.com/bitcoin/bips/blob/master/bip-0348.md) would enable delegation from script path spends, but it's limited, as it only facilitates delegation within a pre-committed script.

When considering delegation, what type of functionality is desirable? I’d argue that generalized delegation has two properties:
1. Delegation _from any valid spending path_
2. Delegation _to arbitrarily complex compositions of programs and script_

The latter is important so that users can delegate to _addresses_, and not just public keys, while retaining the ability to add timelocks and other conditions.

In order to build this, we need two key features: 1) "Generalized Composition," and 2) "Generalized Delegation." Generalized composition involves the conjunction of programs and script, and delegation is merely the addition of a second composition on-the-fly, if the primary one is satisfied.

Is there a simple and safe way to implement generalized composition and delegation within Taproot? I think there is...

I’d like to present a proof-of-concept that I came up with, which I'm calling "Graftleaf."

## Proposal: Graftleaf

Graftleaf is a new Taproot leaf version (0xc2), which uses the annex to perform delegations. Graftleaf adds two significant capabilities:

1. **Composition**: The ability to execute zero, one, or more witness programs in sequence, including a locking script.
4. **Delegation**: The ability to add additional spending conditions at signature-time, which can comprise any combination of programs and script.

The core execution logic is fewer than 60 lines of code. You can find my implementation [here](https://github.com/joshdoman/bitcoin/pull/1). I’ve included a fairly large test suite, though there’s definitely room for improvement.

Below are some technical details on how it works. I wrap up with a more in-depth comparison versus existing proposals and a brief discussion of some possible use cases. I’d appreciate feedback, and I hope this work is of interest to the community.

## Technical Overview

This implementation creates a new tapleaf version (0xc2) and gives consensus meaning to the annex when this leaf version is used:

### Composition Mechanism

- Multiple witness programs can be executed in sequence
- Each program has its own section of the witness stack, defined by explicit size elements
- Programs can be different witness versions (v1+) or script
- Resource limits (stack size, validation weight) are preserved and enforced across executions
- Signatures commit to the chain of tapleafs and annexes in the execution path to prevent witness malleability and replay attacks
- If the annex is absent, empty, or begins with 0x00, all elements on the stack must be consumed

### Delegation Mechanism

- Users can commit to additional spending conditions at signature-time using the annex
- Any valid composition of programs and script can be delegated to
- Delegation is indicated by an annex beginning with the tag 0x01
- Annexes starting with the tag 0x01 are forbidden unless signed to prevent witness malleability
- After delegated execution completes, all elements on the stack must be consumed

## Technical Deep Dive

### Composition

A composed program consists of multiple subprograms that execute in sequence. Each subprogram operates on its own portion of the witness stack, defined by a size element that indicates how many witness elements belong to the program.

The witness stack for a composed program looks like:

```
[program1 witness elements]
[program2 witness elements]
...
[programN witness elements]
[size element for programN]
...
[size element for program2]
[size element for program1]
[tapleaf]
[control block]
```

Each size element encodes the number of stack items for its corresponding program and is at most two bytes. Execution begins with the first program and continues through each program in sequence.

The tapleaf encodes the list of programs as `[witness version][program length][program]`. If the witness version is 0xc0, all subsequent bytes are interpreted as script. Witness version 0 is not allowed as it is not compatible with tapscript.

Tapleaf Example 1 (a series of programs composed with script):

```
[program1 witness version]
[program1 program length]
[program1]
…
[programN witness version]
[programN program length]
[programN]
[0xc0]
[script]
```

Tapleaf Example 2 (a program composed of only script):

```
[0xc0]
[script]
```

If the program is empty or the annex is absent, empty, or begins with 0x00, the stack must be empty post-execution for validation to succeed. 

### Delegation

Delegation is implemented via the annex. When a non-empty annex is present with the leaf version 0xc2 and the annex begins with the tag 0x01, the remainder of the annex is interpreted as a composed program to be executed after the primary program completes.

The witness stack for delegation looks like:

```
[primary program witness elements]
[delegated program witness elements]
[size elements for delegated program]
[size elements for primary program]
[tapleaf]
[control block]
[annex] (0x01 [delegated composed witness program])
```

The primary program executes first according to the composition rules. If successful and a signed annex exists with the 0x01 tag, the remaining stack elements are passed to the delegated witness program. When delegated execution completes, the stack must be empty for validation to succeed.

Annex tags 0x02+ are left as an upgrade path for future modes of delegation and uses of the annex.

### Resource Limitations

The validation weight budget is tracked and decremented across all compositions and delegations, ensuring that programs cannot bypass global resource limits.

Likewise, the global stack size limit is enforced by requiring both the total stack size and the `EvalScript` stack size to be at most `MAX_STACK_SIZE / 2` after a 0xc2 tapleaf is encountered. This conservatively ensures that the total stack size remains at most `MAX_STACK_SIZE`.

### Security Considerations

Replay attacks and witness malleability are the primary security concerns. To prevent replay attacks, signatures commit to the full chain of tapleafs and annexes in the execution path. This provides protection in the event that there are multiple possible execution paths that the user may sign.   

In addition, to prevent witness malleability, execution fails if a delegating annex is not signed. The annex can be signed directly, or it can be signed by a child program, which must commit to the full chain of annexes in the execution path.

## Potential Benefits

This proposal has several potential benefits, including:

1. **Backward-compatibility**: Works with existing P2TR addresses.
2. **Enhanced composability**: Enables "vault-of-vaults" and other complex spending policies, compatible with new witness programs adopted in the future (ex: P2QRH).
5. **Generalized delegation**: Allows users to authorize additional spending conditions at signature-time, delegating to any combination of addresses and script.
6. **Re-delegation**: Allows users to re-delegate to other users, adding additional restrictions to the structure of the final transaction.
7. **Improved privacy and fungibility**: Could increase P2TR adoption by encouraging users to adopt wallets and vaults that can be composed with and delegated to.

Additional benefits may include:

1. **Delegation-only opcodes**: Creates framework to allow math and introspection opcodes (ex: `OP_CAT`) to be committed to in an annex but not in a scriptPubKey. Such opcodes could then be safely used during delegation without enabling AMMs or state-carrying recursive covenants.
2. **Compatibility with future upgrades**: Provides framework for future witness programs and uses of the annex.

## Use Cases

### Vault-of-Vaults
Ex: Alice, Bob, and Carol create a 2-of-3 vault by combining their P2TR addresses, rather than their public keys. Each address corresponds to a personal vault that Alice, Bob, or Carol controls. This can be a single sig, a 2-of-3 multisig, a decaying multisig, or something more complex. Only two addresses must be satisfied in order to unlock the funds. If desired, additional spending paths can be added, such as a time-delayed backup, accessible by a single party.

This vault-of-vaults is easy to audit and assemble, and parties can selectively reveal their spending paths to the other parties, or not reveal any except those that are used.

### Receiver-Paid Payments
Ex: Alice pays Bob without committing to the fee rate of the transaction. Alice simply delegates a portion of her balance to Bob, sending herself the residual using SINGLE|ANYONECANPAY. Bob can then complete the transaction and choose the appropriate fee rate.

This basic pattern could be used to support more complex payment flows in the future.

### Bitcoin Markets
Ex: With a single signature, Bob can make an offer to sell a UTXO-linked asset from cold storage to one of many possible approved buyers, in exchange for bitcoin. Using timelocks, Bob could even give some buyers a right-of-first-refusal.

Likewise, if an introspection opcode is enabled, Bob could offer bitcoin to many possible recipients, in exchange for some onchain or offchain asset tied to a UTXO. The ability to commit to a specific UTXO is important because it provides a built-in mechanism to cancel the offer, if the asset is transferred to someone else.

## A Path to Key Path Delegation

Graftleaf lays the groundwork for delegation from script path *and* key path spends using the annex. Unfortunately, BIP341 Taproot does not appear to have an upgrade mechanism for key path spends, since a key path spend is only possible with a witness stack that has only one element after removing the (optional) annex. This can be resolved with a new witness program, but it would require a new address type, which history shows can take a long time for industry to adopt.

## Comparison with Other Proposals

This proposal shares similarities with several other proposals:

- **Graftroot**: Allows delegation but only from key path spends and only once
- **CSFS**: Complementary feature for re-binding signatures to pre-committed scripts
- **G’root**: Complementary feature that uses Pederson commitments to add script commitments to key path spends
- **Entroot**: Combines ideas from Graftroot and G’root, replacing Pederson commitments with a hash-to-curve scheme, but only facilitates delegations from key path spends and compositions between a key path spend and script.

Pieter Wuille’s Entroot is perhaps the closest cousin to this proposal. The primary difference is that this proposal enables:
1. Script path delegation
2. Delegation from a composition of programs and script
3. Arbitrarily complex compositions on a single leaf

In addition, this proposal does not require hash-to-curve or a new address type. The core composition and delegation logic is upgradeable and reusable, if new programs with hash-to-curve or key path delegation are added in the future.

## Concluding Thoughts

I have taken care to consider possible edge cases and respect resource limitations, but if there is anything I have missed, I'd welcome the community's feedback. Naturally, soft fork proposals are highly contentious, but I believe that generic composition and delegation is an important architectural problem for Bitcoin to solve, provided there’s a way to do so safely with acceptable tradeoffs.

If there’s interest in a BIP, please let me know.

**Special thanks to Alex Lewin for reviewing early drafts of this post*

-------------------------

