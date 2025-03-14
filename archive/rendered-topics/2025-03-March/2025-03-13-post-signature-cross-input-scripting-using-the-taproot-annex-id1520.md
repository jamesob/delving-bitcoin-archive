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

