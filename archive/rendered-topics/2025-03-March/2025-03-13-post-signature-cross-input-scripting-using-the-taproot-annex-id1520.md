# Post-Signature Cross-Input Scripting Using the Taproot Annex

josh | 2025-03-14 12:15:17 UTC | #1

*This post is not an endorsement of CTV or any specific covenant proposal. It should also not be confused with cross-input signature aggregation, which is a separate proposal.*

Hi all, I’d like to present an interesting idea for a cross-input scripting capability, which users can commit to *when signing*. I haven’t seen other mentions of this idea, but if they exist, I’d love to learn more!

Here is the scenario. Alice wants to create a SINGLE|ANYONECANPAY PSBT offer from her existing P2TR address, but when making her signature, she wishes to commit to additional spending criteria. This might be a timelock, a signature from another user, a CTV template, etc.

This is not possible for Alice to do today, at least not in a single transaction.

I recently learned about the taproot annex, and I believe that *signature-time* subscripting could be the ideal use case for it. Here’s how it might work:

1. The first byte is a protocol tag (e.g., 0x00).
2. The remaining bytes are TLV encoded and represent either:

* **Public subscripts:** a standard `script_pubkey`.
* **Redeem subscripts:** Begins with the input index, followed by the index of the public subscript, followed by the redeem script itself.
* **Uninterpreted data:** Functions like an `OP_RETURN` and carries arbitrary data.

Subscripts are evaluated using the same script interpreter as normal scripts. To maintain backward compatibility, subscripts are only executed if the regular script succeeds. Subscript evaluation succeeds if each `script_pubkey` is satisfied by at least one corresponding redeem subscript, or satisfies itself.

### Why might this benefit Bitcoin?

This functionality introduces on-the-fly programmability to Bitcoin script. Subscripts essentially represent an optional script commitment, which can be evaluated *post-signature* to enforce further constraints against the input.

This can be used to create on-the-fly delegated signatures (e.g. Alice signs a partial transaction, but it’s only valid if Bob co-signs through an annex-encoded redeem script).

This could also be used for post-signature timelocks (e.g. Alice signs a transaction that is only valid after $X$ blocks).

Another interesting use case is post-signature transaction templating, if covenants are one day introduced (e.g., Alice signs a partial transaction committing to any one of thousands of transaction templates).

On this point, an unexpected side benefit is that opcodes may be enabled exclusively within a subscript that would otherwise enable recursive covenants (e.g., Alice may commit to a hash of the spent outpoints or the legacy txid when signing her PSBT, but she would not be able to permanently encumber the resulting outputs).

### Request for feedback

1. Implementing this would require a soft fork. Does the community see value in this type of functionality? Is there interest in giving consensus meaning to the taproot annex?
2. Have there been previous proposals or discussions of post-signature cross-input scripting?
3. Could this proposal introduce security vulnerabilities or DDoS risks in Bitcoin Core?
4. Is there a better way to structure the annex to enable this functionality?

Thank you for your time! Looking forward to hearing the community's thoughts and insights.

-------------------------

harding | 2025-04-02 08:05:02 UTC | #2

I found this post a bit confusing because it seems to be proposing two orthogonal things:

1. A signer satisfying existing script conditions but adding new conditions that must be satisfied by other signers
2. Additional introspection

For point 1 (adding conditions), I think you get that with pretty much any delegation feature (e.g., OP_CSFS, graftroot/g'root, BitVM-style Script-based Lamport sigs).  I don't think there's any benefit in those cases to putting the additional script operations in the annex rather than on the regular witness stack.

For point 2 (additional introspection opcodes), this is heavily discussed.  The novelty in your proposal is potentially only allowing certain types of introspection during delegation rather than original script creation.  Do you have any examples of how that might be useful?  (I don't understand your example about "permanently encumber[ing] the resulting outputs".)

-------------------------

josh | 2025-04-03 22:22:17 UTC | #3

@harding Thank you for the feedback! I'll focus on your second point about introspection, as that is perhaps more interesting. I'll come back to your first point later when I've had more time to gather my thoughts.

> For point 2 (additional introspection opcodes), this is heavily discussed. The novelty in your proposal is potentially only allowing certain types of introspection during delegation rather than original script creation. Do you have any examples of how that might be useful?

Sure! Full introspection during "delegation" would be useful for non-interactive single-transaction protocols where a user needs to commit to many possible transaction templates with a single signature. This is particularly valuable if we want two-sided bitcoin-native markets to develop without enabling recursive covenants.

### The bidding problem

Let's suppose that there exists some fungible asset $A$ held across $N$ UTXOs, transferrable according to some metaprotocol. Ideally, this would represent equity in some bitcoin-aligned public company, but for now, let's assume it is a token that already exists, like a rune.

Suppose that a user wishes to make an offer to buy $X_A$ of asset $A$ for $b$ bitcoin, which anyone can accept if they own a UTXO with sufficient $A$. Today, the bidder would need to produce separately signed PSBTs for every offer they wish to make, using sighash ALL. This is impractical without a hot wallet, if there are thousands of UTXOs they need to bid on.

For this reason, non-interactive bitcoin markets are one-sided. If we want to see two-sided markets without trusted escrow agents, we need introspection.

### Why introspection helps

Suppose that an introspection opcode exists. Let's put delegation / subscripting aside for the moment and assume this opcode can be used anywhere. As we'll see, introspection would make non-interactive buy offers practical, by using a single signature to commit to thousands of possible transaction templates.

Here's how it might work:

1. The bidder creates a staging transaction $T_0$ spending to a taproot output, which anyone can spend by satisfying any one of thousands of possible locking scripts.
2. Using an introspection opcode, each locking script commits to a single transaction template, spending from a UTXO holding $A$. The template sends $X_A$ to the bidder's output and lets the seller take $b$ bitcoin from the bidder's input.
3. The bidder broadcasts $T_0$ alongside the data necessary for anyone to derive the locking scripts.
4. A seller accepts the offer by signing a second transaction $T_1$, spending from $T_0$ by revealing the locking script it satisfies.

### Avoiding recursive covenants

The approach outlined above has two problems:

1. It requires two transactions, rather than one.

2. It uses functionality that would enable recursive covenants.

Any opcode that can 1) encumber a UTXO, and 2) enable *full* introspection (including of the spent outpoints) can enable recursive covenants. On the other hand, **an opcode that cannot encumber a UTXO cannot enable multi-transaction covenants.**

A locking subscript merely adds spending conditions to a transaction input after it has been signed. It is impossible for a UTXO to commit to these conditions ahead of time. Therefore, we can safely turn on introspection during "delegation" / subscripting without enabling recursive covenants.

In summary, "delegated" introspection would add meaningful expressivity to PSBT offers and facilitate the creation of non-interactive two-sided bitcoin-native markets, without enabling recursive covenants.

-----

### Follow up

I thought I might add a few additional "minor" use cases that would benefit from introspection during delegation / subscripting:

1. *Non-interactive multi-input offers:* If Alice wishes to make an offer to sell exactly $X_A$ held across two or more of her UTXOs, which anyone can accept, she needs to make two transactions today (one to consolidate her balances, and one to make the offer). If she can introspect, however, through delegation / locking subscripts, each input can commit to the presence of the other, and she can make her offer in a single transaction.

2. *Non-interactive multi-output offers:* I understand that proposals have been made to add new types of sighashes, like SIGHASH_GROUP. Enabling introspection during delegation / subscripting could provide a highly generic alternative.

-----

I hope that's helpful! Let me know if any of what I said would benefit from clarification.

-------------------------

