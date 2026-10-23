# A Novel Post-Quantum Approach: "Proof-Of-Decay" (Cryptographic Half-Life) for Unmigrated Dormant UTXOs

bitKingAE | 2026-10-23 09:20:18 UTC | #1

**### Context & Problem Statement**

As the Bitcoin community discusses how to protect the network from future quantum attacks, we are largely stuck between two bad options for old, exposed-key outputs (like early P2PK or long-dormant wallets):

 **1. \*\*A Hard Freeze:\*\*** Declare unmigrated coins dead after a deadline and lock them forever. (Criticized for violating property rights and punishing lost keys/heirs).

**2. \*\*Doing Nothing:\*\*** Leave them vulnerable, risking a catastrophic market-wide supply dump if a quantum hacker cracks the keys.

 Both choices treat old coins as an "all-or-nothing" jackpot for whoever claims them first—either the hacker gets everything, or nobody gets anything.

 **### The New Idea: "Proof-of-Decay" (Cryptographic Half-Life)**

 Instead of artificially throttling transactions or hard-freezing balances, what if the protocol introduces a \*\*cryptographic decay function\*\* tied to block height for any output that stays unmigrated past a future quantum transition deadline?

 Think of it like radioactive decay: if an exposed UTXO is not moved to a quantum-safe address before the grace period ends, its balance begins to automatically shrink over time at a predictable, mathematical rate.

 **#### How it would work:**

**1. \*\*The Grace Period:\*\*** The network sets a multi-year deadline for everyone to upgrade their keys to post-quantum safe formats.

**2. \*\*The Decay Clock:\*\*** Once the deadline passes, any legacy/exposed UTXO that remains untouched begins a slow, programmatic "decay" (e.g., losing a small percentage of its total satoshis every 50,000 blocks / roughly once a year).

**3. \*\*Recycling the Decayed Satoshis:\*\*** The satoshis that "decay" away are automatically burned or redirected into the global coinbase reward pool, supporting the Proof-of-Work miners securing the network.

 **### Why This Changes the Game Theory**

 **\* \*\*Destroys the Hacker's Incentive:\*\*** If a quantum hacker spends millions of dollars and computing power to crack an old, abandoned wallet (like a multi-year-old dormant whale or Satoshi-era coins), they realize a harsh economic truth: \*\*the longer they wait, the less Bitcoin is actually left in the wallet.\*\*

**\* \*\*Eliminates the Market Shock:\*\*** Because the supply is slowly and predictably shrinking or trickling back into active circulation over a long period, there is never a sudden, market-crashing supply shock.

**\* \*\*Fairness for Lost Keys / Heirs:\*\*** If a legitimate owner or their family recovers their lost keys years later, their wallet won't have 100% of its original balance, but it won't be entirely wiped out either. They can still claim whatever remains, completely bypassing the need for a total coin freeze.

\-

**### Questions for the Community:**

 I’d love to hear developers' and cryptographers' thoughts on an approach like this:

1\. Could a mathematical decay rate be enforced at the consensus script level without bloating the UTXO state too severely?

2\. Does shifting from "absolute ownership preservation" to "time-based economic decay" offer a better game-theoretic defense against quantum adversaries than a hard freeze?

 -

Looking forward to your feedback on whether a decay/half-life model could work as a creative middle ground!

-------------------------

