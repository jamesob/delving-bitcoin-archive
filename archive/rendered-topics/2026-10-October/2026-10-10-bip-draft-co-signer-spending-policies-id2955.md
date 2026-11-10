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

lukecarriere | 2026-11-08 21:29:44 UTC | #6

For cross-reference, my original October 5 bitcoindev post is here:
https://groups.google.com/g/bitcoindev/c/yoL5VVMO4Qg

Draft v2.1 is now published:
https://github.com/lukecarriere/bip-cosigner-spending-policies/commit/c21ed45c1eb78a562962d17a5fdaa86c9d7befb3

Thanks @tehiyadu and @PREDGE. This incorporates strict verifier semantics, versioned commitments and signed finality, plus the k-of-n correction, citation fixes and header updates. The complete schema, exact-byte vectors and reference implementation remain unfinished. It is still a draft seeking criticism; comments here or on the mailing list are welcome.

-------------------------

lukecarriere | 2026-11-08 21:40:02 UTC | #7

Since the draft still leaves the serialization open, here's a summary of the options before I write the exact-byte vectors and reference implementation. @PREDGE, you raised this earlier, so I'd especially value your view.

**TLV (BOLT 1 / DLC style)**
- One valid encoding per policy, so duplicates and reordering fail by construction
- Same building blocks as DLC oracle messages and Lightning, so evidence drops in unchanged
- Not human-readable without a decoder; implementers reusing Lightning code must disable the "ignore unknown odd types" rule, since v2.1 rejects all unknown fields

**Canonical JSON (RFC 8785)**
- Readable, diffable, supported everywhere
- Common parsers silently accept duplicate keys (last one wins), which v2.1 must reject; large integers and Unicode normalization need extra rules
- Nostr hashes JSON and still sees event-ID mismatches across implementations

**CBOR (RFC 8949 deterministic encoding)**
- Compact, standardized, already used for airgapped wallet QR transfer
- A third format DLC, Lightning and PSBT don't use for these objects; duplicate-key handling varies by library

**Hybrid**
- The policy ID is computed only over strict TLV (version first, tag `CSP/policy/v0`); a JSON view maps 1:1 to the TLV fields for authoring and review but is never hashed or signed
- Costs two formats to specify and test

I currently lean toward the hybrid, but nothing is decided. Which would you choose, and is there a failure mode I'm missing? Comments from anyone implementing co-signers or DLC oracles are welcome too.

-------------------------

lukecarriere | 2026-11-10 17:30:05 UTC | #8

**The seven pillars of an open lending protocol to replace rehypothecating lenders**
Celsius and BlockFi failed because they pooled customer coins, held the keys and lent them out again. The open version keeps every loan one-to-one, never takes custody of the collateral and never lends it out again.

1. **Wallet:** the borrower keeps their own keys and can always recover the collateral, using standard PSBTs and descriptors.
2. **Offer board:** an open place to post loan offers and negotiate privately, such as Nostr listings.
3. **Price feed:** signed price data from existing DLC oracles, used to check the loan-to-value ratio and trigger a liquidation.
4. **Payment attestor:** signed proof that a repayment really happened, on Lightning or on-chain.
*5. **Co-signer:** releases or liquidates the collateral only according to the agreed policy. This is the part the BIP standardizes.*
6. **Alert service:** warns both parties about margin calls and deadlines, but can never move funds.
7. **Agreement template:** loan terms readable by both people and software, tied to the exact policy ID.

-------------------------

lukecarriere | 2026-11-10 17:31:13 UTC | #9

**What becomes possible once the protocol works across lenders:**

- **Portability:** you can move a loan or refinance with another lender without selling your Bitcoin or giving up control of it.
- **Competition:** lenders compete on rates, and none of them gets to hoard customer coins.
- **Mix and match:** any wallet can work with any lender, oracle or co-signer, so no single company becomes a point of failure.
- **Auditability:** anyone can check a policy before funding a loan, which replaces "trust us" with rules you can verify.
- **Lower barrier for new lenders:** a new lender can plug into existing infrastructure instead of building all seven pieces.
- **Beyond lending:** the same standard works for vaults, escrow, inheritance plans and spending caps for AI agents.
- **Honest caveat:** this doesn't make co-signers trustless and doesn't settle the legal questions, which still need counsel. It shrinks and standardizes the trust you have to place in them.

-------------------------

lukecarriere | 2026-11-10 17:33:03 UTC | #10

**How Co-signer Spending Policies works, as drafted**

