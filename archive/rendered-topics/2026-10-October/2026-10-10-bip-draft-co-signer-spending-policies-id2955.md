# [BIP Draft] Co-signer Spending Policies

lukecarriere | 2026-10-10 04:46:04 UTC | #1

I'd like feedback on a draft that standardizes the policy a co-signer enforces, not the co-signer itself.

A co-signer that signs only when a predicate holds is a known way to emulate covenants today. Towns described the multisig version of this in 2021. Recent work (Halseth's blinded co-signers, Chain Code Delegation, predicate blind signatures, bonded co-signers) and products such as Liquidium and Lendasat each define their own policy. A wallet built for one cannot use a co-signer or oracle built for another.

The draft is one policy format for that. It covers open verification, blinded verification, and enclave verification, and conditions taken from the transaction or from external attestations. Co-signers must accept DLC oracle attestations, so existing oracles can be evidence sources unchanged. Anything script can enforce is required to be in script. Every policy keeps a spending path that needs no co-signer.

BIP Draft: https://github.com/lukecarriere/bip-cosigner-spending-policies/blob/468fdd1b75647432ed56c3a2b5579cbcfedaec74/bip-cosigner-spending-policies.md

I am not requesting a BIP number. I want to know whether this is the right layer to standardize, and comments on the open questions in the draft: canonical JSON or TLV; whether blinded verification can be specified now; whether the statement attestation format belongs here or in the DLC specifications; whether the output expression language is the right size; and whether the fallback path must be timelocked.

Luke Carriere

-------------------------

PREDGE | 2026-10-22 21:01:05 UTC | #2

On statement-v0: the signed tuple is event ID, outcome and timestamp, and nothing says whether the outcome can still change. For escrow delivery and other events settled by people, attestors often publish a first answer and correct it later. A co-signer that sees a valid statement within max age would sign on the first one. Either add a finality flag to the signed tuple and let evidence conditions require it, or say in the escrow example that the template should wait out a dispute window. On JSON vs TLV: whichever wins, the policy ID needs test vectors for the exact bytes, canonical JSON is where implementations tend to drift.

-------------------------

tehiyadu | 2026-10-27 15:12:46 UTC | #3

The policy hash should commit to the exact verifier semantics, not merely a parseable document. In particular, define whether unknown fields are rejected, how duplicate keys are handled, and whether defaults are explicit; otherwise two implementations could accept the same bytes but enforce different conditions. This is separate from choosing JSON or TLV. I’d also domain-separate the policy commitment from statement and transaction hashes, and include the format version in its preimage.

-------------------------

lukecarriere | 2026-10-28 14:37:19 UTC | #4

Thanks, agreed. The draft defines the policy ID as the BIP340 tagged hash (tag `CSP/policy`) of the canonical serialization, but it doesn't pin down what the verifier does with that document. I'll add these rules, independent of JSON vs TLV:

- Unknown fields make the policy invalid, and a co-signer MUST refuse to sign under it. Extensions go through new profile versions, not ignored fields.
- Duplicate keys make the policy invalid.
- There are no implicit defaults. Every field is serialized explicitly, so the hashed bytes are exactly the conditions enforced.
- The format version is the first field in the preimage, and the tag becomes `CSP/policy/v0` to match `CSP/statement/v0`.

Policy and statement commitments already use separate tags. I'll say explicitly that every CSP tag is distinct from the BIP340/341 tags, and I'll add test vectors that include rejected documents (unknown field, duplicate key, missing field), alongside the exact-bytes vectors @PREDGE asked for.

-------------------------

lukecarriere | 2026-10-28 14:38:05 UTC | #5

Thanks, good catch. As written, `statement-v0` signs event ID, outcome and timestamp, and says nothing about whether the outcome can still change, so a co-signer could act on a first answer that's later corrected. I'll add a finality flag to the signed tuple and let evidence conditions require it. The escrow example will require a final statement, and it will note that a template can wait out a dispute window instead when an attestor never marks finality. Attestations that use DLC nonce pre-commitment are already effectively final, since signing two outcomes for one event reveals the attestor's key. Agreed on test vectors for the exact policy ID bytes. They'll ship with whichever serialization wins.

-------------------------

