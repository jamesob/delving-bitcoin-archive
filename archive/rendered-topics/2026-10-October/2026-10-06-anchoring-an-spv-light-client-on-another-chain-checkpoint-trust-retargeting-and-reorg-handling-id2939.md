# Anchoring an SPV light client on another chain: checkpoint trust, retargeting and reorg handling

sebas | 2026-10-06 16:32:13 UTC | #1

I work on Writz Protocol, an open-source lending protocol where BTC collateral stays on Bitcoin and loans are issued as USDC on Stellar. I would like critique of the Bitcoin-facing parts, because that is where I least want to be wrong. This is not an announcement; the code is on testnet and has not been audited.

**The collateral script**

Each deposit goes to a P2WSH output with this redeem script:

\`\`\`

OP_IF

    <protocol_pubkey> OP_CHECKSIGVERIFY

    <user_pubkey>     OP_CHECKSIG

OP_ELSE

    <cltv_value> OP_CHECKLOCKTIMEVERIFY OP_DROP

    <user_pubkey> OP_CHECKSIG

OP_ENDIF

\`\`\`

The first branch is a cooperative release after repayment: both keys sign. The second is the exit if the protocol disappears: the user alone, after an absolute height \`T\`. We default \`T\` to the confirming height plus 4,320 plus 1,008 blocks (about 37 days), and the lending contract only accepts a timelock between 1,008 and 105,000 blocks above the block that confirmed the deposit. A cooperative spend has a witness of about 257 bytes (72 + 71 + 1 + 114) and was 149 vbytes in a Signet test.

The protocol key is a single key in an HSM for now. Moving to a threshold scheme, and later to Taproot with MuSig2, is planned.

**The SPV client**

The contract that has to believe a Bitcoin deposit happened runs on a smart contract platform, so it cannot see Bitcoin. Our first design was stateless: the caller supplied a run of headers with each proof. We dropped it because a verifier that only checks each header against the \`bits\` in the same header can be fed a cheaply mined chain with a low difficulty.

The current design is an on-chain light client:

\- one trusted checkpoint (a real recent block), set once;

\- anyone may submit contiguous headers, each checked for proof of work, link to a stored parent, the exact \`bits\` required at that height (retargets at 2,016 blocks), and a timestamp rule;

\- cumulative chainwork is tracked and the heaviest chain is followed, with a bound on how many blocks one call may re-point;

\- a transaction proof is a Merkle path against a stored header with a minimum depth of 6, and 64-byte transactions are rejected to close the inner-node forgery.

**Where I would like review**

1\. *\*Checkpoint trust.\** The checkpoint is a single admin-set value. Is there a cleaner way to reduce that trust without storing a long header history, for example requiring a minimum cumulative work on top of it, or several independent attestations of the same block?

2\. *\*Depth.\** We use 6 confirmations for deposits, and a faster lane with 3 for small deposits is part of the design. For collateral that backs a loan, is a fixed depth the right shape, or should it scale with the amount?

3\. *\*Timelock type.\** We chose an absolute \`OP_CHECKLOCKTIMEVERIFY\` so that the exit date is fixed at deposit time. A relative \`OP_CHECKSEQUENCEVERIFY\` would start counting from confirmation instead. I would like to hear the case for either.

4\. *\*Release transaction fees.\** The release is built by the protocol and the user together, so the fee is fixed when it is signed. How would you handle fee bumping here: anchor outputs, CPFP from the user's change, or something else?

5\. *\*Header submission cost.\** Re-validating retargets on a metered platform is not free. Is there a cheaper way to check difficulty continuity that you would trust?

6\. *\*Reorgs.\** A heavier fork re-points the canonical chain up to a bounded number of blocks. How would you pick that bound?

Write-up with the full construction: https://writz.xyz/whitepaper . Note that it describes the earlier stateless design; the light client above is what the contract does now. Source: https://github.com/WritzProtocol/writz .

I would rather hear about a flaw now than later.

-------------------------

