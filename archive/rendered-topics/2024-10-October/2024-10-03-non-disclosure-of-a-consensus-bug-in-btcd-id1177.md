# Non-disclosure of a consensus bug in btcd

AntoineP | 2024-10-03 14:19:37 UTC | #1

Niklas Gögge and I reported a consensus bug in btcd to the project maintainer back in March of 2024. This bug was fixed in [btcd v0.24.2](https://github.com/btcsuite/btcd/releases/tag/v0.24.2) and the project awarded us a bounty for this finding. Although it has a minor impact on the network overall, this vulnerability is critical for btcd users: it allows an attacker to hard fork btcd nodes using a simple standard transaction (i.e. at virtually no cost).

We want to remind btcd users of the issue and **urge them to upgrade to the latest released version [v0.24.2](https://github.com/btcsuite/btcd/releases/tag/v0.24.2)**. We had agreed on a public disclosure date (23rd of September) with the maintainer but in the last two weeks we were repeatedly asked to delay the disclosure with increasingly tenuous justifications.

Transparency is important when handling security vulnerabilities. Not only for trust in the software itself but also in the release process: users deserve to know why they have been pressed to upgrade. Unless there is a good reason not to, the announced public disclosure schedule should be respected.

We considered his concerns, the most reasonable of which had to do with the number of vulnerable nodes. There are currently about [70k Bitcoin full nodes running on the network, of which about 60 are running btcd](https://luke.dashjr.org/programs/bitcoin/files/charts/software.html). From these 60 btcd nodes, 20 are still running known to be vulnerable versions (see [Niklas' previous btcd disclosure](https://delvingbitcoin.org/t/disclosure-btcd-consensus-bugs-due-to-usage-of-signed-transaction-version/455)). Of the remaining 40 nodes, 24 upgraded to the latest version with the fix for the new consensus bug discovered. This leaves only 16 vulnerable nodes which could be reasonably expected to upgrade.

Given that these 16 nodes only represent 0.022% of the Bitcoin network and we don’t know when or if they will be upgraded, we decided this was not sufficient ground for exceptionally delaying the public announcement. However, out of an abundance of caution we will refrain from disclosing the full details until October 10th. If you haven't already, **please upgrade** to [btcd v0.24.2](https://github.com/btcsuite/btcd/releases/tag/v0.24.2)!

-------------------------

Crypt-iQ | 2024-10-03 18:51:34 UTC | #2

(post deleted by author)

-------------------------

