# How CTV+CSFS improves BitVM bridges

RobinLinus | 2025-04-10 02:30:33 UTC | #1

**TL;DR:** CTV fundamentally improves BitVM bridges by eliminating the need for a presigning committee and the existential honesty assumption for safety. Additionally, CSFS drastically improves capital efficiency and reduces setup overhead.

### Introduction
In BitVM, we use input-committing covenants to represent state. E.g., we express "the operator can withdraw funds from the bridge if no challenge occurred" with a specific output that challengers may burn to indicate that they believe the operator's withdrawal request was malicious, forcing the operator to go through the challenge process. In contrast, in the happy path, if that output "survives" the challenge period then the operator can use it to execute the withdrawal transaction.


The current BitVM bridge design relies on emulated covenants to guarantee safety of the deposits. This committee requires a 1-of-n trust assumption and introduces various undesirable complexities in practice. Up until recently we had assumed that CTV is not sufficient to replace the committee because CTV is designed to commit only to _outputs_ but not to _inputs_. However, it turned out that there is a trick that enables the functionality that we need.

### The scriptSig Trick
The key idea is to use the fact that CTV commits to the scriptSig of all inputs.
Say we want to express "inputA is spendable only together with inputB". 
1. Define inputB to be a (legacy) P2SH output. 
2. Presign a signature using sighash `SINGLE|NONE`, effectively signing only inputB. This signature commits to inputB and since P2SH is not SegWit the signature will be in inputB's scriptSig.
3. Define inputA to be a P2TR output and contain a CTV condition with a template hash that commits to the scriptSigs, including the signature for inputB. 

The result: **inputA** commits to the signature for **inputB**, which itself commits to **inputB**. So **inputA** becomes spendable only in conjunction with **inputB**.

### Details
These commitments are one-way: **inputA** commits to **inputB**, but not vice versa. Two-way commitment would ideally be desirable but is impossible due to hash cycles (even with `TXHASH` or introspection opcodes).

For our bridge it would be sufficient to constrain **inputB** to always be burned. But that too is unachievable with CTV—its template hash would have to commit to its own `scriptSig`, which already commits to **inputB**, and therefore its TXID—creating a hash cycle.

To work around this, we redesigned the transaction graph such that the contract remains secure even if **inputB** is spent independently of **inputA**. The exact mechanics of that redesign are beyond the scope of this post, as they require a deeper dive into the bridge contract architecture.


### Concrete Improvements

- **CTV enables unconditional safety of deposits.** Even if **all** operators are malicious, they cannot steal from the bridge. While we still rely on an **existential liveness assumption** to ensure withdrawals, the simplified setup allows us to scale the operator set significantly—potentially by an order of magnitude. This makes it far more likely that at least one operator remains live.

- **CSFS replaces Lamport signatures**, reducing transaction sizes by approximately **10x**. This dramatically lowers the capital cost required for bridge operations.

- **Non-hardened key derivation** becomes possible with CSFS, enabling anyone to compute the operator's public keys **non-interactively**. This greatly simplifies the peg-in process by reducing the amount of data operators need to provide.

- In the current design, **peg-ins still require an operator signature**, which allows for censorship. While this can be mitigated in the side system, our long-term goal is to **eliminate this interaction entirely** by modifying the bridge contract to support fully non-interactive peg-ins.

For these reasons, the BitVM Alliance strongly supports the [CTV + CSFS](https://delvingbitcoin.org/t/ctv-csfs-can-we-reach-consensus-on-a-first-step-towards-covenants/1509/11) proposal.

### Conclusion
This trick demonstrates that with a clever use of CTV and CSFS, we can dramatically simplify BitVM bridge construction while improving both safety and efficiency. By removing the need for a presigning committee and eliminating the existential honesty assumption, CTV enables bridges with unconditional safety guarantees. At the same time, CSFS reduces capital costs and operational complexity, paving the way for more scalable and decentralized bridges. While a few open challenges remain—such as making peg-ins fully non-interactive—this design represents a significant step forward in practical, trust-minimized Bitcoin interoperability.

-------------------------

Cyimon | 2025-04-10 14:28:23 UTC | #4

![image|689x411](upload://8BdEnPVfhewYeSZ8YaYIS99GlRD.png)
I tried to integrate OP_CTV in BitVM2 Bridge based on your idea. And this is the graph. Are there any mistakes?

-------------------------

ekrembal | 2025-04-10 16:09:12 UTC | #5

![bitvm_bridge_opctv|640x500](upload://nHkN9JnYCqgwQrk6JFN0zkvuZuz.jpeg)

Here is a transaction graph that supports efficient collateral usage for operators:  
- Operators provide signatures for the red arrows at the time of deposit.  
- Every kickoff must be finalized before the operator sends the **ready-to-reimburse transaction**.  
- A kickoff can get finalized with either **challenge timeout transaction** or **disprove timeout transaction**.
- Kickoffs can proceed in parallel, allowing many withdrawals to be processed with a single collateral.  
- If one of the kickoffs is disproven, all reimbursements stop.

Current Limitations:  
- A Bitcoin light client is needed to prove the inclusion of the payout transaction and the sidesystem's state.  (We are still working on this, but this also might add trust-minimized assumptions for safety)
- All operators' signatures need to be published somewhere for data availability, and Bitcoin is not ideal for this due to limited block space.

-------------------------

RobinLinus | 2025-04-11 17:37:20 UTC | #6

It seems I can’t edit my post anymore, but there’s an error:

[quote="RobinLinus, post:1, topic:1591"]
Presign a signature using sighash `SINGLE|NONE`
[/quote]

It should say:

[quote="RobinLinus, post:1, topic:1591"]
Presign a signature using sighash `ANYONECANPAY|NONE`
[/quote]

-------------------------

1440000bytes | 2025-04-15 00:00:02 UTC | #7

Added the link in https://en.bitcoin.it/wiki/Covenants_Uses

-------------------------

