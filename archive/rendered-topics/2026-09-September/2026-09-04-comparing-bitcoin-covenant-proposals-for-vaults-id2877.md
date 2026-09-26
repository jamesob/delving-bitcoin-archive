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

PraneethG | 2026-09-17 21:49:57 UTC | #3

This is interesting work comparing Möser Eyal Sirer type vaults. I've done some similar academic work around them, with measurements and some vulnerabilities in the different constructions of these vaults. Would love to get some feedback.

https://research.praneethg.xyz/

-------------------------

askii21m | 2026-09-26 04:16:23 UTC | #4

Thanks for writing this up!

One thing on the APOAS section. You noted that APOAS signatures could be replayed against other UTXOs at the same vault address, and recommend unique keys per vault.

Replaying the signature in a separate transaction only unvaults the second deposit, which is harmless, but I think the half-spend problem is worth mentioning as well: that both deposits can be spent in the same transaction. Since APOAS commits to neither the input count nor the input index, a transaction with two vault inputs and a single fixed output verifies at both inputs. 200k sats in, 99k out, 101k to the miner.

BIP-119 gives this as the reason CTV commits to both: committing to the index "makes it safer to design wallet vault contracts without half-spend vulnerabilities".

Perhaps this is helpful: [https://covenants.diy/g/nhcg5hYdxO](https://covenants.diy/g/nhcg5hYdxO). I've wired up an APOAS vault and a CTV vault side by side with the same two deposits. The APOAS transaction is complete and verifies at both inputs (discarding half the payment); CTV refuses the second input because the index is in the hash.

The missing commitments do buy fee flexibility, but they create the half-spend exposure, so never reusing a vault address is a requirement rather than a recommendation

-------------------------