- **The setup:** the coins sit in a Taproot output with several ways to spend them. Each way can need the participants' keys, a co-signer group's signature, conditions enforced in script such as timelocks, or some combination.
- **The policy:** one portable document, with its own ID, that lists exactly which outputs the co-signers may sign for, under what conditions, and what evidence they need first.
- **Script first:** anything Bitcoin script can enforce, it enforces. The co-signers handle only what script can't, like a price or a repayment.
- **The co-signer group:** either an all-must-sign MuSig2 key or a k-of-n threshold. In a k-of-n group, an invalid spend is blocked when at least n−k+1 co-signers refuse to sign it.
- **Evidence:** attestors sign facts, such as a BTC price or that a payment was made. Every co-signer must accept existing DLC oracle attestations, and there's an optional statement format that carries a finality flag.
- **Signing:** in open-v0 mode, the co-signer sees the policy, the PSBT and the evidence. It checks the policy ID, makes sure every output matches a template, and checks that the evidence is recent, signed by enough attestors and final where the policy demands it. Then it either signs or refuses, and gives every participant a record of the decision.
- **The escape hatch:** there's always a path that needs no co-signer, usually a long timelock, so funds can't be frozen forever if the co-signers vanish.
- **Loan example:** the collateral is released to the borrower once repayment is attested, or liquidated if the price or a default condition is met. The borrower and lender can also settle together, and the borrower has a long-timelock recovery path.
- **What's not in it:** no consensus change, no blinded or enclave verification yet (those are planned), and it doesn't make co-signers trustless. It makes their rules standard and open to inspection.

-------------------------

lukecarriere | 2026-11-10 17:34:14 UTC | #11

**The core ask: criticism before code**

I'm asking people to break it now, so that whatever survives is something everyone can share rather than something I own.
- **The first question:** is the policy the right layer to standardize, or should the effort go somewhere else? (This is the question you asked on Delving.)
- **Criticism of the design:** what's missing, what's ambiguous, and where two co-signers could read the same policy differently.
- **Open decisions you want input on:**
  - JSON or TLV.
  - Whether the escape hatch must always be timelocked.
  - Whether the statement format belongs in the DLC specs instead.
  - How big the output expression language should be.
  - When to specify the blinded and enclave modes.
- **What you're not asking for:** a BIP number, endorsements, or adopters. It's an unnumbered discussion draft on purpose.

**How to avoid becoming just another proprietary policy format**

- **Developed in the open:** it's public and MIT-licensed, and it's on bitcoindev, Delving and GitHub before any code exists.
- **Standardizes the policy, not the product:** any wallet, co-signer or oracle can implement it. You aren't selling the co-signer, and lending is just one application profile on top.
- **Builds on existing standards:** Taproot, Miniscript, descriptors, PSBTs, MuSig2 and DLC oracles, with no new consensus rules and no new oracle network.
- **Gives ownership away where it fits:** you're offering the statement format to the DLC spec maintainers instead of keeping it.
- **Makes it testable:** exact-byte test vectors and documents that must be rejected, so anyone can prove their implementation is compatible without trusting yours.
- **Real success test:** two independent implementations agree on every policy ID and every accept or reject decision. Until then, it isn't a standard.
- **Draws on prior art:** it credits Towns, Halseth, Rubin, BIP 89 and others, and positions itself as the common format for their ideas, not a competitor.
- **Invites existing players:** ask Liquidium, Lendasat, Casa, Unchained, Nunchuk and others what it would take for them to adopt it. Don't assume they will.

-------------------------

lukecarriere | 2026-11-10 17:34:47 UTC | #12

**What's the problem?**

- Bitcoin co-signers (in lenders, vaults, escrow) each enforce their own rulebook, so there's no portability, nothing to audit, and you're locked into one vendor.
- Celsius and BlockFi showed the cost: pooled custody, coins lent out again, and "trust us" instead of rules anyone can check.

**What's the solution?**

- Co-signer Spending Policies is one open, portable format for what a co-signer may sign, when, and on what evidence.
- It standardizes the policy, not the co-signer, and it needs no soft fork.

**Why does it work?**

- Script enforces what it can. Co-signers enforce only the rest, like a price or a repayment.
- There's always a path that needs no co-signer, so funds are never stuck forever.
- It reuses what already exists: Taproot, Miniscript, PSBTs, MuSig2 and DLC oracles.
- Exact test vectors let anyone prove compatibility, so no one owns the standard.

**How does it work?**

- Coins sit in a Taproot output with co-signer, script and fallback paths.
- A policy with its own ID defines the allowed outputs and the evidence each one needs, such as signed price or repayment attestations, recency and finality.
- The co-signer checks the PSBT and the evidence against the policy, then signs or refuses and gives everyone a record of the decision.
- For example, a loan's collateral is released on a repayment attestation or liquidated on a price condition, and the borrower can still recover it after a long timelock.

**What's the ask?**

- Criticism before code: is the policy the right layer to standardize, and where does the design break?
- Input on the open questions: JSON or TLV, the fallback timelock, and where the statement format should live.
- Implementers to test it. It only becomes a standard once two independent implementations agree.

-------------------------

