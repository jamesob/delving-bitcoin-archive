# Comparing Bitcoin Covenant Proposals for Vaults

lillianwang | 2026-09-04 18:21:56 UTC | #1

Hello!

I am sharing a report analyzing simplified Bitcoin vault constructions using presigned transactions, CTV, APO/APOAS, TXHASH, CCV, and the CAT-based Purrfect Vault construction. Comparison factors include partial withdrawals, withdrawal address commitment timing, flexibility for in-transaction fee management, operational complexity, and on-chain cost. The report's main conclusions are that CTV is suited to simple vaults with precomputed outputs, CCV best supports vaults requiring partial withdrawals or trigger-time selection of a withdrawal address, and TXHASH enables greater commitment flexibility but places more responsibility on the vault designer. Finally, APOAS and OP_CAT may be more appealing if their broader non-vault applications are also valued.

The report was written with a broader technical audience in mind rather than specifically for vault or covenant experts. This work came from a semester-long project in fall 2025 mentored by Michael Maurer and Neha Narula. I was motivated to create it because vault discussions seemed scattered across BIPs, implementations, and forum posts. I hope this is useful for other work consolidating analyses of vault constructions. I'd appreciate any feedback or corrections, especially with my interpretations of the APOAS and CCV constructions, comparison methodology, and other opcode combinations or vault properties.

[Read the full report here](https://raw.githubusercontent.com/Skyler-Cloud/Bitcoin-Vault-Comparison/main/bitcoin-vault-comparison.pdf)

-------------------------

Anzus_GemWallet | 2026-09-08 16:10:01 UTC | #2

One addition that could help nontechnical readers is a short comparison of what each approach means for the user: can they make partial withdrawals, change the recovery destination, recover after losing a device, and handle unexpectedly high fees? That would make the differences easier to understand without needing to know how each proposal works internally.

-------------------------

