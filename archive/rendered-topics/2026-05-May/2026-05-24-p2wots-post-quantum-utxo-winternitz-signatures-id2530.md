# P2WOTS: Post Quantum UTXO Winternitz Signatures

opus-lux | 2026-05-24 23:08:14 UTC | #1

Hello Bitcoin Community!

This is the revised proposal for post quantum bitcoin using lightweight Winternitz signatures.

I would like to thank Murch for correctly pointing out the flaws in my original proposal.

Read the revised proposal essay along with the source code at the following links below!

Proposal:  block_opuslux.ar.io/#/bitcoin-proposal

Code: https://github.com/opus-lux/bitcoin-proposal

WOTS-39 essay:  block_opuslux.ar.io/#/essay

Let's start by explaining what was wrong with my original proposal and why Winternitz + Lamport works on the EVM but not for Bitcoin! I originally developed WOTS-39 on the EVM which has a fundamentally different blockchain structure than Bitcoin. Since Ethereum based chains are account based you can easily and cheaply use the Lamport authorization chain to verify the upload of each Winternitz public key for each message signature. Ethereum is built for smart contract execution so tracking this data is trivial on the ethereum virtual machine. Bitcoin is fundamentally different than Ethereum. Bitcoin treats each UTXO as an individual object so your coins are not as easily tracked under a single account address, you query the chain to see which UTXO's you can actually spend to get your full Bitcoin balance. So we have to take a fundamentally different approach to solve Winternitz signatures on Bitcoin because we must solve it at the UTXO level instead of the account object level. The good thing is that the core discovery of WOTS-39 still works without the account object. The core insight of WOTS-39 is that by using the message TXID as a unique anchor you can reliably use Winternitz signatures without ever repeating a public key. So the new construction for my proposal introduces a new native bitcoin output type using witness version 2. Each time the user wants to receive post quantum bitcoin they generate a new receive address, they generate a fresh wots_pk which gets included in their receive address and allows the sender to set the scriptPubKey. The scriptPubKey controls who can spend the UTXO. Since P2WOTS is an additive new output type all existing taproot and multi-sig functionality remains intact. The lightning network continues to function as well. All of this is possible without a hard fork thanks to BIP-141. This proposal is for a single sender only, post quantum multi-sig is not possible with P2WOTS, but the existing Schnorr multi-sig will continue to function as is. I look forward to the feedback on this idea from the bitcoin community! Thanks for taking the time to consider this option!

Best, Opus Lux!

-------------------------

murch | 2026-05-26 19:29:57 UTC | #2

Not a blocker, but the P2MR proposal (BIP360) is also using witness version 2.

![image|336x500](upload://qD39fNW4hVC9iDn8VWtg878gFmt.png)

Since your proposal commits to the nonce in the output script, this would still only allow signing a single time for each UTXO, but Bitcoin requires the ability to sign more than once for a UTXO without leaking the private key in case addresses are reused, users want to participate in multi-user transactions, or simply need to replace a prior spending attempt.

-------------------------

opus-lux | 2026-05-26 21:14:43 UTC | #3

Thanks for the awesome reply Murch, you are correct.

I should switch the proposal to target witness version 3 to avoid conflict.

Also, excellent point that this proposal would still only allow signing a single time for each UTXO and you are correct about the mathematical limitation of WOTS. As far as I am aware of there are three main reasons that Bitcoin requires the ability to sign more than once for a UTXO. These three main reasons are address reuse, multi-user transactions and fee bumping / spending attempt retries. 

I tried to address the address reuse concern with the utxo_index in this revised proposal. The utxo_index counter guarantees that each UTXO gets a unique wots_pk. The BIP already calls this a hard security requirement and attempts to address this issue. 

As far as the mult-user transactions go the proposal explicitly states that the proposal is for a single signer only.

Now for the fee bumping or spending attempt retries this is something that is very valid and needs to be addressed in my proposal in more detail. If users tried to bump their transaction's fee using RBF it would be dangerous because they would sign two different messages with the same key. I should make it explicitly clear that CPFP (child pays for parent) via a change output is the only safe way to adjust your fee after transaction broadcast. This is because CPFP allows users to bump their transaction fee without a second WOTS signature. I should update the proposal to explicitly state this limitation and include wallet guidance prohibiting RBF on P2WOTS inputs and recommending CPFP via a P2TR change output instead.

Does this address your concerns with the one time nature of WOTS signatures? Is there something else that I missed or don't understand?

Thanks again for your insightful reply!

-------------------------

murch | 2026-05-27 16:44:01 UTC | #4

[quote="opus-lux, post:3, topic:2530"]
The utxo_index counter guarantees that each UTXO gets a unique wots_pk.

[/quote]

No, you generate the *output script* via the UTXO index. Address reuse occurs if someone sends to the same output script again. If an output script is paid twice, the recipient has the choice to leak his private key or forgo the second payment. This proposal does not mitigate the address reuse concern.

I've raised the same issue multiple times now. That and the style of the answers make feel like I'm remote-prompting your LLM. I'm out, good luck.

-------------------------

