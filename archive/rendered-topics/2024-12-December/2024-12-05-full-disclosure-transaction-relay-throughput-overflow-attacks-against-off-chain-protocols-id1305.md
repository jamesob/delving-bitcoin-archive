# Full Disclosure: “Transaction-Relay Throughput Overflow attacks against Off-Chain Protocols

ariard | 2024-12-05 17:55:07 UTC | #1

Today is fully disclosed a new transaction-relay jamming attack affecting bitcoin time-sensitive contracting protocols by exploiting the transaction selection, announcement and propagation mechanisms of base-layer full-nodes, that has been under embargo for a good year now.

The full report has been shared on the mailing list and it also available here:
https://ariard.github.io/

There is a CVE ID request pending on the MITRE, with the temporary identifier 1780258, that should be update accordingly in the future. One single CVE has been requested as this vulnerability originating from the transaction-relay rules and components, affecting an unbounded number of bitcoin use-cases, among others lightning.

-------------------------

ariard | 2024-12-09 15:11:22 UTC | #2

This is now tracked under CVE-2024-55563.

-------------------------

