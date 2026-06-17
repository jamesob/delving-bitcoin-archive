# Defining 0x50 0x00 as unstructured taproot annex data

0300dbdd1b | 2026-06-17 08:05:42 UTC | #1

Hi all,

I would like to reopen the discussion around unstructured data in the taproot annex, with a narrower scope than previous proposals.

This has been discussed before. BIP341 introduced the annex as a witness element starting with `0x50`. The annex, or the lack of annex, is covered by the signature, contributes to transaction weight, and is otherwise ignored by taproot validation. BIP341 also warns users not to include an annex until its meaning is defined, because a future soft fork could assign meaning to it.

Antoine Riard (@ariard) later proposed a structured annex format using TLV-like records. Joost Jager (@joostjager) then proposed making a limited form of unstructured annex standard, first on the mailing list and then in Bitcoin Core PR #27926. That proposal used annexes starting with `0x00` for unstructured data. The Core PR also included relay-policy rules: if one input committed to an annex, all inputs had to commit to an annex; empty annexes were allowed; and unstructured annex data was limited to 256 bytes. The PR was closed for inactivity.

My reading is that the discussion started from the idea of `0x00` as free-form annex data, but then moved into relay policy, standardness limits, multiparty concerns, TLV integration, and concrete use cases before the basic documentation question was settled.

A later direction in the thread was to make unstructured annex data standard only under restrictive policy rules. I am intentionally asking a smaller question here: whether the raw `0x50 0x00 <bytes>` form should be documented as the unstructured annex form, independent of relay policy.

I would like to separate those questions and focus only on the consensus-level interpretation of:

```text
0x50 0x00 <bytes>
```

where `0x50` is the existing annex marker, `0x00` marks the annex as unstructured, and the remaining bytes are left to the application that created them.

This is not a proposal to change consensus rules. The annex is already valid at consensus level. The proposal is only to define this form as unstructured annex data, so that future consensus changes do not assign another meaning to the same form.

The motivation is that the underlying incentive remains open. Witness bytes have a lower weight than non-witness bytes. This gives applications placing data on-chain a strong reason to use witness-based encodings.

I am not arguing that data embedding is desirable. The point is that the incentive exists anyway. If applications are economically pushed toward witness-based data encodings, it seems better to define a form that avoids the worse side effects of other constructions. The annex is relevant for that discussion because it is not script, it does not create UTXOs, it is signed by the spending input, it is counted in transaction weight, and BIP341 already specifies that it is otherwise ignored by taproot validation.

What is missing is a defined unstructured form. Without that, applications using the annex still have to worry that their data may conflict with some future consensus meaning.

The proposal is therefore:

```text
0x50 0x00 <bytes>
```

means unstructured taproot annex data, with no consensus interpretation for the bytes after `0x00`.

Future structured annex proposals should avoid giving another meaning to this same form. This should not conflict with the earlier structured-annex work; it only asks that structured encodings avoid the `0x50 0x00` prefix.

This post is intentionally not about relay policy. It does not propose making such transactions standard. It does not propose a default mempool size limit. It does not propose that default relay accept large annexes. It does not require all inputs to carry an annex. It also does not try to settle the multiparty annex-inflation concern, beyond acknowledging that it matters for policy and protocol design.

I am looking for feedback on the narrow consensus-level idea: whether `0x50 0x00 <bytes>` should be treated as the unstructured annex form, and whether there are reasons future consensus extensions would need to use that same form for something else.

If the feedback is positive, I will try to write this up as a short BIP.

## References

BIP341:
https://github.com/bitcoin/bips/blob/40cd7b4dd6dae2d9d38162b019ba588839331ba1/bip-0341.mediawiki

BIP141:
https://github.com/bitcoin/bips/blob/40cd7b4dd6dae2d9d38162b019ba588839331ba1/bip-0141.mediawiki

Taproot Annex Format proposal:
https://github.com/bitcoin/bips/pull/1381

Standardisation of an unstructured taproot annex:
https://gnusha.org/pi/bitcoindev/CAJBJmV-L4FusaMNV=_7L39QFDKnPKK_Z1QE6YU-wp2ZLjc=RrQ@mail.gmail.com/

Bitcoin Core PR #27926:
https://github.com/bitcoin/bitcoin/pull/27926

Ord issue mentioning the need for an unstructured annex carve-out:
https://github.com/ordinals/ord/issues/2405

Best,
LesLie (0300dbdd1b)

-------------------------

